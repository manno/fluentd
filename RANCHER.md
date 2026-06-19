# Rancher Fluentd Fork

Fork of `kube-logging/fluentd-images` with automated security workflows for
the version consumed by the `rancher-logging` 4.10 chart.

## Scope

**Only `v1.16-4.10/` is in scope.** The chart references this version as
`v1.16-4.10-full` (see `packages/rancher-logging/4.10/generated-changes/patch/values.yaml.patch`
in `ob-team-charts`). The `v1.17-5.0/` directory is left untouched by our
automation — it's maintained by upstream but unused by Rancher 4.10.

## Status: SUSE migration in progress

The migration to `registry.suse.com/bci/ruby:3.4` is now underway on
branch `bci-ruby-migration`. The Alpine + Sumologic pipeline stays live
during the transition; the SUSE pipeline runs in parallel:

| Track | Dockerfile | Workflow | Tags pushed |
|---|---|---|---|
| Alpine + Sumo (current prod) | `v1.16-4.10/Dockerfile` | `.github/workflows/artifacts.yaml` | `v1.16-4.10-{base,filters,full}` |
| SUSE BCI (in migration) | `v1.16-4.10/Dockerfile.suse` | `.github/workflows/artifacts-suse.yaml` | `v1.16-4.10-{base,filters,full}-suse` |

Once the SUSE images pass the smoke test against a real chart deploy and a
full release cycle, the Alpine track will be removed and `artifacts-suse.yaml`
becomes the single pipeline.

### Why this took until now

- Current Alpine base brought pre-built gem bundles → ~5–10 min builds.
- bci/ruby:3.4 means `bundle install` from scratch for 40+ gems with native
  extensions (rdkafka, oj, snappy, libmaxminddb, …). Expect ~30–45 min
  build cycles and iteration to get every native gem compiling against
  the SUSE library set.
- Largest unknown: `librdkafka-devel` availability in the BCI repos. If it
  isn't shipped, the `full` stage's `rdkafka` gem will fail and we'll need
  to vendor librdkafka from source.

Strategy `sumo-bump` in `cve-response.md` remains available as a fallback
while the migration stabilizes.

## Automation layer

| Layer | Mechanism | What it owns |
|---|---|---|
| Continuous | `renovate.json5` | Gem updates (bundler manager) for v1.16-4.10 — vuln auto-merge, patches auto-merge |
| Triggered | `.github/workflows/cve-response.md` (agentic) | Long-tail CVE fixes (Ruby bump, sumo bump, gem replace-with-fork) |
| Weekly | `.github/workflows/weekly-health-check.md` (agentic) | Meta-monitor — bundler-audit + Renovate flow + Ruby/Sumo freshness |
| Push to rancher-main | `.github/workflows/artifacts.yaml` (upstream, modified) | Build + push base/filters/full images for v1.16-4.10 |
| PR | `.github/workflows/ci.yaml` (upstream, unchanged) | Build-only on PR — no push |

## Why no auto-update-go / auto-update-bci / auto-update-ruby / goreleaser

- **No Go**: Ruby project.
- **No auto-update-bci** *(yet)*: will be added once `Dockerfile.suse` is the
  canonical Dockerfile and the Alpine track is retired. Same shape as the
  other forks' `auto-update-bci.yaml`.
- **No auto-update-ruby**: Ruby patch bumps in the base FROM line carry ABI
  risk against compiled native gems. Manual bump only.
- **No auto-update-sumo**: Sumo's release cadence is opaque; auto-bumping
  their tag changes the bundled gems too — high risk. Manual bump only via
  `cve-response.md` `sumo-bump` strategy. Goes away once the SUSE
  migration lands.
- **No goreleaser**: build runs entirely in Docker via Bundler.

## Renovate override

Upstream `kube-logging/fluentd-images` intentionally disables Renovate for
`v1.16-4.10/*` and `v1.17-5.0/*` because they handle Ruby gem updates manually
upstream. Our `renovate.json5` overrides this for `v1.16-4.10/*` only — we
want vuln auto-merge for the version we ship.

## Coexistence with upstream workflows

We kept upstream's `.github/workflows/artifacts.yaml` and `ci.yaml`. The only
modification: `artifacts.yaml` now fires on push to `rancher-main` (in
addition to `main`), so OUR fork's builds push our tags.

## Local build

```bash
# Build the v1.16-4.10 full image
docker buildx build \
  -f v1.16-4.10/Dockerfile \
  --target full \
  -t fluentd:dev-v1.16-4.10-full \
  v1.16-4.10/
```

## Images

Built by `artifacts.yaml` on push to `rancher-main`, pushed to GHCR:

- `ghcr.io/manno/fluentd:v1.16-4.10-base`
- `ghcr.io/manno/fluentd:v1.16-4.10-filters`
- `ghcr.io/manno/fluentd:v1.16-4.10-full` ← what the rancher-logging chart consumes

## Upstream

- **Upstream**: https://github.com/kube-logging/fluentd-images
- **Sync strategy**: None — code/Dockerfiles stay frozen at fork point
- **Security strategy**: Renovate-driven gem updates for v1.16-4.10/*; agentic handler for the long tail

## References

- POC state: `docs/logging/fork/STATE.md` in `ob-team-charts`
- Sibling forks: `manno/logging-operator`, `manno/config-reloader`, `manno/fluent-bit`
