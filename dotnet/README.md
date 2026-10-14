# dotnet

A mixin that installs the [.NET SDK](https://dotnet.microsoft.com) into any
agent sandbox, making `dotnet` available on PATH and setting the standard
`DOTNET_ROOT` environment variable. The SDK channel is configurable — default
is 9.0, but any valid `dotnet-install.sh` channel (e.g. `8.0`, `10.0`) can be
supplied as a kit argument.

## Usage

`dotnet-mixin` is agent-agnostic — pair it with whichever agent you're using.
From its published OCI artifact on Docker Hub:

```console
sbx run claude --kit "docker.io/sbx/dotnet-kit:latest" ~/my-project
```

Or from a git URL targeting this repo:

```console
sbx run claude --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=dotnet" ~/my-project
sbx run kiro   --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=dotnet" ~/my-project
```

Or from a local checkout:

```console
sbx run claude --kit ./dotnet/ ~/my-project
```

To request a specific SDK channel, add `--kit-arg`:

```console
sbx run claude \
  --kit "docker.io/sbx/dotnet-kit:latest" \
  --kit-arg "dotnet.version=8.0" \
  ~/my-project
```

Once the sandbox is running, `dotnet` is on PATH:

```console
agent@sandbox:~$ dotnet --version
9.0.x
agent@sandbox:~$ dotnet new console -o hello && cd hello && dotnet run
Hello, World!
```

## Arguments

| Argument | Default | Description |
|---|---|---|
| `version` | `9.0` | The .NET SDK channel passed to `dotnet-install.sh --channel`. Accepts any `X.Y` channel string, e.g. `8.0`, `9.0`, `10.0`. |

## Network policy

The following domains are added to the sandbox allow-list:

| Domain | Purpose |
|---|---|
| `archive.ubuntu.com` | `apt-get install wget` (amd64) |
| `security.ubuntu.com` | Security updates (amd64) |
| `ports.ubuntu.com` | `apt-get` for arm64 (Apple Silicon / Graviton) |
| `download.docker.com` | Docker's apt repo, pre-added by the shell-docker template; `apt-get update` fails if any configured source is unreachable |
| `dot.net` | Entry URL for the `dotnet-install.sh` script (redirects) |
| `builds.dotnet.microsoft.com` | Redirect target — the script is served from here |
| `dotnetcli.azureedge.net` | SDK tarballs (primary CDN) |
| `dotnetbuilds.azureedge.net` | SDK tarballs (secondary CDN) |
| `azurecom.azureedge.net` | Azure CDN edge used by the installer |
| `api.nuget.org` | NuGet package restore at runtime |
| `nuget.org` | NuGet package listings at runtime |

## How it works

The install step downloads `dotnet-install.sh` from `dot.net` (which redirects
to `builds.dotnet.microsoft.com`), installs the requested SDK channel into
`/usr/share/dotnet`, and force-symlinks the binary to `/usr/local/bin/dotnet`.
`set -euo pipefail` ensures any failure in this chain surfaces immediately
rather than silently producing a sandbox without a working SDK.

`DOTNET_ROOT` is set to `/usr/share/dotnet`, `DOTNET_CLI_TELEMETRY_OPTOUT` to
`1`, and `DOTNET_NOLOGO` to `true` — keeping command output clean inside the
sandbox.

## Scope

This kit installs the SDK only. It does not pre-populate a NuGet cache,
create a `global.json`, or configure any particular target framework — those
decisions belong in the project being worked on. If `dotnet restore` or
`dotnet build` needs to reach package sources beyond `nuget.org`, add the
relevant domains in a composed kit or via `sbx policy allow network`.
