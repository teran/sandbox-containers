# node-sandbox — Node.js 26.13.0 sandbox on a vanilla ubuntu:26.04

A sandbox from the image set, built on a **vanilla `ubuntu:26.04`**
(Resolute Raccoon, LTS): Node.js 26.13.0 (LTS "Lithium") is installed from the
official tarball into `/usr/local`, the environment mirrors the official
`node:26.13.0` image, and the linter/CLI toolset is the same as the golang
sandbox plus the npm toolchain.

## Image registry

This image is built and published by CI to GitHub Container Registry under
the full name:

```
ghcr.io/teran/sandbox-containers/node
```

On every merge to `master` the image is built for **linux/amd64** and
**linux/arm64** and pushed with the following tags:

| Tag | Example |
|---|---|
| Node version | `26.13.0` |
| Node version + commit SHA | `26.13.0-<short-sha>` |
| Node version + commit SHA + timestamp | `26.13.0-<short-sha>-<yyyyMMddHHmmss>` |

The Node version tag is extracted automatically from the `Dockerfile`
(`ARG NODE_VERSION=...`), so it always stays in sync with the build.
On every pull request the same image is built (multi-platform) but **not**
pushed, and the `Dockerfile` is linted with hadolint — see
`.github/workflows/`.

## Contents and versions

| Tool | Version | Installation | Checksum verification |
|---|---|---|---|
| Node.js | 26.13.0 | official tarball → `/usr/local` | ✅ official sha256 (nodejs.org SHASUMS256.txt) |
| pnpm | 12.16.1 | `npm install -g` | ✅ npm integrity hashes |
| typescript (tsc) | 7.0.2 | `npm install -g` | ✅ npm integrity hashes |
| eslint | 10.13.0 | `npm install -g` | ✅ npm integrity hashes |
| prettier | 3.9.10 | `npm install -g` | ✅ npm integrity hashes |
| markdownlint-cli | 0.50.0 | `npm install -g` | ✅ npm integrity hashes |
| yq | 4.54.1 | GitHub binary release | ✅ pinned sha256 (non-standard checksums format) |
| ripgrep (rg) | 15.2.0 | GitHub binary release (musl) | ✅ pinned sha256 (per-asset `.sha256` only) |
| fd | 10.5.0 | GitHub binary release | ✅ pinned sha256 (per-asset `.sha256` only) |
| jq | apt (universe) | apt package | ✅ apt-managed |
| shellcheck | apt (universe) | apt package | ✅ apt-managed |

Release sources:
- Node tarball: https://nodejs.org/dist/v26.13.0/node-v26.13.0-linux-<arch>.tar.gz
  (`<arch>` is `x64` for amd64, `arm64` for arm64 — Node names the amd64 asset
  `linux-x64`)
- pnpm: https://www.npmjs.com/package/pnpm
- typescript: https://www.npmjs.com/package/typescript
- eslint: https://www.npmjs.com/package/eslint
- prettier: https://www.npmjs.com/package/prettier
- markdownlint-cli: https://github.com/igorshubovych/markdownlint-cli/releases/tag/0.50.0
- yq: https://github.com/mikefarah/yq/releases/tag/v4.54.1
- ripgrep: https://github.com/BurntSushi/ripgrep/releases/tag/15.2.0
- fd: https://github.com/sharkdp/fd/releases/tag/v10.5.0

## Official sha256 of the Node.js 26.13.0 tarballs

| Architecture | File | sha256 |
|---|---|---|
| amd64 | `node-v26.13.0-linux-x64.tar.gz` | `5848553868b402d3fabb5b09270649977780a390f832155721025a08b5104455` |
| arm64 | `node-v26.13.0-linux-arm64.tar.gz` | `439e9c438acdf6a6ffe47418da709153d66c4bed812e30a473faacb107dd2791` |

Source: https://nodejs.org/dist/v26.13.0/SHASUMS256.txt (same values as in the
official `node:26.13.0` Dockerfile).

## libatomic dependency (Node.js >= 25)

Official Node.js **>= 25** Linux binaries are linked against `libatomic`.
Without it `node` fails to start with a missing `libatomic.so.1` loader error.
The final stage therefore installs **`libatomic1`** via apt. Node is **not**
executed in the build stage (which lacks libatomic1); its version is
smoke-tested only in the final stage.

## Environment (ENV), as in node:26.13.0

