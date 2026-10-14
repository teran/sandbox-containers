# syntax=docker/dockerfile:1
#
# Go 1.27.2 sandbox on a VANILLA ubuntu:26.04 (Resolute Raccoon) base.
# Go is installed from the official tarball https://go.dev/dl/ (dl.google.com)
# into /usr/local/go; the environment mirrors the official golang:1.27.2 image,
# and the linter toolset is the same as the root golang sandbox.
#
# Principles:
#   * Versions and checksums are pinned as ARG at the top.
#   * The Go tarball is fetched from https://dl.google.com/go/go1.27.2.linux-<arch>.tar.gz
#     and verified against the official sha256 (published at https://go.dev/dl/?mode=json).
#   * Tools distributed as binary releases are downloaded from GitHub and
#     verified by checksum before installation.
#   * Architecture is TARGETARCH (amd64/arm64), matching GOARCH and asset names.
#   * markdownlint-cli is installed natively via apt (nodejs 22.22.1 >= 22); see README.

# ============================================================================
# Versions (deterministic)
# ============================================================================
ARG UBUNTU_VERSION=26.04

ARG GOLANG_VERSION=1.27.2
# Official sha256 of the Go tarballs (https://go.dev/dl/?mode=json)
ARG GO_SHA256_AMD64=ecbadb99091a3f46e31f5f934b068b1864eafa7995211b39eaddf76996045fe5
ARG GO_SHA256_ARM64=94f3e30b8e374bc285e7dadc11e0865726b9bc6e85b841ccceaabc0214c6b7c8

# Go tools (GitHub binary releases)
ARG GOLANGCI_LINT_VERSION=2.14.0
ARG GOARCHLINT_VERSION=1.19.0
ARG GREMLINS_VERSION=0.6.0
ARG GOFUMPT_VERSION=0.12.0
ARG GOSEC_VERSION=2.29.0

# govulncheck does NOT publish binary releases -> go install (sum.golang.org)
ARG GOVULNCHECK_VERSION=v1.1.4

# yq / ripgrep / fd do not publish a sha256sum-compatible checksums.txt
# (yq's `checksums` file uses a non-standard multi-hash format; rg/fd ship
# per-asset .sha256 files) -> checksums pinned here as ARGs.
ARG YQ_VERSION=4.54.1
ARG YQ_SHA256_AMD64=e68a456f90c577af3fe4960184b3a3cf5c461e0348407c10f107da3a5fec8972
ARG YQ_SHA256_ARM64=8c27f6e7476fb5acefb9352e3d30f6bc2dc93f1956eb92e8da727173fb7ec6d3

ARG RIPGREP_VERSION=15.2.0
# ripgrep 15.2.0 ships a musl build for amd64 and gnu+musl for arm64; we use
# the musl (statically linked) builds for both arches.
ARG RIPGREP_SHA256_AMD64=33e15bcf1624b25cdd2a55813a47a2f95dbe126268203e76aa6a585d1e7b149c
ARG RIPGREP_SHA256_ARM64=800b1e7206afe799dfb5a6901f23147cfaabe0e52210538100f61e86e1740915

ARG FD_VERSION=10.5.0
ARG FD_SHA256_AMD64=a1259cd129636efbc3fef123525c1b49e88fe5088c012630983c310e52fdfa95
ARG FD_SHA256_ARM64=c0ee43802e3313a317c5af2f4eabd6ba13eeedd595af9775f05e18a13ac4f52c

# gofumpt does not publish a checksums file -> checksums pinned here
ARG GOFUMPT_SHA256_AMD64=3bc4fcad497439918f5f56d4987de4709a99da0a355e6bec13a7278532b01d38
ARG GOFUMPT_SHA256_ARM64=4322f081241ab0618356aa5921a8c110e4c7acfc17175cc15dc57b2f4a4b652d

