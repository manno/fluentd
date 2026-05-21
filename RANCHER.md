# Rancher Fluentd Fork

Fork of `kube-logging/fluentd-images` with automated security workflows for
the version consumed by the `rancher-logging` 4.10 chart.

## Scope

**Only `v1.16-4.10/` is in scope.** The chart references this version as
`v1.16-4.10-full` (see `packages/rancher-logging/4.10/generated-changes/patch/values.yaml.patch`
in `ob-team-charts`). The `v1.17-5.0/` directory is left untouched by our
automation — it's maintained by upstream but unused by Rancher 4.10.

## Status: SUSE migration deferred

**This fork currently stays on the upstream Alpine + Sumologic base image.**
The migration to `bci-ruby:3.3` (SUSE Base Container Image) is deferred:

- Current base: `public.ecr.aws/sumologic/kubernetes-fluentd:1.16.5-sumo-0-alpine` + `ruby:3.2.5-alpine3.20`
- The Sumo image brings pre-built gem bundles, making builds fast (~5-10 min)
- Switching to bci-ruby:3.3 means full `bundle install` from scratch — 40+ gems with native extensions (rdkafka, oj, snappy, geoip-c). Build time grows to ~30-45 min and there's trial-and-error compiling against zypper packages.

**Supply-chain trade-off**: we depend on Sumo's public ECR image. They rebuild
it on their own cadence. If we need a security fix Sumo hasn't shipped, we
have to bump it ourselves (manual — see `cve-response.md` strategy `sumo-bump`)
or migrate to bci-ruby.

This decision is documented in `docs/logging/fork/STATE.md` in `ob-team-charts`
and is the largest remaining risk in our supply-chain hardening story.

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
- **No BCI**: deferred (see above) — we're on Alpine.
- **No auto-update-ruby**: Ruby patch bumps in the base FROM line carry ABI risk for the pre-built gem bundle from Sumo. Manual bump only.
- **No auto-update-sumo**: Sumo's release cadence is opaque; auto-bumping their tag changes the bundled gems too — high risk. Manual bump only (handled by `cve-response.md` `sumo-bump` strategy when needed).
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
