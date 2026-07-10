# Containers

A mirror of upstream container base images, republished to GHCR.

## Why

Pulling base images directly from public registries on every build runs into
two problems:

- **Rate limits.** Anonymous and shared-IP pulls (CI runners in particular) are
  throttled by the upstream registry, causing flaky, slow builds.
- **Outages.** When the upstream registry has an incident, every build that
  pulls from it fails.

Mirroring the images we depend on into GHCR removes both: GHCR has its own,
much higher rate limits, and the mirror keeps serving the last good copy even
while the upstream is down.

## What's mirrored

The .NET base images from the Microsoft Container Registry (`mcr.microsoft.com`,
backed by Azure Container Registry):

| Upstream (MCR)                                                      | Mirror (GHCR)                                                     |
| ------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim`                    | `ghcr.io/<owner>/dotnet/sdk:8.0-bookworm-slim`                   |
| `mcr.microsoft.com/dotnet/aspnet:8.0-bookworm-slim`                 | `ghcr.io/<owner>/dotnet/aspnet:8.0-bookworm-slim`                |
| `mcr.microsoft.com/dotnet/aspnet:8.0-azurelinux3.0-distroless-extra` | `ghcr.io/<owner>/dotnet/aspnet:8.0-azurelinux3.0-distroless-extra` |

Their roles in the Craft build: the **SDK** compiles/publishes the app, the
**Debian aspnet** stages libmimalloc, and the **Azure Linux distroless-extra
aspnet** is the runtime base of the shipped (distroless) image.

The path after the registry host is preserved verbatim, and `<owner>` is the
lower-cased repository owner.

## How it works

Each image has its own workflow:

- [`.github/workflows/mirror-dotnet-sdk.yml`](.github/workflows/mirror-dotnet-sdk.yml)
- [`.github/workflows/mirror-dotnet-aspnet.yml`](.github/workflows/mirror-dotnet-aspnet.yml)
- [`.github/workflows/mirror-dotnet-aspnet-distroless.yml`](.github/workflows/mirror-dotnet-aspnet-distroless.yml)

Each runs weekly (Monday 06:00 UTC), on demand (`workflow_dispatch`), and when
its own workflow file changes. They authenticate to GHCR with the default
`GITHUB_TOKEN` (`packages: write`) and use `docker buildx imagetools create` to
copy the full multi-arch manifest (amd64 + arm64) straight between registries —
layers are mounted, not pulled into the runner. Only the upstream tag is
mirrored.

Before that copy runs, the upstream image is vulnerability-scanned — see
[Security scanning](#security-scanning) below.

## Security scanning

Every mirror workflow scans the **upstream** image with
[Trivy](https://github.com/aquasecurity/trivy-action) before it is copied into
GHCR, so you always know what you're mirroring. Trivy pulls the layers itself,
so the scan works even though the copy step never pulls the image into the
runner.

The scan is **report-only** — it never blocks the publish. The mirror copies an
unmodified upstream image byte-for-byte; a CVE in it isn't ours to remediate
(only the upstream vendor can, by shipping a patched tag, which then flows
through the mirror automatically). Failing the job on such a CVE wouldn't fix
anything — it would just stall the mirror on the last good copy, defeating the
availability the mirror exists to provide. So instead of gating, each run:

- **Logs** the full `CRITICAL`/`HIGH` result as a table in the Actions log, so
  the findings are visible on every run.
- **Uploads** the same result as SARIF to the repository's **Security → Code
  scanning** tab for an audit trail and diff-over-time. This upload is
  best-effort (it needs code scanning enabled) and never blocks the mirror.

Note that .NET SDK images in particular carry `HIGH` findings in Microsoft's
bundled dev tooling (`dotnet-format`, MSBuild, F#) — these clear when Microsoft
ships a new SDK patch tag, which the weekly mirror picks up automatically.

The scan runs on the runner's native architecture (`linux/amd64`); the full
multi-arch manifest is still what gets mirrored. To change what's reported —
`severity`, or to switch to a blocking gate with `exit-code: "1"` — edit the
`Scan … for vulnerabilities` steps in each workflow.

## Using the mirror

Make the published packages **public** (org/repo Packages settings) so they can
be pulled without authentication, then reference them in a `FROM`:

```dockerfile
FROM ghcr.io/<owner>/dotnet/sdk:8.0-bookworm-slim AS build
```

## Adding an image

Copy one of the existing workflows, change the `IMAGE` env value and the
`paths:` filter to the new file name, and commit. Also give the
`Upload scan results to the Security tab` step a unique `category:` (e.g. the
new file name) so its scan results don't collide with the other workflows' in
the Security tab. The first run scans and publishes it.
