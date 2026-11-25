# oCIS MCP Server Docker Image

[![Docker Pulls](https://img.shields.io/docker/pulls/owncloud/ocis-mcp-server.svg)](https://hub.docker.com/r/owncloud/ocis-mcp-server)
[![License: Apache-2.0](https://img.shields.io/github/license/owncloud-docker/ocis-mcp-server)](https://github.com/owncloud-docker/ocis-mcp-server/blob/main/LICENSE)
[![ownCloud OSPO](https://img.shields.io/badge/OSPO-ownCloud-blue)](https://kiteworks.com/opensource)

Docker image for the [oCIS MCP Server](https://github.com/owncloud/ocis-mcp-server) — a
[Model Context Protocol](https://modelcontextprotocol.io/) server that exposes
[ownCloud Infinite Scale (oCIS)](https://github.com/owncloud/ocis) to AI assistants as
80+ tools (users, groups, spaces, files, shares, search and more), using only the public
oCIS APIs. This repo builds and publishes the image; the application source lives in
[owncloud/ocis-mcp-server](https://github.com/owncloud/ocis-mcp-server).

> **Build & maintenance:** see [how these images are built, scanned, updated and published](https://github.com/owncloud-docker/.github/blob/master/docs/IMAGE-LIFECYCLE.md).

## Quick Start

By default the server speaks MCP over **stdio**: the MCP client starts the container and
talks to it through stdin/stdout. `-i` is required — without it stdin is closed and the
server exits immediately. For the same reason a detached (`-d`) or Compose container in
stdio mode exits at once with status 0; to run the server as a long-lived service, use the
[HTTP transport](#http-transport).

```bash
docker run -i --rm \
  -e OCIS_MCP_OCIS_URL=https://ocis.example.com \
  -e OCIS_MCP_APP_TOKEN_USER=alice \
  -e OCIS_MCP_APP_TOKEN_VALUE=<app-token> \
  owncloud/ocis-mcp-server:1.1.0
```

Create the app token in the oCIS web UI under **Settings > Security > App tokens** (oCIS
needs `OCIS_ADD_RUN_SERVICES=auth-app` and `PROXY_ENABLE_APP_AUTH=true`). The server
acts with that user's permissions.

### MCP client configuration

Most MCP clients (Claude Desktop, Claude Code, mcphost, …) accept a command to launch.
Pass the variable names with `-e NAME` and their values through the client's `env`, so
the token does not appear in the command line:

```json
{
  "mcpServers": {
    "ocis": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-e", "OCIS_MCP_OCIS_URL",
        "-e", "OCIS_MCP_APP_TOKEN_USER",
        "-e", "OCIS_MCP_APP_TOKEN_VALUE",
        "owncloud/ocis-mcp-server:1.1.0"
      ],
      "env": {
        "OCIS_MCP_OCIS_URL": "https://ocis.example.com",
        "OCIS_MCP_APP_TOKEN_USER": "alice",
        "OCIS_MCP_APP_TOKEN_VALUE": "<app-token>"
      }
    }
  }
}
```

### HTTP transport

To run the server as a long-lived service that clients reach over the network, switch to
the streamable HTTP transport. The endpoint is `/mcp`, and every request must carry
`Authorization: Bearer <OCIS_MCP_HTTP_SECRET>`:

```bash
umask 077
printf 'OCIS_MCP_HTTP_SECRET=%s\nOCIS_MCP_APP_TOKEN_VALUE=%s\n' "$(openssl rand -hex 32)" '<app-token>' > mcp.env

docker run -d --name ocis-mcp-server \
  --env-file mcp.env \
  -e OCIS_MCP_TRANSPORT=http \
  -e OCIS_MCP_OCIS_URL=https://ocis.example.com \
  -e OCIS_MCP_APP_TOKEN_USER=alice \
  -p 127.0.0.1:8090:8090 \
  --read-only \
  owncloud/ocis-mcp-server:1.1.0
```

The image listens on `0.0.0.0:8090` so the port can be published, and the server
**refuses to start** in HTTP mode without `OCIS_MCP_HTTP_SECRET`. Anyone holding the
secret can use every tool with the configured oCIS credentials: put TLS in front of it
and do not expose it publicly. See upstream's
[Securing the HTTP transport](https://github.com/owncloud/ocis-mcp-server#securing-the-http-transport).

## Configuration

All settings are environment variables:

| Variable | Default | Purpose |
|----------|---------|---------|
| `OCIS_MCP_OCIS_URL` | — (required) | Base URL of oCIS, e.g. `https://ocis.example.com` |
| `OCIS_MCP_AUTH_MODE` | auto-detected | `app-token` or `oidc`; required only when both kinds of credentials are set |
| `OCIS_MCP_APP_TOKEN_USER` | — | oCIS user the app token belongs to |
| `OCIS_MCP_APP_TOKEN_VALUE` | — | The app token (recommended authentication) |
| `OCIS_MCP_OIDC_ACCESS_TOKEN` | — | OIDC access token (alternative authentication) |
| `OCIS_MCP_OIDC_ISSUER`, `OCIS_MCP_OIDC_CLIENT_ID`, `OCIS_MCP_OIDC_CLIENT_SECRET` | — | OIDC client settings |
| `OCIS_MCP_EDUCATION_ACCESS_TOKEN` | — | Optional token for the oCIS Education API tools |
| `OCIS_MCP_TRANSPORT` | `stdio` | `stdio` or `http` |
| `OCIS_MCP_HTTP_ADDR` | `0.0.0.0:8090` in this image | Listen address of the HTTP transport (upstream default `127.0.0.1:8090`) |
| `OCIS_MCP_HTTP_SECRET` | — | Bearer token required on `/mcp`; mandatory in HTTP mode on a non-loopback address |
| `OCIS_MCP_LOG_LEVEL` | `info` | `debug`, `info`, `warn` or `error` (logs go to stderr) |
| `OCIS_MCP_HTTP_TIMEOUT` | `30s` | Timeout of requests to oCIS (Go duration) |
| `OCIS_MCP_INSECURE` | `false` | Allow a plaintext `http://` oCIS URL (development only) |
| `OCIS_MCP_TLS_SKIP_VERIFY` | `false` | Skip TLS certificate verification towards oCIS (development only) |

See the upstream [README](https://github.com/owncloud/ocis-mcp-server#readme) and
[Getting Started guide](https://github.com/owncloud/ocis-mcp-server/blob/main/GETTING_STARTED.md)
for authentication details and the full tool inventory.

## Supported Tags

| Tag | Meaning |
|-----|---------|
| `latest` | The highest released version |
| `X.Y.Z` | A specific release (e.g. `1.1.0`), rebuilt in place while it is the newest one |
| `X.Y.Z-YYYYMMDD` | The build of a release on a given day (e.g. `1.1.0-20261125`); never touched again after that day |

> **Note:** Only plain `vX.Y.Z` upstream release tags are built; there are no floating
> `X.Y` or `X` tags. Images start at `1.1.0`: `1.0.0` has no authentication on its HTTP
> transport, so it is not published. For production, pin an `X.Y.Z` or `X.Y.Z-YYYYMMDD`
> tag instead of `latest`. A second rebuild on the same day replaces that day's
> `X.Y.Z-YYYYMMDD` tag; pin by digest if you need the exact bytes.

```bash
docker pull owncloud/ocis-mcp-server:latest
```

## How New Releases Get Built

- **New releases:** CI checks the upstream repository for new release tags every 6 hours
  and builds every release that is not on Docker Hub yet. Maintainers can start the same
  check by hand (`workflow_dispatch`) instead of waiting.
- **Signed tags only:** a release is built only from an annotated upstream tag whose
  signature GitHub verifies. Other tags are skipped with a warning.
- **Security rebuilds:** the newest release is rebuilt and re-pushed whenever a change is
  merged into this repository, including base-image and Go toolchain updates, and once a
  week, picking up Alpine package fixes.
- **Scanning:** every build is scanned with Trivy and smoke-tested before it is
  published. An image with a fixable HIGH or CRITICAL vulnerability is never pushed.

## Volumes

None. The server keeps no state; the container can run with `--read-only`.

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| `8090` | TCP | MCP streamable HTTP endpoint `/mcp` — only with `OCIS_MCP_TRANSPORT=http` |

In the default stdio mode no port is opened. There is no health endpoint and no
`HEALTHCHECK`. In HTTP mode, probe the port over TCP (e.g. a Kubernetes `tcpSocket`
probe on `8090`): an HTTP probe fails, because `/mcp` answers `401` without the bearer
token and `/` answers `404`.

## User and Filesystem

The image is based on Alpine Linux and runs as uid/gid `10001` (`mcp`). The binary is
`/usr/local/bin/ocis-mcp-server`; the licence is in `/usr/share/doc/ocis-mcp-server/`.

## Build Arguments

| ARG | Default | Purpose |
|-----|---------|---------|
| `GIT_REF` | `main` | `ocis-mcp-server` git ref (tag or branch) to clone and build, e.g. `v1.1.0` |
| `GIT_SHA` | `""` | Optional exact commit to check out after cloning `GIT_REF`. CI passes the commit the verified tag points to |
| `VERSION` | `""` | Version reported by the server (`dev` when empty) and embedded in the OCI labels, e.g. `1.1.0` |
| `REVISION` | `""` | Git commit embedded in the OCI labels |
| `TARGETARCH` | set by buildx | Target architecture (`amd64`, `arm64`) |

## Building

The image is built entirely from source via a three-stage `Dockerfile.multiarch`:

**`source`** — clones `ocis-mcp-server` at `${GIT_REF}`, pinned to `${GIT_SHA}` when set.

**`build`** — compiles the static Go binary (`cmd/ocis-mcp-server`) with
`CGO_ENABLED=0` for the target architecture, embedding `${VERSION}`.

**Runtime** — Alpine with the binary, CA certificates and the licence.

```bash
docker buildx build -f Dockerfile.multiarch \
  --build-arg GIT_REF=v1.1.0 \
  --build-arg VERSION=1.1.0 \
  -t ocis-mcp-server:test .
```

## CI

The GitHub Actions workflow (`.github/workflows/main.yml`) runs on pull requests,
pushes to `main`, a schedule and by hand:

1. **Test** — lints the scripts and tests the release resolution logic.
2. **Resolve** — works out which releases to build (see
  [How New Releases Get Built](#how-new-releases-get-built)). A pull request always builds
  the newest release (or upstream `main` while there is none, or it does not verify), without publishing it.
3. **Build** — multi-arch image (`linux/amd64`, `linux/arm64`), built natively on each
  architecture.
4. **Trivy scan** — fails on HIGH/CRITICAL CVEs that have a fix; accepted exceptions are
  listed with a justification in `.trivyignore`.
5. **Smoke test** — sends an MCP `initialize` request over stdio and checks that the
  server reports the version being built.
6. **Publish** — pushes to Docker Hub and syncs this README as the image description
  (not on pull requests).

## Community & Support

- [ownCloud Website](https://owncloud.com)
- [Community Discussions](https://github.com/orgs/owncloud/discussions)
- [Matrix Chat](https://app.element.io/#/room/#owncloud:matrix.org)
- [Documentation](https://doc.owncloud.com)
- [Enterprise Support](https://owncloud.com/contact-us/)
- [OSPO Home](https://kiteworks.com/opensource)

See [SUPPORT.md](SUPPORT.md) for the full list of support channels.

## Contributing

We welcome contributions! Please read the [Contributing Guidelines](CONTRIBUTING.md)
and our [Code of Conduct](CODE_OF_CONDUCT.md) before getting started.

- **Rebase Early, Rebase Often!** We use a rebase workflow — rebase on the target
  branch before submitting a PR.
- **Signed commits**: All commits **must** be PGP/GPG signed and carry a DCO
  `Signed-off-by` line (`git commit -S -s`).
- **Conventional Commits**: PR titles must follow the
  [Conventional Commits](https://www.conventionalcommits.org/) format — enforced
  by CI.
- **GitHub Actions Policy**: Workflows may only use actions owned by `owncloud`,
  created by GitHub (`actions/*`), or verified in the GitHub Marketplace, pinned
  to a full commit SHA.

Note: bug reports and feature requests for the oCIS MCP Server itself belong upstream in
[`owncloud/ocis-mcp-server`](https://github.com/owncloud/ocis-mcp-server) — this repository
tracks only the Docker packaging.

## Security

**Do not open a public GitHub issue for security vulnerabilities.**

Report vulnerabilities at **<https://security.owncloud.com>** — see [SECURITY.md](SECURITY.md).

## About the ownCloud OSPO

The [Kiteworks Open Source Program Office](https://kiteworks.com/opensource), operating under
the [ownCloud](https://owncloud.com) brand, launched on May 5, 2026, to steward the open source
ecosystem around ownCloud's products. The OSPO ensures transparent governance, license compliance,
community health, and sustainable collaboration between the open source community and
[Kiteworks](https://www.kiteworks.com), which acquired ownCloud in 2023.

- **OSPO Home**: <https://kiteworks.com/opensource>
- **GitHub**: <https://github.com/owncloud>
- **ownCloud**: <https://owncloud.com>

For questions about the OSPO or licensing, contact ospo@kiteworks.com.

This repository is licensed under the **Apache License 2.0**, which is the license
the OSPO is adopting across the ecosystem. No relicensing is required.

## License

Apache-2.0 — see [LICENSE](LICENSE).