# Go tools installed via `go install` (sum.golang.org verification)
ARG MOCKERY_VERSION=v3.8.1
ARG GO_LICENSES_VERSION=v2.0.1
ARG GOIMPORTS_VERSION=v0.52.0
# nilaway publishes no release tags; the version is a module pseudo-version
# resolved through the Go module proxy (sum.golang.org).
ARG NILAWAY_VERSION=v0.0.0-20260918162853-acb8859b9031

# markdownlint-cli (npm, native node from apt)
ARG MARKDOWNLINT_CLI_VERSION=0.49.1

# prettier (npm, native node from apt) — formats JS/TS/JSON/YAML/Markdown
ARG PRETTIER_VERSION=3.9.10

# ============================================================================
# build stage: download and verify Go + tools
# ============================================================================
FROM ubuntu:${UBUNTU_VERSION} AS build
ARG TARGETARCH
ARG GOLANG_VERSION
ARG GO_SHA256_AMD64
ARG GO_SHA256_ARM64
ARG GOLANGCI_LINT_VERSION
ARG GOARCHLINT_VERSION
ARG GREMLINS_VERSION
ARG GOFUMPT_VERSION
ARG GOFUMPT_SHA256_AMD64
ARG GOFUMPT_SHA256_ARM64
ARG GOSEC_VERSION
ARG GOVULNCHECK_VERSION
ARG YQ_VERSION
ARG YQ_SHA256_AMD64
ARG YQ_SHA256_ARM64
ARG RIPGREP_VERSION
ARG RIPGREP_SHA256_AMD64
ARG RIPGREP_SHA256_ARM64
ARG FD_VERSION
ARG FD_SHA256_AMD64
ARG FD_SHA256_ARM64
ARG MOCKERY_VERSION
ARG GO_LICENSES_VERSION
ARG GOIMPORTS_VERSION
ARG NILAWAY_VERSION

