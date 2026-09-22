# cicd

Reusable GitHub Actions workflows for the rh C repositories. Callers pin a major tag (`@v1`); compatible fixes move the tag, breaking changes get `v2`.

## c-release.yml

Tag check against `VERSION`, `deps.txt` resolve (if present), `utils/build_deb.sh`, tarball + deb + `SHA256SUMS`, attestation, GitHub Release. `workflow_dispatch` publishes nothing.

Caller:

```yaml
jobs:
  quality-gate:
    uses: ./.github/workflows/quality.yml
  release:
    needs: quality-gate
    permissions: { contents: write, id-token: write, attestations: write }
    uses: RomanHorshkov/cicd/.github/workflows/c-release.yml@v1
    with: { lib-name: spscring, title: SPSCring }
    secrets:
      SIBLING_REPOS_PAT: ${{ secrets.SIBLING_REPOS_PAT }}
```

Inputs: `lib-name`, `title`, `attest` (default true), `apt-packages`.

Repo contract: `VERSION`, `utils/build_deb.sh` -> `build/debs/*.deb`, `app/<lib>.h`, `build/release/lib<lib>.{a,so.<ver>}`, optional `deps.txt` + `utils/deps.sh`.

## Security

Actions pinned by SHA, `contents: read` default, `persist-credentials: false`, no `${{ }}` in `run:`, secrets passed by name. `lint.yml`: zizmor + actionlint.

Repo settings to set once: read-only workflow permissions, require SHA-pinned actions, rulesets on `master` and `v*` tags, secret scanning, CodeQL, 2FA.