```dockerfile
ENV NODE_VERSION=26.13.0
ENV PNPM_HOME=/pnpm
ENV NPM_CONFIG_CACHE=/cache/npm
ENV PATH=/pnpm:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

`node`/`npm` land in `/usr/local/bin` (already on the base image PATH). pnpm
global shims land in `PNPM_HOME=/pnpm`, which is prepended to `PATH`.

### Non-root user (UID/GID 1000)

The image runs as an unprivileged user with **UID/GID 1000** (`USER 1000:1000`).
`ubuntu:26.04` already ships an `ubuntu` account with uid/gid 1000, so creation
is idempotent: the `user` group/user is created only if uid/gid 1000 are not
already taken. `USER` is numeric so the image does not depend on the name of
the existing account.

Tools are installed as root, but the runtime (CMD `node --version`) and
commands run as the unprivileged user. `PNPM_HOME=/pnpm` and the npm cache
`/cache` are owned by uid 1000, and the user's home (`/home/user`, created by
`useradd --create-home`) hosts the pnpm store — so `pnpm add -g`, `npm i`, and
the package caches all work without root.

### Volumes (VOLUME)

Volumes for temp files and caches are declared so they can be reused across
container runs (useful for recurring installs/linters):

| Volume | Purpose |
|---|---|
| `/tmp` | temp files |
| `/cache` | npm cache (`NPM_CONFIG_CACHE=/cache/npm`) |
| `/pnpm` | pnpm global home (`PNPM_HOME`) |

The pnpm store itself lives under the user's home (`/home/user`), which is
persisted/owned by uid 1000.

### System packages

The official `node:26.13.0` is built on `buildpack-deps:bookworm-scm`
(git, ca-certificates, curl, wget, openssh-client, etc.). On ubuntu:26.04 this
maps to:

- base set: `ca-certificates curl wget git openssh-client`
- required by Node >= 25: `libatomic1`
- sandbox toolset: `jq`, `shellcheck`
- on `arm64` additionally: `binutils-gold`

## How checksum verification is implemented

**Node tarball** — downloaded from
`https://nodejs.org/dist/v26.13.0/node-v26.13.0-linux-<arch>.tar.gz` and
verified against the official sha256 (the same trick as in the official node
image — `echo "$sha256 *<file>" | sha256sum -c -`), then extracted into
`/usr/local` with `--strip-components=1`:

```dockerfile
asset="node-v${NODE_VERSION}-linux-${arch}.tar.gz"  # arch = x64 | arm64
curl -fsSL -o "/tmp/${asset}" "https://nodejs.org/dist/v${NODE_VERSION}/${asset}"
(cd /tmp && echo "${sha256} *${asset}" | sha256sum -c -)
tar -C /usr/local --strip-components=1 -xzf "/tmp/${asset}"
```

**yq / ripgrep / fd** — these projects do not publish a file that
`sha256sum -c` can consume directly (yq's `checksums` file uses a non-standard
multi-hash format; ripgrep and fd ship only per-asset `.sha256` files). Their
`sha256` values are pinned in the `Dockerfile` as
`<TOOL>_SHA256_AMD64/ARM64` ARGs (identical to the golang image) and verified
before installation: `echo "${expected}  ${asset}" | sha256sum -c -`.

**pnpm / typescript / eslint / prettier / markdownlint-cli** — installed via
`npm install -g <pkg>@<version>`; integrity is verified by npm's integrity
hashes.

**jq / shellcheck** — installed from Ubuntu's `universe` repository via `apt`;
integrity is managed by apt (Debian package signing).

## Build

```bash
cd node
docker build -t node-sandbox:26.13.0 .
# explicit platform
docker build --platform linux/amd64 -t node-sandbox:26.13.0 .
docker build --platform linux/arm64 -t node-sandbox:26.13.0 .
```

## Run

```bash
docker run --rm node-sandbox:26.13.0 \
  sh -c 'node --version && npm --version && pnpm --version && tsc --version'

# lint a project (as user UID 1000)
docker run --rm -u 1000:1000 -v "$PWD":/work -w /work node-sandbox:26.13.0 \
  sh -c 'pnpm install && eslint . && prettier --check .'

# interactive session
docker run --rm -it -u 1000:1000 -v "$PWD":/work -w /work node-sandbox:26.13.0 bash
```

## Conflicts / decisions

1. **yq / ripgrep / fd have no `sha256sum -c`-compatible checksum file** →
   checksums are pinned in the Dockerfile (identical values to the golang
   image).
2. **Node >= 25 needs libatomic** → `libatomic1` is installed in the final
   stage; Node is deliberately not executed in the build stage.
3. **Architecture**: `TARGETARCH` (amd64/arm64) maps to the Node tarball names
   (`x64` for amd64, `arm64` for arm64) and to the yq/ripgrep/fd asset names
   (`x86_64`/`aarch64` arch naming and per-tool musl/gnu build selection, as in
   golang/).

## Structure

```
node/
├── Dockerfile
├── README.md
└── .dockerignore
```