# Minimal set to download over HTTPS (tarball + GitHub assets)
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl tar gzip; \
    rm -rf /var/lib/apt/lists/*

ENV GOROOT=/usr/local/go
ENV PATH="/usr/local/go/bin:${PATH}"
WORKDIR /build

# --- Go 1.27.2 (official tarball, verified against official sha256) ---------
# Verification mirrors the official golang image: echo "$sha256 *<file>" | sha256sum -c -
RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) sha256="${GO_SHA256_AMD64}" ;; \
      arm64) sha256="${GO_SHA256_ARM64}" ;; \
      *) echo "Unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="go${GOLANG_VERSION}.linux-${TARGETARCH}.tar.gz"; \
    curl -fsSL -o "/tmp/${asset}" "https://dl.google.com/go/${asset}"; \
    (cd /tmp && echo "${sha256} *${asset}" | sha256sum -c -); \
    tar -C /usr/local -xzf "/tmp/${asset}"; \
    rm "/tmp/${asset}"; \
    go version

# --- golangci-lint (checksums: golangci-lint-<v>-checksums.txt) -------------
RUN set -eux; \
    asset="golangci-lint-${GOLANGCI_LINT_VERSION}-linux-${TARGETARCH}.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/golangci/golangci-lint/releases/download/v${GOLANGCI_LINT_VERSION}/${asset}"; \
    curl -fsSL -o golangci-lint-checksums.txt \
      "https://github.com/golangci/golangci-lint/releases/download/v${GOLANGCI_LINT_VERSION}/golangci-lint-${GOLANGCI_LINT_VERSION}-checksums.txt"; \
    sha256sum -c --ignore-missing golangci-lint-checksums.txt; \
    tar -xzf "${asset}"; \
    install -m 0755 "golangci-lint-${GOLANGCI_LINT_VERSION}-linux-${TARGETARCH}/golangci-lint" /usr/local/bin/golangci-lint; \
    rm -rf "${asset}" golangci-lint-checksums.txt "golangci-lint-${GOLANGCI_LINT_VERSION}-linux-${TARGETARCH}"

# --- go-arch-lint (checksums: checksums.txt) --------------------------------
RUN set -eux; \
    asset="go-arch-lint_${GOARCHLINT_VERSION}_linux_${TARGETARCH}.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/fe3dback/go-arch-lint/releases/download/v${GOARCHLINT_VERSION}/${asset}"; \
    curl -fsSL -o goarchlint-checksums.txt \
      "https://github.com/fe3dback/go-arch-lint/releases/download/v${GOARCHLINT_VERSION}/checksums.txt"; \
    sha256sum -c --ignore-missing goarchlint-checksums.txt; \
    tar -xzf "${asset}"; \
    install -m 0755 go-arch-lint /usr/local/bin/go-arch-lint; \
    rm -rf "${asset}" goarchlint-checksums.txt go-arch-lint

# --- gremlins (checksums: checksums.txt) ------------------------------------
RUN set -eux; \
    asset="gremlins_${GREMLINS_VERSION}_linux_${TARGETARCH}.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/go-gremlins/gremlins/releases/download/v${GREMLINS_VERSION}/${asset}"; \
    curl -fsSL -o gremlins-checksums.txt \
      "https://github.com/go-gremlins/gremlins/releases/download/v${GREMLINS_VERSION}/checksums.txt"; \
    sha256sum -c --ignore-missing gremlins-checksums.txt; \
    tar -xzf "${asset}"; \
    install -m 0755 gremlins /usr/local/bin/gremlins; \
    rm -rf "${asset}" gremlins-checksums.txt gremlins

# --- gofumpt (no upstream checksums file -> pinned checksum) -----------------
RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) expected="${GOFUMPT_SHA256_AMD64}" ;; \
      arm64) expected="${GOFUMPT_SHA256_ARM64}" ;; \
      *) echo "Unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="gofumpt_v${GOFUMPT_VERSION}_linux_${TARGETARCH}"; \
    curl -fsSL -o "/tmp/${asset}" \
      "https://github.com/mvdan/gofumpt/releases/download/v${GOFUMPT_VERSION}/${asset}"; \
    echo "${expected}  /tmp/${asset}" | sha256sum -c -; \
    install -m 0755 "/tmp/${asset}" /usr/local/bin/gofumpt; \
    rm -f "/tmp/${asset}"

# --- gosec (checksums: gosec_<v>_checksums.txt) -----------------------------
RUN set -eux; \
    asset="gosec_${GOSEC_VERSION}_linux_${TARGETARCH}.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/securego/gosec/releases/download/v${GOSEC_VERSION}/${asset}"; \
    curl -fsSL -o gosec-checksums.txt \
      "https://github.com/securego/gosec/releases/download/v${GOSEC_VERSION}/gosec_${GOSEC_VERSION}_checksums.txt"; \
    sha256sum -c --ignore-missing gosec-checksums.txt; \
    tar -xzf "${asset}"; \
    install -m 0755 gosec /usr/local/bin/gosec; \
    rm -rf "${asset}" gosec-checksums.txt gosec

# --- govulncheck (no binary releases -> go install, sum.golang.org) ---------
RUN set -eux; \
    GOBIN=/usr/local/bin GOPATH=/tmp/gopath go install "golang.org/x/vuln/cmd/govulncheck@${GOVULNCHECK_VERSION}"

# --- yq (mikefarah/yq) (no sha256sum-compatible checksums file -> pinned) --
RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) expected="${YQ_SHA256_AMD64}" ;; \
      arm64) expected="${YQ_SHA256_ARM64}" ;; \
      *) echo "Unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="yq_linux_${TARGETARCH}.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/mikefarah/yq/releases/download/v${YQ_VERSION}/${asset}"; \
    echo "${expected}  ${asset}" | sha256sum -c -; \
    tar -xzf "${asset}"; \
    # the yq tarball contains a single arch-suffixed binary (yq_linux_<arch>)
    install -m 0755 "yq_linux_${TARGETARCH}" /usr/local/bin/yq; \
    rm -rf "${asset}" "yq_linux_${TARGETARCH}" yq.1 install-man-page.sh

# --- ripgrep (BurntSushi/ripgrep) (per-asset .sha256 only -> pinned) -------
RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) arch="x86_64"; expected="${RIPGREP_SHA256_AMD64}" ;; \
      arm64) arch="aarch64"; expected="${RIPGREP_SHA256_ARM64}" ;; \
      *) echo "Unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="ripgrep-${RIPGREP_VERSION}-${arch}-unknown-linux-musl.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/BurntSushi/ripgrep/releases/download/${RIPGREP_VERSION}/${asset}"; \
    echo "${expected}  ${asset}" | sha256sum -c -; \
    tar -xzf "${asset}"; \
    install -m 0755 "ripgrep-${RIPGREP_VERSION}-${arch}-unknown-linux-musl/rg" /usr/local/bin/rg; \
    rm -rf "${asset}" "ripgrep-${RIPGREP_VERSION}-${arch}-unknown-linux-musl"

# --- fd (sharkdp/fd) (per-asset .sha256 only -> pinned) ---------------------
RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) arch="x86_64"; expected="${FD_SHA256_AMD64}" ;; \
      arm64) arch="aarch64"; expected="${FD_SHA256_ARM64}" ;; \
      *) echo "Unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="fd-v${FD_VERSION}-${arch}-unknown-linux-gnu.tar.gz"; \
    curl -fsSL -o "${asset}" \
      "https://github.com/sharkdp/fd/releases/download/v${FD_VERSION}/${asset}"; \
    echo "${expected}  ${asset}" | sha256sum -c -; \
    tar -xzf "${asset}"; \
    install -m 0755 "fd-v${FD_VERSION}-${arch}-unknown-linux-gnu/fd" /usr/local/bin/fd; \
    rm -rf "${asset}" "fd-v${FD_VERSION}-${arch}-unknown-linux-gnu"

# --- mockery (vektra/mockery) (no binary releases -> go install) ------------
RUN set -eux; \
    GOBIN=/usr/local/bin GOPATH=/tmp/gopath go install "github.com/vektra/mockery/v3@${MOCKERY_VERSION}"

# --- go-licenses (google/go-licenses) (no binary releases -> go install) ----
RUN set -eux; \
    GOBIN=/usr/local/bin GOPATH=/tmp/gopath go install "github.com/google/go-licenses/v2@${GO_LICENSES_VERSION}"

# --- goimports (golang.org/x/tools) (no binary releases -> go install) ------
RUN set -eux; \
    GOBIN=/usr/local/bin GOPATH=/tmp/gopath go install "golang.org/x/tools/cmd/goimports@${GOIMPORTS_VERSION}"

# --- nilaway (go.uber.org/nilaway) (no binary releases/tags -> go install) --
# /tmp/gopath (shared module cache for all go-install tools) is removed here,
# after the last go-install step, so the module cache is reused across layers.
RUN set -eux; \
    GOBIN=/usr/local/bin GOPATH=/tmp/gopath go install "go.uber.org/nilaway/cmd/nilaway@${NILAWAY_VERSION}"; \
    rm -rf /tmp/gopath

# ============================================================================
# final stage: ubuntu:26.04 + golang:1.27.2 environment + tools
# ============================================================================
FROM ubuntu:${UBUNTU_VERSION} AS final
ARG GOLANG_VERSION
ARG MARKDOWNLINT_CLI_VERSION
ARG PRETTIER_VERSION

# System packages, as in the official golang:1.27.2 image:
#   * cgo dependencies of the golang image's final stage: g++, gcc, libc6-dev, make,
#     pkg-config (+ binutils-gold on arm64);
#   * the buildpack-deps:bookworm-scm base set (git, ca-certificates, curl, wget,
#     openssh-client, etc.);
#   * nodejs+npm for markdownlint-cli (ubuntu 26.04 ships node 22.22.1 >= 22).
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        wget \
        git \
        openssh-client \
        gcc \
        g++ \
        make \
        libc6-dev \
        pkg-config \
        nodejs \
        npm \
        jq \
    ; \
    dpkgArch="$(dpkg --print-architecture)"; \
    if [ "$dpkgArch" = 'arm64' ]; then \
        apt-get install -y --no-install-recommends binutils-gold; \
    fi; \
    rm -rf /var/lib/apt/lists/*

# Go runtime (checksum-verified in the build stage)
COPY --from=build /usr/local/go /usr/local/go

# Go tools (checksum-verified in the build stage)
COPY --from=build /usr/local/bin/golangci-lint /usr/local/bin/golangci-lint
COPY --from=build /usr/local/bin/go-arch-lint /usr/local/bin/go-arch-lint
COPY --from=build /usr/local/bin/gremlins /usr/local/bin/gremlins
COPY --from=build /usr/local/bin/gofumpt /usr/local/bin/gofumpt
COPY --from=build /usr/local/bin/gosec /usr/local/bin/gosec
COPY --from=build /usr/local/bin/govulncheck /usr/local/bin/govulncheck
COPY --from=build /usr/local/bin/yq /usr/local/bin/yq
COPY --from=build /usr/local/bin/rg /usr/local/bin/rg
COPY --from=build /usr/local/bin/fd /usr/local/bin/fd
COPY --from=build /usr/local/bin/mockery /usr/local/bin/mockery
COPY --from=build /usr/local/bin/go-licenses /usr/local/bin/go-licenses
COPY --from=build /usr/local/bin/goimports /usr/local/bin/goimports
COPY --from=build /usr/local/bin/nilaway /usr/local/bin/nilaway

# markdownlint-cli + prettier — native npm (apt nodejs). prettier formats
# JS/TS/JSON/YAML/Markdown. npm cache is cleaned in the same layer so it does
# not bloat the final image.
RUN npm install -g \
        "markdownlint-cli@${MARKDOWNLINT_CLI_VERSION}" \
        "prettier@${PRETTIER_VERSION}" \
    && npm cache clean --force

# Environment identical to the official golang:1.27.2
ENV GOLANG_VERSION=${GOLANG_VERSION}
ENV GOTOOLCHAIN=local
ENV GOROOT=/usr/local/go
ENV GOPATH=/go
ENV PATH="$GOPATH/bin:/usr/local/go/bin:${PATH}"

# Go caches in dedicated volumes (see VOLUME below): reused across runs
ENV GOCACHE=/go/cache
ENV GOMODCACHE=/go/pkg/mod

# Standard GOPATH layout (as in the official image)
RUN mkdir -p "$GOPATH/src" "$GOPATH/bin" && chmod -R 1777 "$GOPATH"

# Smoke test: all tools are installed and run
RUN set -eux; \
    go version; \
    go env GOROOT GOPATH GOTOOLCHAIN; \
    golangci-lint --version; \
    gofumpt -version; \
    gosec -version; \
    govulncheck -version; \
    go-arch-lint version; \
    gremlins --version; \
    yq --version; \
    rg --version; \
    fd --version; \
    mockery version; \
    go-licenses help >/dev/null; \
    echo 'package main' | goimports >/dev/null; \
    nilaway -h >/dev/null; \
    jq --version; \
    node --version; \
    markdownlint --version; \
    prettier --version

# Container runtime user (UID/GID 1000).
# ubuntu:26.04 already ships an account with uid/gid 1000 (the ubuntu user),
# so creation is idempotent: we create `user` only if it does not exist and
# always hand /go over to uid 1000. USER is numeric so the image does not
# depend on the name of the existing account.
RUN set -eux; \
    if ! getent group 1000 >/dev/null; then \
        groupadd --gid 1000 user; \
    fi; \
    if ! getent passwd 1000 >/dev/null; then \
        useradd --uid 1000 --gid 1000 --create-home user; \
    fi; \
    chown -R 1000:1000 "$GOPATH"

USER 1000:1000

# Dedicated volumes: temp files and Go caches (GOCACHE, GOMODCACHE),
# so they can be reused across container runs.
VOLUME /tmp
VOLUME /go/cache
VOLUME /go/pkg/mod

WORKDIR /go
CMD ["go", "version"]
