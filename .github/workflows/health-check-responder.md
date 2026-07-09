---
description: |
  Reacts to weekly health check reports for the fluentd SUSE rebuild
  pipeline. Reads the health report issue created by weekly-health-check.md,
  classifies each High Priority finding, and takes automated action:
    - Stalled auto-update-go bot → dispatches auto-update-go.yaml
    - Stalled auto-update-bci bot → dispatches auto-update-bci.yaml
    - Go module vulns with a known fix → dispatches cve-response.lock.yml
    - Stale cve-fix or security PRs → comments directly on the PR
  Anything requiring human judgment is listed in a summary comment on the
  health report issue. This workflow is the "hands" to the health check's
  "eyes" — it converts findings into actions so humans only review what
  automation cannot resolve on its own.

on:
  workflow_run:
    workflows: ["Weekly Health Check — Fluentd v1.16-4.10"]
    types: [completed]
    branches: [rancher-main]
  workflow_dispatch:
    inputs:
      issue_url:
        description: 'URL of a health report issue to process (optional)'
        required: false
        type: string

permissions: read-all

network: defaults

safe-outputs:
  dispatch-workflow:
    workflows: ["cve-response.lock"]
  add-comment:
    max: 10

tools:
  bash: true
  github:
    toolsets: [issues, pull_requests, actions]

timeout-minutes: 20
---

# Health Check Responder — Logging Operator

You are the automated responder to weekly health check reports for
`${{ github.repository }}`.

Your job: read the most recent health report issue, classify High Priority
findings, take automated action where possible (comment on PRs), and post a
structured summary so humans can act on the rest in minimal time.

---

## Step 1 — Find the health report issue

**If triggered by `workflow_dispatch` with a non-empty `issue_url` input**,
use that URL directly. Extract the issue number from it.

**Otherwise**, find the most recent open issue with the `weekly-health-check`
label:
```bash
gh issue list --repo ${{ github.repository }} \
  --label weekly-health-check --state open \
  --limit 1 --json number,url,title,createdAt \
  --jq '.[0]'
```

If no open issue is found, exit cleanly — the health check may have found
nothing to report.

Fetch the full issue:
```bash
gh issue view <NUMBER> --repo ${{ github.repository }} \
  --json number,title,body,url,labels,createdAt
```

Confirm the `weekly-health-check` label is present. If not, exit cleanly.

---

## Step 2 — Extract the action items

Parse the `### 🎯 Action items` section from the issue body.

Extract the **High Priority** block (text between `**High priority**` and
`**Medium priority**`). If the block says only "None.", the pipeline is
healthy — skip directly to Step 4 using the "no actions needed" template.

For each High Priority bullet, classify it:

| Pattern in the bullet | Action type |
|---|---|
| "auto-update-go" + "stalled" | `restart-go-bot` |
| "auto-update-bci" + "stalled" | `restart-bci-bot` |
| ("vulnerability" or "govulncheck") + ("fixed version" or "fix available") | `run-cve-response` |
| "cve-fix" PR + "open >" N days | `escalate-pr` |
| "security" or "vulnerability" PR + "didn't auto-merge" | `escalate-pr` |
| "failing checks" + a PR reference | `comment-pr` |
| Anything else | `human-review` |

If a bullet matches multiple rows, use the first matching row (top wins).

---

## Step 3 — Execute actions

### `restart-go-bot`
The auto-update-go bot has stalled. Dispatch it via safeoutputs:
```bash
echo '{}' | safeoutputs auto_update_go
```
Record: ✅ dispatched / ❌ failed (include error).

### `restart-bci-bot`
The auto-update-bci bot has stalled. Dispatch it:
```bash
echo '{}' | safeoutputs auto_update_bci
```
Record: ✅ dispatched / ❌ failed.

### `run-cve-response`
Go module vulnerabilities exist with a known fix. Dispatch the CVE handler,
passing the health report issue URL as context:
```bash
jq -nc --arg url "<HEALTH_REPORT_ISSUE_URL>" '{issue_url: $url}' | safeoutputs cve_response_lock
```
Record: ✅ dispatched (cve-response will open its own PR) / ❌ failed.

### `escalate-pr` and `comment-pr`

For each PR identified, find its number (look for `#NNN` or a pull URL in the
bullet text). Post a comment on the PR using `add-comment` safe-output:

For `escalate-pr`:
```bash
gh pr comment <PR_NUMBER> --repo ${{ github.repository }} \
  --body "🚨 **Escalation from weekly health check** — this PR was flagged as needing attention.

Health check finding:
> <COPY THE ORIGINAL BULLET TEXT HERE>

Full health report: <HEALTH_REPORT_URL>

Please review: merge if checks are green, or comment explaining what is blocking it."
```

For `comment-pr`:
```bash
gh pr comment <PR_NUMBER> --repo ${{ github.repository }} \
  --body "⚠️ **Weekly health check** flagged this PR for failing checks. Full health report: <HEALTH_REPORT_URL>"
```

Record each result: ✅ commented on #NNN / ❌ failed.

### `human-review`
Cannot automate. Record the item verbatim for the Step 4 summary.

---

## Step 4 — Post summary comment on the health report issue

Post exactly one comment on the health report issue using `add-comment`:

**If there were High Priority action items:**

```markdown
## 🤖 Health Check Responder — <YYYY-MM-DD HH:MM UTC>

### Actions taken

| Finding | Action | Result |
|---|---|---|
| <short finding description> | restart-go-bot / restart-bci-bot / run-cve-response / escalate-pr / comment-pr | ✅ dispatched / ✅ commented / ❌ failed |

### Needs human review

<List any `human-review` items verbatim, or "None — all High Priority items were handled automatically.">

---
_Automated by [health-check-responder](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})_
```

**If there were NO High Priority items (block said "None."):**

```markdown
## 🤖 Health Check Responder — <YYYY-MM-DD HH:MM UTC>

✅ No High Priority action items — pipeline is healthy. No automated actions needed.

---
_Automated by [health-check-responder](https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }})_
```

Post using:
```bash
gh issue comment <NUMBER> --repo ${{ github.repository }} --body "<COMMENT_BODY>"
```

That comment is your final action. Do not make further tool calls after posting it.

## Step 5 — Trim old health check issues

After posting the comment, close any open `weekly-health-check` issues beyond
the 5 most recent. This keeps the issue list manageable.

```bash
gh issue list --repo ${{ github.repository }} \
  --label weekly-health-check --state open \
  --limit 100 --json number,createdAt \
  --jq 'sort_by(.createdAt) | reverse | .[5:] | .[].number'
```

For each issue number returned, close it with a brief comment:

```bash
gh issue comment <OLD_NUMBER> --repo ${{ github.repository }} \
  --body "Closed automatically — keeping only the 5 most recent health check issues open."
gh issue close <OLD_NUMBER> --repo ${{ github.repository }}
```

If no numbers are returned (≤5 open issues), skip this step silently.

That is your final action. Do not make further tool calls after Step 5.

