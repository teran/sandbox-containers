# golang-sandbox — Go 1.27.3 sandbox on a vanilla ubuntu:26.04

A sandbox from the image set, built on a **vanilla `ubuntu:26.04`**
(Resolute Raccoon, LTS): Go 1.27.3 is installed from the official tarball,
the environment mirrors the official `golang:1.27.3` image, and the toolset
is the same.

## Image registry

This image is built and published by CI to GitHub Container Registry under
the full name:

```
ghcr.io/teran/sandbox-containers/golang
```

On every merge to `master` the image is built for **linux/amd64** and
**linux/arm64** and pushed with the following tags:

| Tag | Example |
|---|---|
| Go version | `1.27.3` |
| Go version + commit SHA | `1.27.3-<short-sha>` |
| Go version + commit SHA + timestamp | `1.27.3-<short-sha>-<yyyyMMddHHmmss>` |

The Go version tag is extracted automatically from the `Dockerfile`
(`ARG GOLANG_VERSION=...`), so it always stays in sync with the build.
On every pull request the same image is built (multi-platform) but **not**
pushed, and the `Dockerfile` is linted with hadolint — see
`.github/workflows/`.

## Contents and versions

| Tool | Version | Installation | Checksum verification |
|---|---|---|---|
| Go (golang) | 1.27.3 | official tarball → `/usr/local/go` | ✅ official sha256 (go.dev/dl/?mode=json) |
| golangci-lint | 2.15.0 | GitHub binary release | ✅ `sha256sum -c` against `golangci-lint-2.15.0-checksums.txt` |
| govulncheck | v1.9.0 | `go install` (no binary releases) | ✅ via sum.golang.org |
| go-arch-lint | 1.19.0 | GitHub binary release | ✅ `sha256sum -c` against `checksums.txt` |
| gofumpt | 0.12.0 | GitHub binary release | ✅ pinned sha256 (no upstream checksums) |
| gosec | 2.29.0 | GitHub binary release | ✅ `sha256sum -c` against `gosec_2.29.0_checksums.txt` |
| gremlins | 0.6.0 | GitHub binary release | ✅ `sha256sum -c` against `checksums.txt` |
| yq | 4.54.1 | GitHub binary release | ✅ pinned sha256 (non-standard checksums format) |
| ripgrep (rg) | 15.2.0 | GitHub binary release (musl) | ✅ pinned sha256 (per-asset `.sha256` only) |
| fd | 10.5.0 | GitHub binary release | ✅ pinned sha256 (per-asset `.sha256` only) |
| mockery | v3.8.1 | `go install` (no binary releases) | ✅ via sum.golang.org |
| go-licenses | v2.0.1 | `go install` (no binary releases) | ✅ via sum.golang.org |
| goimports | v0.52.0 | `go install` (no binary releases) | ✅ via sum.golang.org |
| nilaway | pseudo-version `v0.0.0-…-acb8859b9031` | `go install` (no binary releases/tags) | ✅ via sum.golang.org |
| jq | apt (universe) | apt package | ✅ apt-managed |
| prettier | 3.9.10 | npm (native nodejs from apt) | ✅ npm integrity hashes |
| markdownlint-cli | 0.49.1 | npm (native nodejs from apt) | ✅ npm integrity hashes |

Release sources:
- Go tarball: https://go.dev/dl/go1.27.3.linux-<arch>.tar.gz (or https://dl.google.com/go/)
- golangci-lint: https://github.com/golangci/golangci-lint/releases/tag/v2.15.0
- govulncheck: https://github.com/golang/vuln/releases/tag/v1.9.0
- go-arch-lint: https://github.com/fe3dback/go-arch-lint/releases/tag/v1.19.0
- gofumpt: https://github.com/mvdan/gofumpt/releases/tag/v0.12.0
- gosec: https://github.com/securego/gosec/releases/tag/v2.29.0
- gremlins: https://github.com/go-gremlins/gremlins/releases/tag/v0.6.0
- yq: https://github.com/mikefarah/yq/releases/tag/v4.54.1
- ripgrep: https://github.com/BurntSushi/ripgrep/releases/tag/15.2.0
- fd: https://github.com/sharkdp/fd/releases/tag/v10.5.0
- mockery: https://github.com/vektra/mockery/releases/tag/v3.8.1
- go-licenses: https://github.com/google/go-licenses/releases/tag/v2.0.1
- goimports: https://github.com/golang/tools/releases/tag/v0.52.0
- nilaway: https://github.com/uber-go/nilaway (no releases/tags; pseudo-version from module proxy)
- markdownlint-cli: https://github.com/igorshubovych/markdownlint-cli/releases/tag/0.49.1
- prettier: https://www.npmjs.com/package/prettier

