# syntax=docker/dockerfile:1
#
# Node.js 26.13.0 (LTS "Lithium") sandbox on a VANILLA ubuntu:26.04
# (Resolute Raccoon) base. Node is installed from the official tarball
# https://nodejs.org/dist/ into /usr/local (as in the official node:26.13.0
# image), npm ships inside, and the linter/CLI toolset matches the golang
# sandbox (yq, ripgrep, fd) plus npm-based tools (pnpm, typescript, eslint,
# prettier, markdownlint-cli).
#
# Principles:
#   * Versions and checksums are pinned as ARG at the top.
#   * The Node tarball is fetched from
#     https://nodejs.org/dist/v26.13.0/node-v26.13.0-linux-<arch>.tar.gz
#     (arch = x64 for amd64, arm64 for arm64) and verified against the official
#     sha256.
#   * Tools distributed as binary releases (yq/ripgrep/fd) are downloaded from
#     GitHub and verified by pinned checksum before installation.
#   * npm tools are installed via `npm install -g` (npm integrity hashes).
#   * Architecture is TARGETARCH (amd64/arm64), matching asset names.
#
# IMPORTANT: official Node.js >= 25 Linux binaries are linked against libatomic,
# so the final stage MUST install libatomic1 (via apt). Node is NOT executed in
# the build stage (which lacks libatomic1); its version is smoke-tested in the
# final stage where libatomic1 is present.

# ============================================================================
# Versions (deterministic)
# ============================================================================
ARG UBUNTU_VERSION=26.04

ARG NODE_VERSION=26.13.0
# Official sha256 of the Node.js tarballs (https://nodejs.org/dist/v26.13.0/SHASUMS256.txt)
ARG NODE_SHA256_AMD64=5848553868b402d3fabb5b09270649977780a390f832155721025a08b5104455
ARG NODE_SHA256_ARM64=439e9c438acdf6a6ffe47418da709153d66c4bed812e30a473faacb107dd2791

# npm tools (global install via npm, integrity verified by npm)
ARG PNPM_VERSION=12.16.1
ARG TYPESCRIPT_VERSION=7.0.2
ARG ESLINT_VERSION=10.13.0
ARG PRETTIER_VERSION=3.9.10
ARG MARKDOWNLINT_CLI_VERSION=0.50.0

# yq / ripgrep / fd do not publish a sha256sum-compatible checksums.txt
# (yq's `checksums` file uses a non-standard multi-hash format; rg/fd ship
# per-asset .sha256 files) -> checksums pinned here as ARGs (same versions and
# checksums as in golang/Dockerfile).
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

# ============================================================================
# build stage: download and verify Node.js + CLI tools
# ============================================================================
FROM ubuntu:${UBUNTU_VERSION} AS build
ARG TARGETARCH
ARG NODE_VERSION
ARG NODE_SHA256_AMD64
ARG NODE_SHA256_ARM64
ARG YQ_VERSION
ARG YQ_SHA256_AMD64
ARG YQ_SHA256_ARM64
ARG RIPGREP_VERSION
ARG RIPGREP_SHA256_AMD64
ARG RIPGREP_SHA256_ARM64
ARG FD_VERSION
ARG FD_SHA256_AMD64
ARG FD_SHA256_ARM64

