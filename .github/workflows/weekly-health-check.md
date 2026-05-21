---
description: |
  Weekly health check for the fluentd v1.16-4.10 fork.
  Monitors build status, Renovate gem-PR flow, open security PRs, gem
  vulnerabilities (bundler-audit), and Ruby + Sumo base image freshness.

  Meta-monitor — flags when the automation stalls. Note: fluentd has NO
  custom auto-update bots (Renovate owns gem updates; Ruby and Sumo bumps
  are manual). The bot-liveness check is therefore Renovate-focused.

on:
  schedule: weekly on monday
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: read
  actions: read
  issues: read

network: defaults

safe-outputs:
  create-issue:
    max: 1
    title-prefix: "[weekly-health-check] "
    labels: [weekly-health-check, automated]

tools:
  bash: ["git:*", "gh:*", "sed", "grep", "cat", "echo", "date", "curl", "jq", "docker:*", "awk", "head", "tail"]
  github:
    toolsets: [actions, pull_requests, issues]

timeout-minutes: 25
---

# Weekly Health Check — Fluentd v1.16-4.10

Generate the weekly health report for `${{ github.repository }}`, scoped to
`v1.16-4.10/`. The `v1.17-5.0/` directory is upstream-only and out of scope
for the rancher-logging 4.10 chart.

This fork stays on the upstream Alpine + Sumologic base. The SUSE (`bci-ruby:3.3`)
migration is deferred. Renovate owns gem updates; the agentic `cve-response.md`
handles long-tail (Ruby bump, Sumo bump, replace-with-fork, frozen-specific-install).

---

## Step 1 — Build status on rancher-main

```bash
gh run list --repo ${{ github.repository }} --branch rancher-main --limit 30 \
  --json conclusion,status,name,workflowName,createdAt,event,headSha,url
```

Workflows to expect: `Artifacts`, `CI`, `Weekly Health Check`. The
`Artifacts` workflow builds 3 image-types (base/filters/full) for
v1.16-4.10 — expect 3 matrix rows per run.

If a workflow you expect is missing entirely, treat that as a finding.

## Step 2 — Renovate flow health

Fluentd has no custom auto-update bots — Renovate is the rebuild pipeline.
Check that gem-update PRs are flowing.

```bash
# All Renovate-authored open PRs targeting v1.16-4.10
gh pr list --repo ${{ github.repository }} --state open --author "app/renovate" \
  --json number,title,createdAt,labels,url,headRefName
gh pr list --repo ${{ github.repository }} --state open --label dependencies \
  --json number,title,createdAt,labels,url
```

Also check the Renovate "Dependency Dashboard" issue (Renovate creates one
when `:dependencyDashboard` is in the extends list — our renovate.json5
includes it):

```bash
gh issue list --repo ${{ github.repository }} --state open \
  --search "Dependency Dashboard in:title" --json number,title,createdAt,url
```

**Escalation rules:**
- No Renovate PR in the last 7 days AND no Dependency Dashboard issue → Renovate may not be installed/active on the fork
- Dependency Dashboard issue exists but lists outstanding security advisories → vuln auto-merge may be failing

## Step 3 — Open PR analysis

```bash
gh pr list --repo ${{ github.repository }} --state open --limit 50 \
  --json number,title,labels,createdAt,updatedAt,author,isDraft,url,headRefName,statusCheckRollup
```

Group by labels:

| Group | Label(s) | Source |
|---|---|---|
| CVE fixes | `cve-fix` | `cve-response.md` |
| CVE tracking (frozen) | `cve-tracking` | `cve-response.md` (frozen-* strategies) |
| Security | `security`, `vulnerability` | Renovate vuln alerts |
| Gem updates | `ruby`, `dependencies` | Renovate (bundler) |
| GitHub Actions | `ci`, `github-actions` | Renovate |
| Untagged | (none of above) | Manual |

For each group: count, oldest age in days, count with failing checks.

**Escalation rules:**
- Any `cve-fix` PR open > 3 days → review escalation
- Any `cve-tracking` issue open > 14 days with no upstream movement → flag for security team
- Any `security`/`vulnerability` PR present (should auto-merge) → investigate

## Step 4 — Gem vulnerability scan (bundler-audit)

Run bundler-audit against both Gemfile.lock files. This is the analog of
`govulncheck` for Ruby — checks `Gemfile.lock` against the Ruby Advisory
Database.

```bash
for dir in v1.16-4.10/filters v1.16-4.10/outputs; do
  echo "=== ${dir} ==="
  docker run --rm \
    -v "$(pwd)/${dir}:/work" -w /work \
    ruby:3.2.5-alpine3.20 \
    sh -c "gem install bundler-audit --no-document --silent \
           && bundler-audit check --update --ignore-known" 2>&1 | tail -50
done
```

Report per Gemfile: number of advisories found, gem names, fixed versions.

**Escalation rules:**
- Any advisory with `Solution: upgrade` available → either Renovate should
  have caught it (investigate auto-merge failure) or a major bump is needed
  (cve-response should handle)

## Step 5 — Base image freshness

