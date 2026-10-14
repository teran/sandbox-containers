# SPEC — sandbox-containers

> Technical specification for the `teran/sandbox-containers` repository.

## 1. Purpose

This repository builds and publishes **sandbox container images** for use with
[`mcp-sandbox`](https://github.com/teran/mcp-sandbox). Each image provides a
self-contained, unprivileged environment for a specific language ecosystem —
compilers/runtimes, package managers, and a linter/quality toolset — that
sandboxed agents use to build, test, and lint user code.

Every image is:

- built **multi-platform** for `linux/amd64` and `linux/arm64`;
- **deterministic and reproducible** (pinned versions + `sha256` verification);
- run as an **unprivileged user**;
- published to **GitHub Container Registry (GHCR)** under
  `ghcr.io/teran/sandbox-containers/<name>`.

## 2. Current contents

| Image | Directory | Runtime | Base |
|---|---|---|---|
| `golang` | `golang/` | Go 1.27.2 | `ubuntu:26.04` |

The only image today is the **Go sandbox** (`golang/`). See
[`golang/README.md`](golang/README.md) for its full contents and rationale.

### 2.1 The Go sandbox (`golang/`)

- **Go 1.27.2** installed from the official tarball into `/usr/local/go`;
  the environment mirrors the official `golang:1.27.2` image
  (`GOTOOLCHAIN=local`, `GOROOT=/usr/local/go`, `GOPATH=/go`).
- **Linter/quality toolset** (checksum-verified):
  `golangci-lint 2.14.0`, `govulncheck v1.1.4`, `go-arch-lint 1.19.0`,
  `gofumpt 0.12.0`, `gosec 2.29.0`, `gremlins 0.6.0`, `markdownlint-cli 0.49.1`.
- **Non-root** runtime user `1000:1000`; dedicated volumes for Go caches.
- System packages mirroring the cgo/base requirements of the official image
  (`gcc g++ make libc6-dev pkg-config binutils-gold` on arm64, git, etc.).

## 3. Build architecture

Each image is a **multi-stage Dockerfile**:

- **build stage** — downloads and verifies the runtime and all tools against
  their published `sha256` checksums;
- **final stage** — a minimal `ubuntu:26.04` base, copies the verified
  artifacts, configures the environment, and runs as an unprivileged user.

Architecture is handled via `TARGETARCH` (resolved by BuildKit for each target
platform), which maps directly to `GOARCH` and to the architecture-specific
asset names and checksums (`amd64`/`arm64`).

Multi-platform builds use **Docker Buildx with QEMU** (`docker/setup-qemu-action`
+ `docker/setup-buildx-action`), producing and pushing an image manifest for
`linux/amd64` and `linux/arm64`.

### 3.1 Reproducibility & verification

- **Versions** are pinned as `ARG`s at the top of each `Dockerfile`.
- **Binary releases** (Go tarball, golangci-lint, go-arch-lint, gremlins,
  gosec) are verified with `sha256sum -c` against official checksum files.
- **gofumpt** publishes no upstream checksum file → its `sha256` is pinned
  directly in the `Dockerfile`.
- **govulncheck** publishes no binary releases → installed via
  `go install ...@v<ver>` with integrity checked through the Go module
  checksum database (`sum.golang.org`).
- The `final` stage installs only what is needed at runtime; the `build` stage
  is discarded.

## 4. CI (GitHub Actions)

The workflow `.github/workflows/build.yml` ("Build sandbox images") runs on
`pull_request` and `push` to `master`, filtered by paths
(`golang/**`, `.github/workflows/**`).

### 4.1 Jobs

| Job | On PR | On push to `master` |
|---|---|---|
| `hadolint` | ✅ lint `golang/Dockerfile` | ✅ lint `golang/Dockerfile` |
| `build` | ✅ build multi-platform (no push) | ✅ build + **push** multi-platform |

- `hadolint` runs `hadolint/hadolint-action@v3.1.0` on `golang/Dockerfile`
  with `failure-threshold: error`.
- `build` `needs: hadolint`, so a lint failure blocks the build.

### 4.2 Tag derivation

The **Go version** is **not hardcoded** in the workflow. It is extracted from
the `Dockerfile`:

```bash
GO_VERSION="$(grep -oP 'GOLANG_VERSION=\K[0-9.]+' golang/Dockerfile | head -n1)"
```

On a **push** to `master` the image is tagged with three tags:
`<version>`, `<version>-<short-sha>`, `<version>-<short-sha>-<timestamp>`,
where `<short-sha>` is the first 7 characters of `GITHUB_SHA` and
`<timestamp>` is UTC `YYYYMMDDHHMMSS`. On a **pull request** only the
`<version>` tag is used for the build (never pushed).

### 4.3 Tag scheme

| Tag | Example | Meaning | Pushed |
|---|---|---|---|
| `<go-version>` | `1.27.2` | immutable, per Go release | ✅ push |
| `<go-version>-<short-sha>` | `1.27.2-4c8f2a1` | per commit, stable | ✅ push |
| `<go-version>-<short-sha>-<timestamp>` | `1.27.2-4c8f2a1-20241014120000` | per build, unique | ✅ push |
| `<go-version>` (PR build) | `1.27.2` | local build only | ❌ never pushed |

Pushing occurs only when `github.event_name == 'push'`.

### 4.4 Registry & authentication

- **Registry:** GitHub Container Registry (`ghcr.io`).
- **Image:** `ghcr.io/teran/sandbox-containers/golang`.
- **Login:** `docker/login-action@v3` with `registry: ghcr.io`,
  `username: ${{ github.actor }}`, `password: ${{ secrets.GITHUB_TOKEN }}`,
  performed **only** on push.
- **Permissions:** the `build` job declares
  `permissions: { contents: read, packages: write }`, which is what authorizes
  the `GITHUB_TOKEN` to push the image.

## 5. Repository structure

```
.
├── .github/
│   └── workflows/
│       └── build.yml          # CI: hadolint + build (+ push on master)
├── golang/
│   ├── Dockerfile             # Go 1.27.2 sandbox (multi-stage, multi-arch)
│   ├── README.md              # full Go-image documentation
│   └── .dockerignore          # keep the build context lean
├── LICENSE                    # Apache License 2.0
├── README.md                  # repository overview
├── AGENTS.md                  # instructions for AI agents
└── SPEC.md                    # this document
```

## 6. Adding a new sandbox image

New images are added as **subdirectories** (one directory per image):
`python/`, `node/`, `rust/`, etc. The Go image (`golang/`) is the reference
implementation to copy.

### 6.1 Procedure

1. **Create the image directory** — e.g. `python/` with a multi-stage,
   multi-arch `Dockerfile`, a `README.md`, and a `.dockerignore`.
2. **Pin everything** — runtime and tool versions as `ARG`s at the top;
   verify every downloaded artifact against a pinned/upstream `sha256`.
3. **Run as non-root** — end the final stage with `USER <uid>:<gid>` and
   writable volumes for caches.
4. **Extend the workflow** — in `.github/workflows/build.yml`:
   - add the new directory to the `paths:` filters (both `pull_request` and
     `push`);
   - set the `IMAGE` env var to the new image name
     (`ghcr.io/teran/sandbox-containers/<name>`);
   - point `hadolint` and `docker/build-push-action` at the new `Dockerfile`
     and `context`;
   - adjust the version-extraction `grep` to the new version `ARG`
     (e.g. `PYTHON_VERSION`).
5. **Document** — add the image to the table in this `SPEC.md` and to the
   root `README.md`; keep the image `README.md` in sync.
6. **Test** — open a PR; the workflow must pass `hadolint` and the
   multi-platform build on both `pull_request` and the eventual `master` push.

### 6.2 Workflow logic to adapt (reference)

- `paths` filters: `'<dir>/**'` and `'.github/workflows/**'`.
- `IMAGE`: `ghcr.io/teran/sandbox-containers/<name>`.
- Version extraction: `grep -oP '<ARG_NAME>=\K[0-9.]+' <dir>/Dockerfile`.
- `context` / `file`: the new subdirectory and its `Dockerfile`.

## 7. Security & reproducibility notes

- **Supply chain:** every downloaded binary is checksum-verified; no
  unverified `curl | sh` style installs.
- **Least privilege:** images run as `USER 1000:1000`, never root at runtime.
- **Determinism:** versions and checksums are pinned; base image `ubuntu:26.04`
  is pinned by version tag.
- **Verification in CI:** `hadolint` enforces Dockerfile best practices and
  fails on `error`-level findings.
