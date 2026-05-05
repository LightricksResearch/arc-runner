# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Lightricks fork of [actions/runner](https://github.com/actions/runner) — the GitHub Actions self-hosted runner. The fork's purpose is to package the upstream runner binary into a custom Docker image (`gcr.io/ltx-research/arc-runner`) suitable for ARC (Actions Runner Controller) on GKE, with CUDA / GPU tooling preinstalled. Upstream sources under `src/` are kept close to vanilla so they can be re-synced from `actions/runner` when the runner version is bumped.

## Layout

| Path | Purpose |
|---|---|
| `src/` | Upstream .NET runner source (Runner.Listener, Runner.Worker, etc.). Built with `./dev.sh`. |
| `src/runnerversion` | Upstream metadata file. In this fork it is read by `publish-image.yml` only to **tag** the published image and stamp the OCI description label — it is **not** passed as a build-arg, so it does not control which runner binary is baked in. |
| `images/Dockerfile` | Custom image: `nvidia/cuda:12.2.0-base-ubuntu22.04` base + Python 3.10 + Docker CLI + the runner binary downloaded from `actions/runner` releases. The `RUNNER_VERSION` ARG (default `2.329.0`) is the actual source of truth for which runner binary is downloaded. |
| `.github/workflows/build.yml` | CI for the upstream runner (builds + L0 tests). Skips on `**.md` and `images/**` paths. |
| `.github/workflows/publish-image.yml` | Manual (`workflow_dispatch`) workflow that builds `images/Dockerfile` and pushes to GCR. Authenticates to GCP via Workload Identity Federation (`gh-pool` / `gh-provider`). |
| `releaseVersion`, `releaseNote.md` | Upstream release metadata — leave alone unless syncing from `actions/runner`. |

## Common Commands

```bash
# Build the .NET runner locally (from src/)
cd src && ./dev.sh layout Release linux-x64

# Run upstream L0 tests
cd src && ./dev.sh test

# Build the custom runner image locally
docker build -f images/Dockerfile -t arc-runner:dev images/

# Bump the runner to a new upstream version
# 1. update RUNNER_VERSION ARG in images/Dockerfile (controls the binary)
# 2. update src/runnerversion to the same value (controls the image tag)
# 3. trigger "Publish Runner Image" on GitHub
#    (or pass runnerVersion as workflow input to override the tag — note: the
#    input currently does NOT override the build-arg, so the binary still comes
#    from images/Dockerfile)
```

## Conventions

- `src/` mirrors upstream — avoid editing it. Customizations go in `images/Dockerfile` or workflows.
- When bumping the runner version, update **both** the `RUNNER_VERSION` ARG in `images/Dockerfile` and `src/runnerversion` in the same commit so the binary in the image matches the tag stamped on it. They drift today (`0.0.1` vs `2.329.0`) — fixing that drift is fine, just do it in one PR.
- `images/**` and `**.md` changes deliberately skip the .NET CI matrix. If you add image build verification, do it in `publish-image.yml` (or a new workflow), not by removing the path filter.
- The `publish-image.yml` `runnerVersion` workflow input only retags the image; it does **not** flow through to the Dockerfile build-arg. To actually change the bundled runner, edit the ARG.

## GCP / GKE Context

- GCP project: `ltx-research`
- Image registry: `gcr.io/ltx-research/arc-runner`
- Workload Identity Federation: `projects/474377433429/locations/global/workloadIdentityPools/gh-pool/providers/gh-provider`
- CI service account: `github-actions@ltx-research.iam.gserviceaccount.com`
- Runners deployed via ARC; Helm chart lives in the `helm` repo under `runner/`.

## Working Agreements for Claude

- **Untracked files belong to the contributor.** Treat any pre-existing untracked file in the working tree as the contributor's personal scratch (local test values, throwaway manifests, ad-hoc scripts, etc.). Do **not** `git add` them, do **not** list them in committed docs, and do **not** assume they should be gitignored. Only stage untracked files that Claude Code created itself for the task at hand, and only if the user asked for them to be committed.
