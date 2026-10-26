# Bluesky fork releases

Source: https://github.com/bluesky-social/vlpds
Upstream: https://github.com/jazware/vlpds

The server is Rust (currently Rust 1.99.0). Go 1.27.3 is pinned for the
sync checker, HA audit/fault proxy helpers, and benchmark MinIO builder.
A Go upgrade does not change the server binary's compiler. CI tests each Go
module with the pinned toolchain, alongside the existing Rust and UI checks.
The MinIO builder still selects MinIO and mc with `@latest`; its rebuild is
not a source-identical compiler-only rebuild.

## Publish

1. Commit the release changes and push to this fork's `main`.
2. Wait for every CI job on that exact commit to pass.
3. Create an annotated fork release tag, such as `v1.2.3-bsky.1`, on
   the tested commit and push that tag to this fork. Do not reuse upstream tags.
4. Wait for the tag's Image workflow to finish. It builds native linux/amd64
   and linux/arm64 images with the `dist` profile, publishes to
   `ghcr.io/bluesky-social/vlpds`, and attaches provenance and SBOMs.
5. Record the manifest digest and source commit in the GitHub Release.
   Image tags omit the leading `v` (for example `1.2.3-bsky.1`).

`just image-push` also defaults to the Bluesky GHCR namespace. Its build
context contains committed files only. It publishes by default; set
`PUSH=0` for a local image. It authenticates to GHCR, not ECR.

## Deployment

Publishing an image does not deploy it. Consult the deployment repository's
instructions, validate the target image, and update its immutable digest through
the owning GitOps configuration. The source fork does not carry deployment
credentials or perform automatic rollout.

### Upgrading the POP1 pilot from 1.1 to 1.2

Keep the existing bucket and prefix. An existing level-1 cluster stays at level 1
after the image upgrade; a fresh prefix starts at the new build's maximum level.
Do not run `cluster finalize --level 2` as part of this upgrade. Level 2 enables
federated sign-in storage and cannot be lowered to level 1. Once finalized, the
1.1 binary cannot serve that storage.

Before rollout, run the two-build integration scenarios against the exact
deployed source revision, not the harness's default previous-release selection:

```bash
VLPDS_PREV_REV=058abbfabfffaec93d8eb50582fa1ae3de81049c \
  bench/ha/upgrade.sh --minio upgrade-rolling-l1 upgrade-rollback
```

This uses disposable MinIO storage and requires Rust, Go, Python, Docker and the
local `vlpds-minio:local` fixture image. `upgrade-rolling-l1` verifies the real
build transition and level-2 finalization on disposable storage;
`upgrade-rollback` verifies return to the old binary without finalizing, using
the test-level build. Feature-gating unit tests do not replace these checks.

For the deployment candidate, validate the release image at the pilot's existing
3072 MiB memory limit and 2560 MiB memory budget, mirror its amd64 manifest to
ECR, and pin the verified digest in `apps/vlpds-poc/values.yaml` in `deploy`.
Retain the current image digest for rollback. The pilot uses a single replica
with `Recreate`, so applying the image change interrupts it briefly. Check the
live cluster feature level before and after rollout, then account reads, writes
on a designated test account, and relay ingestion. Mutating checks and rollout
require their normal approvals.

Keep Spaces and federated sign-in disabled during this upgrade. Before any later
level-2 finalization or provider enablement, review enrollment routing: 1.2 adds
`/oauth/federated/signup`, which is separate from the pilot's existing signup
path block.
