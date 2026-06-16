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

## Using the mirror

Make the published packages **public** (org/repo Packages settings) so they can
be pulled without authentication, then reference them in a `FROM`:

```dockerfile
FROM ghcr.io/<owner>/dotnet/sdk:8.0-bookworm-slim AS build
```

## Adding an image

Copy one of the existing workflows, change the `IMAGE` env value and the
`paths:` filter to the new file name, and commit. The first run publishes it.
