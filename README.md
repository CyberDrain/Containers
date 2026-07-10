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

| Upstream (MCR)                                      | Mirror (GHCR)                                          |
| --------------------------------------------------- | ------------------------------------------------------ |
| `mcr.microsoft.com/dotnet/sdk:8.0-bookworm-slim`    | `ghcr.io/<owner>/dotnet/sdk:8.0-bookworm-slim`         |
| `mcr.microsoft.com/dotnet/aspnet:8.0-bookworm-slim` | `ghcr.io/<owner>/dotnet/aspnet:8.0-bookworm-slim`      |

The path after the registry host is preserved verbatim, and `<owner>` is the
lower-cased repository owner.

## How it works

Each image has its own workflow:

- [`.github/workflows/mirror-dotnet-sdk.yml`](.github/workflows/mirror-dotnet-sdk.yml)
- [`.github/workflows/mirror-dotnet-aspnet.yml`](.github/workflows/mirror-dotnet-aspnet.yml)

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
GHCR, so a base image with a known, fixable high-severity CVE never lands in the
mirror. Trivy pulls the layers itself, so the scan works even though the copy
step never pulls the image into the runner.

Two things happen on each run:

- **Gate.** The job fails on any `CRITICAL` or `HIGH` vulnerability that has a
  fix available (`ignore-unfixed: true`). When the gate fails the mirror step is
  skipped, so the previous good copy in GHCR is left untouched. Unfixed CVEs —
  which are common and unavoidable in Debian base images — don't block the
  mirror; they'd otherwise stop it from ever publishing.
- **Report.** The full `CRITICAL`/`HIGH` result (fixed and unfixed) is uploaded
  as SARIF to the repository's **Security → Code scanning** tab for an audit
  trail, even when the gate passes. This upload is best-effort (it needs
  code-scanning enabled) and never blocks the mirror.

The scan runs on the runner's native architecture (`linux/amd64`); the full
multi-arch manifest is still what gets mirrored. Thresholds live in the
`Gate on fixable CRITICAL/HIGH vulnerabilities` step of each workflow — adjust
`severity` or `ignore-unfixed` there to tune what blocks a publish.

## Using the mirror

Make the published packages **public** (org/repo Packages settings) so they can
be pulled without authentication, then reference them in a `FROM`:

```dockerfile
FROM ghcr.io/<owner>/dotnet/sdk:8.0-bookworm-slim AS build
```

## Adding an image

Copy one of the existing workflows, change the `IMAGE` env value and the
`paths:` filter to the new file name, and commit. The first run publishes it.
Also give the `Upload scan results` step a unique `category:` (e.g. the new file
name) so its scan results don't collide with the other workflows' in the
Security tab. The first run scans and, if the gate passes, publishes it.
