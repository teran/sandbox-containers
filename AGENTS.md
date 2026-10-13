# AGENTS.md — Guide for AI agents working in this repository

This file gives automated agents and contributors the rules and workflow for
making changes to `teran/sandbox-containers` safely and consistently.

## Project overview

`teran/sandbox-containers` builds and publishes **sandbox container images**
for use with `mcp-sandbox`. Each image lives in its own **subdirectory**
(`golang/`, future `python/`, `node/`, …) and provides a self-contained,
unprivileged environment for one language ecosystem.

What you will most often change:

- `golang/Dockerfile` — the Go sandbox image (currently the only image).
- `golang/README.md` — its documentation.
- `.github/workflows/build.yml` — CI that lints, builds, and (on `master`)
  pushes images to GHCR.
- `README.md`, `SPEC.md`, `AGENTS.md` — repository-level docs.

## Hard rules

1. **Never hardcode a runtime version in two places.**
   The CI workflow **extracts** the Go version from the `Dockerfile`
   (`grep -oP 'GOLANG_VERSION=\K[0-9.]+' golang/Dockerfile`). When you bump a
   version, update only the `ARG` in the `Dockerfile` (and the matching
   `sha256` pins). Do **not** edit the version anywhere in the workflow —
   it must stay derived. If you hardcode it, CI will tag images with a stale
   version.

2. **Keep docs in sync.** `README.md`, `SPEC.md`, `golang/README.md`, and the
   workflow must not contradict each other (versions, tag scheme, image name,
   directory layout). If you change one, update the others.

3. **New images are new subdirectories.** Do not cram a second toolchain into
   `golang/`. Add `python/`, `node/`, etc. as separate directories, following
   the `golang/` reference (multi-stage, multi-arch, checksum-pinned,
   non-root).

4. **Pin and verify.** Versions and `sha256` checksums are pinned as `ARG`s at
   the top of the `Dockerfile`. Verify every downloaded artifact
   (`sha256sum -c` against upstream files, or pinned values where upstream
   publishes none). Never introduce unverified downloads.

5. **Images run as non-root.** The final stage must end with
   `USER <uid>:<gid>` (the Go image uses `1000:1000`), with writable volumes
   for caches.

6. **CI must stay green.** `hadolint` runs with `failure-threshold: error`;
   multi-platform builds (`linux/amd64`, `linux/arm64`) must succeed on every
   PR and push.

## Workflow: adding a new sandbox image

Checklist for adding e.g. `python/`:

- [ ] Create `python/Dockerfile` — multi-stage, multi-arch (`TARGETARCH`),
      versions + `sha256` pinned as `ARG`s, non-root `USER`.
- [ ] Add `python/.dockerignore` (keep the build context lean).
- [ ] Add `python/README.md` (mirror the structure of `golang/README.md`).
- [ ] Extend `.github/workflows/build.yml`:
      - add `'python/**'` to both `paths:` filters;
      - set `IMAGE: ghcr.io/teran/sandbox-containers/python`;
      - point `hadolint` and `docker/build-push-action` at
        `python/Dockerfile` and `context: ./python`;
      - adapt the version-extraction `grep` to the new `ARG`
        (e.g. `PYTHON_VERSION=`).
- [ ] Add `python` to the image table in `SPEC.md` and the root `README.md`.
- [ ] Open a PR and confirm `hadolint` + multi-platform build pass on the PR;
      then verify the push produces the 3 tags on `master`.

## Commit rules (Conventional Commits)

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>
```

Examples:

- `feat(golang): bump Go to 1.27.2`
- `feat: add python sandbox image`
- `docs: rewrite root README`
- `ci: pin actions to major versions`
- `chore(golang): refresh gofumpt sha256 pins`

Useful types: `feat`, `fix`, `docs`, `ci`, `chore`, `refactor`, `test`.
A `feat` that adds a whole image touches `SPEC.md`, the root `README.md`, the
workflow, and the new subdirectory — mention that in the body.

## Deliverable

Do not push directly to `master`. Work through PRs; CI handles lint, build and
push on merge.
