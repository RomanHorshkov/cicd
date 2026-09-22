# cicd

Reusable GitHub Actions workflows for the rh C repositories. Written once here, called from
every repository with a few-line caller. A fix lands here and every repository picks it up on
its next run.

## Versioning

- `v1`, `v2`, ... are moving major tags. Callers pin the major: `@v1`.
- A change that keeps every caller working moves `v1`. A change that needs callers to adapt
  is `v2`; repositories migrate one at a time.
- Every published release records which cicd commit built it (`Built by cicd: <ref> @ <sha>`
  in the release notes), so a release is always traceable to the exact workflow.

## Workflows

### `c-release.yml`

Release pipeline for a C library: verify the tag matches `VERSION`, resolve `deps.txt` when
present, build the `.deb` through the repository's own `utils/build_deb.sh`, assemble the
tarball + deb + `SHA256SUMS` bundle, attest it (optional), upload it, and publish the GitHub
Release on a tag push. A `workflow_dispatch` run skips the tag check and publishes nothing.

Caller (the whole `release.yml` of a library):

```yaml
name: Release
on:
  push:
    tags: ["v*.*.*"]
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: false
jobs:
  quality-gate:
    name: Quality gate
    uses: ./.github/workflows/quality.yml
  release:
    name: Release
    needs: quality-gate
    permissions:
      contents: write
      id-token: write
      attestations: write
    uses: RomanHorshkov/cicd/.github/workflows/c-release.yml@v1
    with:
      lib-name: spscring
      title: SPSCring
    secrets: inherit
```

Inputs: `lib-name` (required), `title` (default: lib-name), `attest` (default true; set
false on a private repository), `apt-packages` (default `build-essential pkg-config fakeroot`).

Contract the calling repository must honour: `VERSION` in strict semver, `utils/build_deb.sh`
writing `build/debs/*.deb` after building the release profile, `app/<lib-name>.h`,
`build/release/lib<lib-name>.a` and `lib<lib-name>.so.<version>`, and optionally `deps.txt`
with `utils/deps.sh`.
