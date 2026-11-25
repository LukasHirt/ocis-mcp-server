# agents.md — ocis-mcp-server

## Repository Overview

This repository builds the official **oCIS MCP Server** Docker image
(`owncloud/ocis-mcp-server` on Docker Hub) — the Model Context Protocol server for
ownCloud Infinite Scale. It does not contain the ocis-mcp-server source code — it builds
the server **from source** by cloning
[`owncloud/ocis-mcp-server`](https://github.com/owncloud/ocis-mcp-server) at a signed
release tag and ships a minimal Alpine runtime image. Images are multi-architecture and
built via GitHub Actions.

This repository is the **only** publisher of the image, the same split as oCIS: the
application repository cuts signed release tags and the GitHub release (with binaries,
via goreleaser), and this repository publishes the container image.

- **Classification:** Docker image build (from source)
- **Activity Status:** Active
- **License:** Apache-2.0
- **Language:** Dockerfile, Shell

## Architecture & Key Paths

- `Dockerfile.multiarch` — three-stage build (build context is the repo root, but nothing
  is copied from it):
  - **source** (`golang:*-alpine`) — clones upstream at `${GIT_REF}`, pinned to
    `${GIT_SHA}` when set.
  - **build** (`golang:*-alpine`) — compiles `cmd/ocis-mcp-server` with
    `CGO_ENABLED=0` for `${TARGETARCH}`, with `-X main.version=${VERSION}` (`dev` when
    empty), the version the server reports in its MCP `initialize` response.
  - **runtime** (`alpine`) — `apk upgrade`, `ca-certificates`, `tzdata`,
    `/usr/local/bin/ocis-mcp-server` as the entrypoint, uid/gid 10001 (`mcp`),
    `OCIS_MCP_HTTP_ADDR=0.0.0.0:8090`, port `8090`. No volume, no `HEALTHCHECK`.
- `scripts/resolve-pending-releases.sh` — decides what CI builds; prints the build
  matrix as JSON.
- `scripts/verify-tag.sh` — checks that an upstream tag is annotated and its signature
  is verified, and prints the commit it points to.
- `scripts/test-resolve-pending-releases.sh` — offline tests for the resolver.
- `.github/workflows/main.yml` — **active** CI: test, resolve, build, scan, smoke test,
  publish.
- `.github/workflows/lint-pr-title.yml` — Conventional-Commit PR-title enforcement
- `.github/dependabot.yml` — weekly GitHub Actions and Docker base-image dependency updates
- `.github/CODEOWNERS` — review ownership
- `.editorconfig` — formatting rules (2-space indent, LF, trailing newline)
- `.trivyignore` — accepted-CVE exclusions for the Trivy scan
- `LICENSE` — Apache-2.0

There is **no `CHANGELOG.md`** in this repository.

## Build & CI

GitHub Actions cannot trigger on a tag pushed to another repository, so `main.yml`
polls. Its `prepare` job runs `scripts/resolve-pending-releases.sh`, which:

1. Lists the upstream tags (`git ls-remote --tags`) and keeps only plain `vX.Y.Z` ones
  at or above `MIN_VERSION` (`1.1.0`, see below).
2. Lists the tags already on Docker Hub (paginated). A missing Docker Hub repository
  (HTTP 404) counts as "nothing published"; any other failure aborts the run.
3. Picks candidates by `MODE`, verifies each with `scripts/verify-tag.sh`, and prints
  `[{version, ref, sha, latest}, …]`. `latest` is true only for the highest upstream
  version (`sort -V`).

| Trigger | `MODE` | Builds |
|---------|--------|--------|
| `schedule` `0 */6 * * *`, `workflow_dispatch` without `rebuild-newest` | `pending` | every release not yet on Docker Hub |
| `push` to `main`, `schedule` `0 3 * * 0`, `workflow_dispatch` with `rebuild-newest` | `rebuild` | the same, plus the newest release |
| `pull_request` | `pr` | the newest release, or upstream `main` while there is none or it does not verify; never pushed |

`build` calls the reusable `docker-build-native.yml` workflow from
[`owncloud-docker/ubuntu`](https://github.com/owncloud-docker/ubuntu) per matrix leg
(`fail-fast: false`). Each leg is tagged `X.Y.Z` and `X.Y.Z-YYYYMMDD`, plus `latest` for
the highest release. It is scanned with Trivy (`.trivyignore`, `exit-code: 1`) and
smoke-tested, then on non-PR events pushed to Docker Hub, and the README is synced as the
image description.

### Smoke test

The server defaults to the stdio transport and has no health endpoint, so the URL-polling
smoke test the siblings use does not fit. Instead `main.yml` passes `smoke-test-cmd`
(run through `/bin/sh` in the image, which is one reason the runtime is Alpine and not
distroless): it pipes one JSON-RPC `initialize` request into `ocis-mcp-server`, holding
stdin open for a few seconds (the server exits on EOF before replying), and greps the
reply for `"serverInfo":{…"version":"<matrix.version>"`. That exercises the default mode
and asserts the exact version being published (`main` for PR builds of upstream `main`).
Start-up does not contact oCIS, so a placeholder `OCIS_MCP_OCIS_URL` and dummy app token
suffice.

The resolver's inputs can be replaced for testing: `UPSTREAM_TAGS`,
`UPSTREAM_MAIN_SHA`, `PUBLISHED_TAGS`, `DOCKERHUB_TAGS_URL`, `VERIFY_TAG`,
`CURL_RETRY` (see the script header).

```bash
scripts/test-resolve-pending-releases.sh                                    # offline tests
GH_TOKEN=$(gh auth token) MODE=pending scripts/resolve-pending-releases.sh  # what CI would build now
GH_TOKEN=$(gh auth token) MODE=pr scripts/resolve-pending-releases.sh       # what a PR would build
```

To build and smoke-test locally:

```bash
docker buildx build --load -f Dockerfile.multiarch \
  --build-arg GIT_REF=v1.1.0 --build-arg VERSION=1.1.0 -t ocis-mcp-server:test .
( printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"smoke","version":"0"}}}'; sleep 3 ) |
  docker run -i --rm -e OCIS_MCP_OCIS_URL=https://ocis.invalid \
    -e OCIS_MCP_APP_TOKEN_USER=smoke -e OCIS_MCP_APP_TOKEN_VALUE=smoke ocis-mcp-server:test
```

## Deviations from the Sibling Repositories

- **Minimum version `1.1.0`.** `v1.0.0` has no authentication on its HTTP transport
  (bearer auth and the refusal of unauthenticated non-loopback binds came in upstream #29,
  first released in `v1.1.0`). Because the image sets `OCIS_MCP_HTTP_ADDR=0.0.0.0:8090`,
  a `1.0.0` image in HTTP mode would expose every tool with the configured oCIS
  credentials and silently ignore `OCIS_MCP_HTTP_SECRET`. The resolver therefore never
  builds anything below `MIN_VERSION`. Do not lower it.
- **stdio default, smoke test by command.** See [Smoke test](#smoke-test). No volume (the
  server is stateless) and no `HEALTHCHECK` (no health endpoint; stdio has nothing to
  probe).
- **Release polling instead of a hand-edited matrix.** New upstream tags are built
  automatically, as in `web-extensions`, `ocis-groupware` and `ocis-agent`, rather than
  added to `main.yml` by a PR, as in `ocis`.
- **uid 10001 instead of 1000.** A fixed id, as in `ocis-agent`, so orchestrators can pin
  `runAsUser`. No image with another id was ever published.
- **No rolling or nightly image.** Only signed release tags are published. Upstream
  `main` is built only in pull requests, never pushed. The weekly rebuild of the newest
  release picks up Alpine package fixes via `apk upgrade`.
- **No `vN/` directory.** Only the newest release is rebuilt, so there is one root
  `Dockerfile.multiarch` and `.trivyignore`. Add a `vN/` directory only when a major line
  needs a different build.

## Development Conventions

- **No CHANGELOG** — do not create one.
- Conventional-Commit PR titles, enforced by `lint-pr-title.yml`.
- `.editorconfig` governs formatting; `LICENSE` is exempt from the indent rules.
- GitHub Actions are pinned to full commit SHAs.
- Workflows declare a least-privilege `permissions:` block.
- Scripts must stay `shellcheck`-clean; CI runs it.
- Bug reports for the application itself go upstream to
  [`owncloud/ocis-mcp-server`](https://github.com/owncloud/ocis-mcp-server); this repo
  tracks only the Docker packaging.

## OSPO Policy Constraints

### GitHub Actions
- **Only** use actions owned by `owncloud`, created by GitHub (`actions/*`),
  verified on the GitHub Marketplace, or verified by the ownCloud Maintainers.
- Pin all actions to their full commit SHA (not tags): `uses: actions/checkout@<SHA> # vX.Y.Z`.
- Never introduce actions from unverified third parties.

### Dependency Management
- Dependabot covers both GitHub Actions and Docker base-image updates for this
  repo (`.github/dependabot.yml`) — Renovate is not used here.
- Dependabot PRs do not auto-merge; review and merge them as part of regular
  maintenance. Merging one rebuilds and re-pushes the newest release.
- Base images in `Dockerfile.multiarch` must remain pinned as `tag@sha256:<digest>`;
  Dependabot updates the digest in place. The `golang` builder is the newest toolchain,
  not `go.mod`'s minimum. A stale `golang` digest surfaces as Go `stdlib` CVEs failing
  the Trivy gate; the fix is bumping the digest, not adding `.trivyignore` entries.
- A HIGH/CRITICAL CVE in a Go module compiled into the binary blocks the build until
  upstream ships a fix. Only when upstream cannot fix it promptly does it go into
  `.trivyignore`, with the CVE, the reason and the upstream issue.

### Git Workflow
- **Rebase policy**: Always rebase; never create merge commits.
- **Signed commits**: All commits **must** be PGP/GPG signed (`git commit -S`).
- **DCO sign-off**: Every commit needs a `Signed-off-by` line (`git commit -s`).
- **Conventional Commits & Squash Merge**: PR titles must follow
  [Conventional Commits](https://www.conventionalcommits.org/); the PR title
  becomes the squash-merge commit message and is enforced by CI.

## Context for AI Agents

- This is a Docker-image build repo that compiles the ocis-mcp-server from source —
  not the application codebase. Application changes belong upstream.
- The resolver is the only thing that decides what gets built: every verified upstream
  release at or above `MIN_VERSION` that is missing from Docker Hub. Do not reintroduce a
  "newest version only" reduction for pending builds — it drops a release when two land
  in one polling window.
- Do not change the uid/gid (10001), the entrypoint path or the stdio default without an
  explicit decision; MCP client configurations in the wild depend on them.
- Never bake credentials (`OCIS_MCP_APP_TOKEN_VALUE`, `OCIS_MCP_HTTP_SECRET`, OIDC
  tokens) into the image. The smoke test's `smoke` app token is a placeholder that never
  reaches an oCIS instance.
- The README is published verbatim as the Docker Hub image description — keep it
  accurate and self-contained.
- License is **Apache-2.0**, the OSPO's ecosystem-wide target; no relicensing needed.
