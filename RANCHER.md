# Rancher Fluentd Fork

Fork of `kube-logging/fluentd-images` with automated security workflows for
the version consumed by the `rancher-logging` 4.10 chart.

## Scope

**Only `v1.16-4.10/` is in scope** — it's the only version this fork carries. The
chart references it as `v1.16-4.10-full-suse` (see
`packages/rancher-logging/4.10/generated-changes/patch/values.yaml.patch` in
`ob-team-charts`). Upstream's `v1.17-5.0/` was dropped — it's unused by Rancher 4.10.

## Status: SUSE is the only track — Alpine retired

All three images build and push from `rancher-main` via `artifacts.yaml`, the
single build pipeline. The `full` image has been deployed to a k3d cluster and
verified working with the rendered `4.10.0-rancher.30-suse1` chart. Dispatch
pipeline to `ob-team-charts` is wired and operational.

| Stage   | Build time | Tag                                         |
|---------|------------|---------------------------------------------|
| base    | ~13–18 min | `ghcr.io/manno/fluentd:v1.16-4.10-base-suse`    |
| filters | ~25–27 min | `ghcr.io/manno/fluentd:v1.16-4.10-filters-suse` |
| full    | ~95 min    | `ghcr.io/manno/fluentd:v1.16-4.10-full-suse`    |

The legacy Alpine + Sumologic track (its `Dockerfile` and separate build workflow)
has been removed. The canonical `v1.16-4.10/Dockerfile` is now the SUSE (bci-base)
build. Published tags keep the `-suse` suffix as a base-image lineage marker — it is
what the rancher-logging chart consumes, so retiring the Alpine track required no
chart change.

### SUSE-specific fixes in the Dockerfile

Two issues uncovered during the BCI migration:

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

- The old Alpine base brought pre-built gem bundles → ~5–10 min builds.
- bci-base means `bundle install` from scratch for ~80 gems with
  native extensions. End-to-end build is ~90 min on QEMU multi-arch.
- Five missing build-toolchain pieces had to be discovered one CI
  iteration at a time: `automake`/`autoconf`/`libtool` (autotools),
  `gawk` (configure scripts), `xz` (nokogiri's libxml2 tarball), plus
  vendoring libGeoIP from source.

## Automation layer

| Layer | Mechanism | What it owns |
|---|---|---|
| Continuous | `renovate.json5` | Gem updates (bundler manager) for v1.16-4.10 — vuln auto-merge, patches auto-merge |
| Triggered | `.github/workflows/cve-response.md` (agentic) | Long-tail CVE fixes (Ruby bump, gem replace-with-fork, frozen-specific-install) |
| Weekly | `.github/workflows/weekly-health-check.md` (agentic) | Meta-monitor — bundler-audit + Renovate flow + Ruby/bci-base freshness |
| Daily | `.github/workflows/auto-update-bci.yaml` | Repin the `bci-base` digest in `v1.16-4.10/Dockerfile`; one PR per drift |
| Push to rancher-main | `.github/workflows/artifacts.yaml` | Build + push SUSE base/filters/full images for v1.16-4.10; runs Trivy; dispatches `image-updated` to `ob-team-charts` after `full` build |
| PR | `.github/workflows/ci.yaml` (upstream, trimmed to v1.16-4.10) | Build-only on PR — no push |

## Why no auto-update-go / auto-update-ruby / goreleaser

- **No Go**: Ruby project.
- **No auto-update-ruby**: Ruby is installed from SLE_BCI via the
  `RUBY_PKG_VERSION` ARG; bumps carry ABI risk against compiled native gems and are
  constrained to versions SLE_BCI ships. Manual bump only.
- **No goreleaser**: build runs entirely in Docker via Bundler.

## Renovate override

Upstream `kube-logging/fluentd-images` intentionally disables Renovate for
`v1.16-4.10/*` because they handle Ruby gem updates manually upstream. Our
`renovate.json5` overrides this for `v1.16-4.10/*` — we want vuln auto-merge for
the version we ship.

## Coexistence with upstream workflows

We replaced upstream's `.github/workflows/artifacts.yaml` with our SUSE build
pipeline: it fires on push to `rancher-main`, builds `v1.16-4.10/Dockerfile`,
pushes our `-suse` tags, runs Trivy, and dispatches to `ob-team-charts`. Upstream's
`ci.yaml` is kept for PR build-only, trimmed to the v1.16-4.10 SUSE image.

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

- `ghcr.io/manno/fluentd:v1.16-4.10-base-suse`
- `ghcr.io/manno/fluentd:v1.16-4.10-filters-suse`
- `ghcr.io/manno/fluentd:v1.16-4.10-full-suse` ← what the rancher-logging chart consumes

## Upstream

- **Upstream**: https://github.com/kube-logging/fluentd-images
- **Sync strategy**: None — code/Dockerfiles stay frozen at fork point
- **Security strategy**: Renovate-driven gem updates for v1.16-4.10/*; agentic handler for the long tail

## Dispatch pipeline

After each successful `full` image build on `rancher-main`, `artifacts.yaml` posts
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
