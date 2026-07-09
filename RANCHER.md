# Rancher Fluentd Fork

Fork of `kube-logging/fluentd-images` with automated security workflows for
the version consumed by the `rancher-logging` 4.10 chart.

## Scope

**Only `v1.16-4.10/` is in scope.** The chart references this version as
`v1.16-4.10-full` (see `packages/rancher-logging/4.10/generated-changes/patch/values.yaml.patch`
in `ob-team-charts`). The `v1.17-5.0/` directory is left untouched by our
automation — it's maintained by upstream but unused by Rancher 4.10.

## Status: SUSE migration complete — running in cluster

All three SUSE images build and push from `rancher-main` via `artifacts-suse.yaml`.
The `full` image has been deployed to a k3d cluster and verified working with the
rendered `4.10.0-rancher.30-suse1` chart. Dispatch pipeline to `ob-team-charts` is
wired and operational.

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
| SUSE BCI (active) | `v1.16-4.10/Dockerfile.suse` | `.github/workflows/artifacts-suse.yaml` | `v1.16-4.10-{base,filters,full}-suse` |

Once a full release cycle completes, the Alpine track will be removed and
`artifacts-suse.yaml` becomes the single pipeline.

### SUSE-specific fixes in Dockerfile.suse

Two issues uncovered during the BCI migration that don't affect Alpine builds:

**1. Binstub naming** — SUSE ruby installs gem executables as `*.ruby3.4`
(e.g., `fluentd.ruby3.4`), not plain `fluentd`. The upstream entrypoint
(`exec fluentd ...`) fails because `fluentd` isn't in PATH. Fixed by adding a
symlink step after `bundle install` in the `full` stage:

```dockerfile
RUN find /usr/local/bundle/bin -name '*.ruby3.4' | \
    while read f; do ln -sf "$f" "${f%.ruby3.4}"; done
WORKDIR /fluentd
```

The `WORKDIR /fluentd` is also required — without it, bash's `exec fluentd` resolves
relative to `/`, finds the `/fluentd` directory, and errors with "Is a directory".

**2. Absolute Gemfile path in filters stage** — After adding `WORKDIR /fluentd`,
the `filters` stage's `fluent-gem install --file Gemfile.filters` broke because
`Gemfile.filters` is at `/Gemfile.filters` (not `/fluentd/Gemfile.filters`).
Fixed by using the absolute path: `--file /Gemfile.filters`.

**3. fluentd version bump** — The transitive gem dependencies in `outputs/Gemfile`
pull in `fluentd 1.19.0` (up from `1.16.x`). This is the version that ends up
running. Acceptable for the POC; acceptable for production given code-freeze strategy
(only the gem version changes, not our application code).

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
| Push to rancher-main | `.github/workflows/artifacts.yaml` (upstream, modified) | Build + push Alpine base/filters/full images for v1.16-4.10 |
| Push to rancher-main | `.github/workflows/artifacts-suse.yaml` | Build + push SUSE base/filters/full images; dispatches `image-updated` to `ob-team-charts` after `full` build |
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

## Dispatch pipeline

After each successful `full` image build on `rancher-main`, `artifacts-suse.yaml` posts
a `repository_dispatch` event to `manno/ob-team-charts`:

```bash
curl -X POST \
  -H "Authorization: Bearer ${CHARTS_DISPATCH_TOKEN}" \
  https://api.github.com/repos/manno/ob-team-charts/dispatches \
  -d '{"event_type":"image-updated","client_payload":{"component":"fluentd","tag":"v1.16-4.10-full-suse"}}'
```

`ob-team-charts/.github/workflows/image-update.yaml` receives this, resets
`auto/rancher-logging-suse-updates` from `origin/rancher-logging-4.10-suse1`, updates
`values.yaml.patch` and `package.yaml`, and opens/updates a PR.

`CHARTS_DISPATCH_TOKEN` is a fine-grained PAT with **Contents: write** on
`manno/ob-team-charts`, stored as a repo secret.

## References

- POC state: `docs/logging/fork/STATE.md` in `ob-team-charts`
- Smoke test: `ob-team-charts/dev-scripts/smoke-test-rancher-logging.sh`
- Sibling forks: `manno/logging-operator`, `manno/config-reloader`, `manno/fluent-bit`
