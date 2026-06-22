# Rancher Fluentd Fork

Fork of `kube-logging/fluentd-images` with automated security workflows for
the version consumed by the `rancher-logging` 4.10 chart.

## Scope

**Only `v1.16-4.10/` is in scope.** The chart references this version as
`v1.16-4.10-full` (see `packages/rancher-logging/4.10/generated-changes/patch/values.yaml.patch`
in `ob-team-charts`). The `v1.17-5.0/` directory is left untouched by our
automation — it's maintained by upstream but unused by Rancher 4.10.

## Status: SUSE migration — pipeline green, awaiting smoke test

All three SUSE images now build and push successfully on branch
`bci-ruby-migration` (PR [#6](https://github.com/manno/fluentd/pull/6)):

| Stage   | Build time | Tag                                         |
|---------|------------|---------------------------------------------|
| base    | ~13–18 min | `ghcr.io/manno/fluentd:v1.16-4.10-base-suse`    |
| filters | ~25–27 min | `ghcr.io/manno/fluentd:v1.16-4.10-filters-suse` |
| full    | ~95 min    | `ghcr.io/manno/fluentd:v1.16-4.10-full-suse`    |

The Alpine + Sumologic pipeline stays live during the transition; the
SUSE pipeline runs in parallel:

| Track | Dockerfile | Workflow | Tags pushed |
|---|---|---|---|
| Alpine + Sumo (current prod) | `v1.16-4.10/Dockerfile` | `.github/workflows/artifacts.yaml` | `v1.16-4.10-{base,filters,full}` |
| SUSE BCI (new) | `v1.16-4.10/Dockerfile.suse` | `.github/workflows/artifacts-suse.yaml` | `v1.16-4.10-{base,filters,full}-suse` |

Once the SUSE `full` image passes the smoke test against a real chart
deploy (`ob-team-charts/dev-scripts/smoke-test-rancher-logging.sh`
with `IMAGE_FLUENTD=ghcr.io/manno/fluentd:v1.16-4.10-full-suse`) and
a full release cycle, the Alpine track will be removed and
`artifacts-suse.yaml` becomes the single pipeline.

### Vendored libraries (not in SLE_BCI)

The migration uncovered three libraries that bci-base doesn't ship and
that the gem set hard-depends on:

- **legacy libGeoIP** — required by `geoip-c` (transitive of
  `fluent-plugin-geoip`). MaxMind retired the library in 2018; not in
  any SUSE repo. Built from source (`v1.6.12`) into `/usr/local` in
  the base stage, registered with `ldconfig`. ~15 s, <2 MB.
- **libmaxminddb** — needed by `geoip2_c`. The gem vendors its own
  source and bootstraps with autotools, so we just install
  `automake`/`autoconf`/`libtool`/`gawk` and let extconf do the work.
- **tini** — not packaged. Skipping; bash as PID 1 is fine for fluentd
  which manages its own children via supervisor.

`librdkafka` turned out **not** to be a blocker — the `rdkafka` gem in
`outputs/Gemfile` (loaded by the `full` stage) compiled cleanly from
its bundled C sources via cargo-style vendoring. No vendor step needed.

### Why this took until now

- Current Alpine base brought pre-built gem bundles → ~5–10 min builds.
- bci-base means `bundle install` from scratch for ~80 gems with
  native extensions. End-to-end build is ~90 min on QEMU multi-arch.
- Five missing build-toolchain pieces had to be discovered one CI
  iteration at a time: `automake`/`autoconf`/`libtool` (autotools),
  `gawk` (configure scripts), `xz` (nokogiri's libxml2 tarball), plus
  vendoring libGeoIP from source.

Strategy `sumo-bump` in `cve-response.md` remains available as a
fallback if a CVE lands before the Alpine track is retired.

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