## Official sha256 of the Go 1.27.3 tarballs

| Architecture | File | sha256 |
|---|---|---|
| amd64 | `go1.27.3.linux-amd64.tar.gz` | `65efbc80fdc1fa2cd2631cad5b481777a68e8725f34e89f50fc6ac714e379b40` |
| arm64 | `go1.27.3.linux-arm64.tar.gz` | `a06e99309dbcbdca1a8ccd2282c3ee146e55930886e26b4f1193269587165f49` |

Source: https://go.dev/dl/?mode=json (same values as in the official
`golang:1.27.3` Dockerfile).

## Environment (ENV), as in golang:1.27.3

```dockerfile
ENV GOLANG_VERSION=1.27.3
ENV GOTOOLCHAIN=local
ENV GOROOT=/usr/local/go
ENV GOPATH=/go
ENV PATH=/go/bin:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

`GOROOT=/usr/local/go` and `GOTOOLCHAIN=local` reproduce the behaviour of the
official image (there `GOROOT` is auto-detected from the binary location, but
the value is identical; we set it explicitly).

The standard GOPATH layout is created as in the official image:
`mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 1777 "$GOPATH"`.

### Non-root user (UID/GID 1000)

The image runs as an unprivileged user with **UID/GID 1000** (`USER 1000:1000`).
`ubuntu:26.04` already ships an `ubuntu` account with uid/gid 1000, so creation
is idempotent: the `user` group/user is created only if uid/gid 1000 are not
already taken. `USER` is numeric so the image does not depend on the name of
the existing account.

Tools are installed as root, but the runtime (CMD `go version`) and commands
run as the unprivileged user. `GOPATH=/go` has sticky `1777` permissions and
belongs to uid 1000, so `go install`/`go build` and the module cache work
without root.

### Volumes (VOLUME)

Volumes for temp files and Go caches are declared so they can be reused across
container runs (useful for recurring linters/builds):

| Volume | Purpose |
|---|---|
| `/tmp` | temp files |
| `/go/cache` | `GOCACHE` — Go build cache |
| `/go/pkg/mod` | `GOMODCACHE` — Go module cache |

`GOCACHE` and `GOMODCACHE` are set via `ENV`. Volumes can be mounted at run
time (e.g. a shared module cache for the whole CI):
```bash
docker run --rm -v gocache:/go/cache -v gomodcache:/go/pkg/mod \
  -v "$PWD":/go/src/app -w /go/src/app golang-sandbox:1.27.3 \
  sh -c 'golangci-lint run ./...'
```

### System packages

The official `golang:1.27.3` is built on `buildpack-deps:bookworm-scm`
(git, ca-certificates, curl, wget, openssh-client, etc.) and additionally
installs cgo dependencies in its final stage. On ubuntu:26.04 this maps to:

- base set: `ca-certificates curl wget git openssh-client`
- cgo dependencies: `gcc g++ make libc6-dev pkg-config`
- on `arm64` additionally: `binutils-gold`
- `jq` (universe, for JSON processing in the sandbox)
- markdownlint + prettier: `nodejs npm`

## How Node / markdownlint-cli is installed on ubuntu:26.04

markdownlint-cli 0.49.1 requires **Node >= 22**.

**Decision: native installation via apt.** Verified that Ubuntu 26.04
(Resolute Raccoon) ships in universe:
- `nodejs` = **22.22.1** (≥ 22 — markdownlint requirement met),
- `npm` = **9.2.0~ds3-1**,
- glibc (`libc6`) = **2.43**.

So `apt-get install -y nodejs npm` is used, followed by
`npm install -g markdownlint-cli@0.49.1`. This is a native approach with no
glibc-compatibility issues and no third-party repositories (nodesource not
needed).

**Why not an intermediate node stage:** in the earlier (golang/trixie) variant
a node stage was required because the base image had no Node at all. Here
ubuntu:26.04 already provides node 22 from apt — the native path is simpler,
cleaner and fully reproducible. (The alternative — copying a binary from
`node:22-bookworm-slim` with glibc 2.36 ≤ 2.43 — is also compatible but
redundant.)

## How checksum verification is implemented

**Go tarball** — downloaded from
`https://dl.google.com/go/go1.27.3.linux-<arch>.tar.gz` and verified against
the official sha256 (the same trick as in the official golang image —
`echo "$sha256 *<file>" | sha256sum -c -`):

