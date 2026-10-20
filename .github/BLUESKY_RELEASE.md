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
3. Create an annotated fork release tag, beginning with `v1.1.0-bsky.1`, on
   the tested commit and push that tag to this fork. Do not reuse upstream tags.
4. Wait for the tag's Image workflow to finish. It builds native linux/amd64
   and linux/arm64 images with the `dist` profile, publishes to
   `ghcr.io/bluesky-social/vlpds`, and attaches provenance and SBOMs.
5. Record the manifest digest and source commit in the GitHub Release.
   Image tags omit the leading `v` (for example `1.1.0-bsky.1`).

`just image-push` also defaults to the Bluesky GHCR namespace. Its build
context contains committed files only. It publishes by default; set
`PUSH=0` for a local image. It authenticates to GHCR, not ECR.

## Deployment

Publishing an image does not deploy it. Consult the deployment repository's
instructions, validate the target image, and update its immutable digest through
the owning GitOps configuration. The source fork does not carry deployment
credentials or perform automatic rollout.