**Ruby base** (`ruby:3.2.5-alpine3.20`):

```bash
CURRENT_RUBY=$(grep -E "^FROM ruby:" v1.16-4.10/Dockerfile | head -1 | sed -nE 's|FROM ruby:([^ ]+).*|\1|p')
# Latest stable Ruby 3.2.x from rubygems.org
LATEST_RUBY=$(curl -fsSL https://www.ruby-lang.org/en/downloads/releases/ 2>/dev/null \
  | grep -oE 'Ruby 3\.2\.[0-9]+' | head -1 | awk '{print $2}')
echo "ruby current=${CURRENT_RUBY} latest_3.2.x=${LATEST_RUBY}"
```

Status: in sync if CURRENT matches LATEST major.minor.patch, drift otherwise.
**Ruby bumps are NOT auto-merged** — this is informational.

**Sumologic base image:**

```bash
CURRENT_SUMO=$(grep -E "^FROM public.ecr.aws/sumologic/kubernetes-fluentd" v1.16-4.10/Dockerfile | head -1 | awk '{print $2}')
# Try to fetch the Sumo image's last-modified time via Docker registry
SUMO_AGE=$(docker buildx imagetools inspect "${CURRENT_SUMO}" --format '{{json .}}' 2>/dev/null \
  | jq -r '.image.created // "unknown"')
DOCKERFILE_AGE=$(git log -1 --format=%cr -- v1.16-4.10/Dockerfile)
echo "sumo image=${CURRENT_SUMO} created=${SUMO_AGE} dockerfile_last_changed=${DOCKERFILE_AGE}"
```

Status: informational. Sumo bumps are NOT auto-merged. If the Sumo image is
older than 90 days, that's worth flagging — they likely have a newer tag.

## Step 6 — Emit the report

**Emit the full markdown report as your final response.** The `safe-outputs`
handler will turn it into a GitHub issue.

```markdown
# Health Report — Week of <YYYY-MM-DD>

## Weekly Health Report — Fluentd v1.16-4.10

**Repository**: `${{ github.repository }}`
**Report date**: <YYYY-MM-DD>
**Branch**: `rancher-main`
**Scope**: `v1.16-4.10/` only (v1.17-5.0 out of scope)
**Base**: Alpine + Sumologic (SUSE migration deferred)
**Pipeline status**: <one-line: 🟢 healthy / 🟡 drift / 🔴 broken>

---

### Build health (most recent run per workflow on `rancher-main`)

| Workflow | Conclusion | Age | Run |
|---|---|---|---|
| Artifacts (v1.16-4.10 base/filters/full) | ✅/❌ | Xh | <url> |
| CI | ✅/❌ | Xh | <url> |
| Weekly Health Check | ✅/❌ | Xh | <url> |

<If any failed, list one bullet per failure with run URL.>

---

### Renovate flow

| Indicator | Value |
|---|---|
| Open Renovate PRs | X |
| Most recent Renovate PR opened | <Nh ago> |
| Dependency Dashboard issue | #N or none |

<If no Renovate activity in 7+ days OR no Dependency Dashboard:>
> 🚨 **Renovate may not be active** — no PRs in <N days>; no dashboard issue
> found. Verify the Renovate GitHub App is installed on the fork.

---

### Open PRs / Issues

| Group | Count | Oldest | Failing checks |
|---|---|---|---|
| cve-fix | X | Nd | Y |
| cve-tracking (frozen) | X (issues) | Nd | n/a |
| security / vulnerability | X | Nd | Y |
| ruby / dependencies (gems) | X | Nd | Y |
| ci / github-actions | X | Nd | Y |
| untagged | X | Nd | Y |

---

### Gem vulnerabilities (bundler-audit)

| Gemfile | Advisories | Top gems affected |
|---|---|---|
| v1.16-4.10/filters/Gemfile.lock | N | <gem-list> |
| v1.16-4.10/outputs/Gemfile.lock | N | <gem-list> |

<If advisories exist with available fixes, list them as:>
- `<gem>` `<vulnerable-version>` → `<fixed-version>` — CVE: `<CVE-ID>`

---

### Base image freshness

| Component | Current | Latest known | Status |
|---|---|---|---|
| Ruby (in Dockerfile) | `<current>` | `<latest 3.2.x>` | ✅ in sync / ⚠️ drift (manual bump) |
| Sumo image | `<current tag>` | (last modified <age>) | ✅ / ⚠️ stale (manual bump) |

`v1.16-4.10/Dockerfile` last changed: <relative-time>

---

### 🎯 Action items

Generate strictly from the data above. Empty sections → write "None."

**High priority** (rebuild pipeline broken or unaddressed CVE with fix available):
- ...

**Medium priority** (drift accumulating or stale tracking issues):
- ...

**Low priority** (nuisance / housekeeping):
- ...

---

📅 **Next report**: <next Monday date>
🤖 Generated by [Weekly Health Check](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})

<!-- gh-aw-workflow-id: weekly-health-check -->
```

That markdown block IS your final response — emit it verbatim with placeholders
filled in. Do not run further tool calls after emitting it.
