# sandbox-containers

Container images for use with [`mcp-sandbox`](https://github.com/teran/mcp-sandbox).

This repository builds and publishes **sandbox container images** — self-contained,
unprivileged environments for a language ecosystem (runtime, package managers,
and a linter/quality toolset). Every image is built **multi-platform**
(`linux/amd64` + `linux/arm64`) and published to **GitHub Container Registry**.

## Images

| Image | Directory | Runtime | Documentation |
|---|---|---|---|
| `golang` | [`golang/`](golang/) | Go 1.27.3 on `ubuntu:26.04` | [`golang/README.md`](golang/README.md) |
| `node` | [`node/`](node/) | Node.js 26.13.0 on `ubuntu:26.04` | [`node/README.md`](node/README.md) |

### Registry

Images are published to:

```
ghcr.io/teran/sandbox-containers/<name>
```

On every merge to `master` each image is pushed with three tags:

| Tag | Example |
|---|---|
| Runtime version | `1.27.3` (golang) / `26.13.0` (node) |
| Runtime version + commit SHA | `1.27.3-<short-sha>` / `26.13.0-<short-sha>` |
| Runtime version + commit SHA + timestamp | `1.27.3-<short-sha>-<yyyyMMddHHmmss>` / `26.13.0-<short-sha>-<yyyyMMddHHmmss>` |

The runtime version is extracted automatically from each image's `Dockerfile`,
so tags always stay in sync with the build.

## Build locally

```bash
# from the repo root
docker build -t sandbox-golang:1.27.3 -f golang/Dockerfile golang/
docker build -t sandbox-node:26.13.0 -f node/Dockerfile node/

# explicit platform
docker build --platform linux/amd64 -t sandbox-golang:1.27.3 -f golang/Dockerfile golang/
docker build --platform linux/arm64 -t sandbox-golang:1.27.3 -f golang/Dockerfile golang/
docker build --platform linux/amd64 -t sandbox-node:26.13.0 -f node/Dockerfile node/
docker build --platform linux/arm64 -t sandbox-node:26.13.0 -f node/Dockerfile node/
```

## Run

```bash
docker run --rm sandbox-golang:1.27.3 \
  sh -c 'go version && golangci-lint --version && markdownlint --version'

# run linters on your project (as user UID 1000)
docker run --rm -u 1000:1000 -v "$PWD":/go/src/app -w /go/src/app \
  sandbox-golang:1.27.3 sh -c 'gofumpt -l . && golangci-lint run ./...'

# Node: check the toolset
docker run --rm sandbox-node:26.13.0 \
  sh -c 'node --version && npm --version && pnpm --version && tsc --version'

# Node: lint + typecheck your project (as user UID 1000)
docker run --rm -u 1000:1000 -v "$PWD":/app -w /app \
  sandbox-node:26.13.0 sh -c 'npm ci && npm run lint && tsc --noEmit'
```

See [`golang/README.md`](golang/README.md) and
[`node/README.md`](node/README.md) for the full contents, versions, and usage
of each image.

## CI

GitHub Actions (`.github/workflows/build.yml`) runs on PRs and pushes to
`master` (path-filtered to `golang/**`, `node/**` and `.github/workflows/**`),
as a **matrix** over the `golang` and `node` images:

- **hadolint** lints each image's `Dockerfile` (fails on `error`-level findings).
- **build** builds each image multi-platform with Buildx + QEMU:
  - on **PR** — build only, no push;
  - on **push to `master`** — builds and **pushes** to
    `ghcr.io/teran/sandbox-containers/<name>` with the three tags above
    (runtime version extracted from each `Dockerfile`).

## Repository structure

```
.
├── .github/workflows/build.yml   # CI: lint + build (+ push on master), matrix golang, node
├── golang/                       # Go 1.27.3 sandbox image
│   ├── Dockerfile
│   ├── README.md
│   └── .dockerignore
├── node/                         # Node.js 26.13.0 sandbox image
│   ├── Dockerfile
│   ├── README.md
│   └── .dockerignore
├── LICENSE                       # Apache License 2.0
├── README.md                     # this file
├── AGENTS.md                     # instructions for AI agents
└── SPEC.md                       # technical specification
```

## Adding a new image

Add each image as its own subdirectory (e.g. `python/`) following the
`golang/` reference, and extend the workflow. See
[`SPEC.md`](SPEC.md#6-adding-a-new-sandbox-image) for the full procedure.

## Documentation

- [`SPEC.md`](SPEC.md) — technical specification (build architecture, CI, tag
  scheme, adding images).
- [`AGENTS.md`](AGENTS.md) — conventions and workflow for automated agents.

## License

[Apache License 2.0](LICENSE).