# Minimal set to download over HTTPS (Node tarball + GitHub assets)
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends ca-certificates curl tar gzip; \
    rm -rf /var/lib/apt/lists/*

WORKDIR /build

# --- Node.js 26.13.0 (official tarball, verified against official sha256) ---
# Extracted into /usr/local exactly like the official node:26.13.0 image
# (`--strip-components=1`). Not executed here: Node >= 25 needs libatomic1,
# which is only installed in the final stage.
#
# NOTE: Node's official Linux asset for amd64 is named `linux-x64` (arm64 is
# `linux-arm64`), so TARGETARCH maps to the asset arch name.
RUN set -eux; \
    case "${TARGETARCH}" in \
      amd64) arch="x64";    sha256="${NODE_SHA256_AMD64}" ;; \
      arm64) arch="arm64";  sha256="${NODE_SHA256_ARM64}" ;; \
      *) echo "Unsupported arch: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    asset="node-v${NODE_VERSION}-linux-${arch}.tar.gz"; \
    curl -fsSL -o "/tmp/${asset}" "https://nodejs.org/dist/v${NODE_VERSION}/${asset}"; \
    (cd /tmp && echo "${sha256} *${asset}" | sha256sum -c -); \
    tar -C /usr/local --strip-components=1 -xzf "/tmp/${asset}"; \
    rm "/tmp/${asset}"

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

# ============================================================================
# final stage: ubuntu:26.04 + node:26.13.0 environment + tools
# ============================================================================
FROM ubuntu:${UBUNTU_VERSION} AS final
ARG NODE_VERSION
ARG PNPM_VERSION
ARG TYPESCRIPT_VERSION
ARG ESLINT_VERSION
ARG PRETTIER_VERSION
ARG MARKDOWNLINT_CLI_VERSION

# System packages. The official node:26.13.0 image is built on
# buildpack-deps:bookworm-scm (git, ca-certificates, curl, wget,
# openssh-client, etc.). Additionally:
#   * libatomic1 — required by Node.js >= 25 Linux binaries (see header);
#   * jq (universe) + shellcheck — sandbox toolset (as in golang/);
#   * binutils-gold on arm64 (as in golang/).
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        wget \
        git \
        openssh-client \
        jq \
        libatomic1 \
        shellcheck \
    ; \
    dpkgArch="$(dpkg --print-architecture)"; \
    if [ "$dpkgArch" = 'arm64' ]; then \
        apt-get install -y --no-install-recommends binutils-gold; \
    fi; \
    rm -rf /var/lib/apt/lists/*

# Node.js runtime + CLI tools (checksum-verified in the build stage).
# Node ships in /usr/local/{bin,include,lib,share}; yq/rg/fd go to /usr/local/bin.
COPY --from=build /usr/local/bin /usr/local/bin
COPY --from=build /usr/local/include /usr/local/include
COPY --from=build /usr/local/lib /usr/local/lib
COPY --from=build /usr/local/share /usr/local/share

# Environment identical to the official node:26.13.0 image. node/npm live in
# /usr/local/bin (already in the base PATH); pnpm global shims live in PNPM_HOME.
ENV NODE_VERSION=${NODE_VERSION}
ENV PNPM_HOME=/pnpm
ENV PATH="${PNPM_HOME}:${PATH}"

# npm cache + pnpm global home live in dedicated volumes (see VOLUME below),
# so they can be reused across runs.
ENV NPM_CONFIG_CACHE=/cache/npm
RUN mkdir -p "$PNPM_HOME" "/cache/npm"

# npm tools — global install (npm integrity verification); npm cache is cleaned
# in the same layer so it does not bloat the final image.
RUN npm install -g \
        "pnpm@${PNPM_VERSION}" \
        "typescript@${TYPESCRIPT_VERSION}" \
        "eslint@${ESLINT_VERSION}" \
        "prettier@${PRETTIER_VERSION}" \
        "markdownlint-cli@${MARKDOWNLINT_CLI_VERSION}" \
    && npm cache clean --force

# Smoke test: all tools are installed and run
RUN set -eux; \
    node --version; \
    npm --version; \
    pnpm --version; \
    tsc --version; \
    eslint --version; \
    prettier --version; \
    markdownlint --version; \
    jq --version; \
    yq --version; \
    rg --version; \
    fd --version; \
    shellcheck --version

# Container runtime user (UID/GID 1000).
# ubuntu:26.04 already ships an account with uid/gid 1000 (the ubuntu user),
# so creation is idempotent: we create `user` only if it does not exist and
# always hand the caches over to uid 1000. USER is numeric so the image does
# not depend on the name of the existing account.
RUN set -eux; \
    if ! getent group 1000 >/dev/null; then \
        groupadd --gid 1000 user; \
    fi; \
    if ! getent passwd 1000 >/dev/null; then \
        useradd --uid 1000 --gid 1000 --create-home user; \
    fi; \
    mkdir -p /home/user "$PNPM_HOME" "/cache"; \
    chown -R 1000:1000 "$PNPM_HOME" "/cache" "/home/user"

USER 1000:1000

# Dedicated volumes: temp files, npm cache + pnpm global home, so they can be
# reused across container runs.
VOLUME /tmp
VOLUME /cache
VOLUME /pnpm

WORKDIR /home/user
CMD ["node", "--version"]