```dockerfile
asset="go${GOLANG_VERSION}.linux-${TARGETARCH}.tar.gz"
curl -fsSL -o "/tmp/${asset}" "https://dl.google.com/go/${asset}"
(cd /tmp && echo "${sha256} *${asset}" | sha256sum -c -)
tar -C /usr/local -xzf "/tmp/${asset}"
```

**golangci-lint / go-arch-lint / gremlins / gosec** — the official release
checksums files are downloaded and verified via `sha256sum -c --ignore-missing`
(only the asset downloaded for the current `TARGETARCH` is checked):

```dockerfile
curl -fsSL -o "${asset}" ".../download/v${VER}/${asset}"
curl -fsSL -o checksums.txt ".../download/v${VER}/checksums.txt"
sha256sum -c --ignore-missing checksums.txt
```

**gofumpt / yq / ripgrep / fd** — these projects do not publish a file that
`sha256sum -c` can consume directly (gofumpt ships bare binaries; yq's
`checksums` file uses a non-standard multi-hash format; ripgrep and fd ship
only per-asset `.sha256` files). Their `sha256` values are pinned in the
`Dockerfile` as `<TOOL>_SHA256_AMD64/ARM64` ARGs and verified before
installation: `echo "${expected}  ${asset}" | sha256sum -c -`.

**govulncheck / mockery / go-licenses / goimports / nilaway** — these projects
publish no binary releases (or, in the case of nilaway, no release tags at
all), so `go install <path>@<pinned-version>` is used, with integrity verified
via the Go module checksum database (`sum.golang.org`). Note the pinned
versions are the latest *released* tags where they exist; nilaway has no
release tags, so its pinned version is a module pseudo-version
(`v0.0.0-20260918162853-acb8859b9031`) resolved through the Go module proxy.

**jq** — installed from Ubuntu's `universe` repository via `apt`; integrity is
managed by apt (Debian package signing).

**prettier / markdownlint-cli** — installed natively via npm on the apt-provided
nodejs (see the section above); integrity is verified by npm's integrity hashes.

## Build

```bash
cd golang
docker build -t golang-sandbox:1.27.3 .
# explicit platform
docker build --platform linux/amd64 -t golang-sandbox:1.27.3 .
docker build --platform linux/arm64 -t golang-sandbox:1.27.3 .
```

## Run

```bash
docker run --rm golang-sandbox:1.27.3 \
  sh -c 'go version && golangci-lint --version && markdownlint --version'

# linters on your project
docker run --rm -v "$PWD":/go/src/app -w /go/src/app golang-sandbox:1.27.3 \
  sh -c 'gofumpt -l . && golangci-lint run ./...'

# interactive session as user (UID 1000)
docker run --rm -it -u 1000:1000 -v "$PWD":/go/src/app -w /go/src/app golang-sandbox:1.27.3 bash
```

## Conflicts / decisions

1. **govulncheck / mockery / go-licenses / goimports / nilaway cannot be
   downloaded from a binary release** → installed via `go install` with
   sum.golang.org verification.
2. **gofumpt / yq / ripgrep / fd have no `sha256sum -c`-compatible checksum
   file** → checksums are pinned in the Dockerfile (computed from the official
   release assets; yq's `checksums` uses a non-standard multi-hash format,
   ripgrep/fd ship only per-asset `.sha256`).
3. **Node**: native apt on ubuntu:26.04 (nodejs 22.22.1 ≥ 22) instead of a
   node stage — see the section above. Both `prettier` and `markdownlint-cli`
   are installed through this npm.
4. **Architecture**: `TARGETARCH` (amd64/arm64) matches GOARCH and asset
   names; name differences are handled (golangci-lint in an archive
   subdirectory, go-arch-lint/gremlins in the archive root, gofumpt/yq as
   extracted binaries, ripgrep/fd with `x86_64`/`aarch64` arch names and
   per-tool musl/gnu build selection).

## prettier vs markdownlint: formatting vs linting

Both are npm tools installed on the same apt nodejs, but they do **different**
jobs and it is intentional that they overlap on Markdown:

- **prettier** *formats* (rewrites) JS/TS/JSON/YAML/Markdown/HTML/CSS. Run it
  to enforce a consistent code style:
  ```bash
  prettier --write '**/*.{js,ts,json,yaml,md}'
  ```
- **markdownlint-cli** *lints* Markdown (reports rule violations: line length,
  heading structure, trailing spaces, etc.) and does not rewrite files:
  ```bash
  markdownlint '**/*.md'
  ```

So for Markdown: **prettier formats, markdownlint lints** — they are not
interchangeable and both are included deliberately.

## Structure

```
golang/
├── Dockerfile
├── README.md
└── .dockerignore
```
