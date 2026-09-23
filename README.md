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

## c-quality.yml

Quality gate: clang-format check, build, strict gcc/clang compile, then unit tests + coverage, integration, stress, sanitizers and TSan when the repo has the script, then package build + install + smoke. Artifacts: `uts-coverage`, `integration-results`, `stress-results`, `deb-package`.

Caller (`quality.yml`, keeps its own triggers and concurrency, also `workflow_call` for release.yml):

```yaml
jobs:
  quality:
    uses: RomanHorshkov/cicd/.github/workflows/c-quality.yml@v1
    with: { lib-name: spscring, strict-cflags: "-std=c11 -O2 -Wall -Wextra -Wpedantic -Werror -DSPSC_REQUIRE_ALWAYS_LOCK_FREE" }
```

Inputs: `lib-name`, `strict-cflags`, `apt-packages` (default `pkg-config libcmocka-dev gcovr`), `tsan-runs-on`, `stress-timeout-minutes`.

## c-app-quality.yml, c-app-release.yml

For the deps.txt repositories (DB_*): `deps.sh install`, optional `vendor.sh verify`, build every profile (`build_libs.sh` and/or `build_bin.sh`), tests per profile (`build_tests.sh` first when present), optional integration (`run_ITs.sh`), optional fuzz smoke, package build + install (+ `smoke_test_package.sh` when present). Inputs: `profiles`, `profile-flag` (`--profile` for scripts that take it as an option), `fuzz-seconds`. The release runs `run_pipeline.sh` (then `build_deb.sh` if no deb came out), checksums, optional attestation, GitHub Release. Inputs: `title`, `attest` (default false). Both take the optional `SIBLING_REPOS_PAT` secret for private releases.

```yaml
jobs:
  quality:
    uses: RomanHorshkov/cicd/.github/workflows/c-app-quality.yml@v1
    with: { profile-flag: "--profile" }
    secrets: { SIBLING_REPOS_PAT: ${{ secrets.SIBLING_REPOS_PAT }} }
```

## Security

Actions pinned by SHA, `contents: read` default, `persist-credentials: false`, no `${{ }}` in `run:`, secrets passed by name. `lint.yml`: zizmor + actionlint.

Repo settings to set once: read-only workflow permissions, require SHA-pinned actions, rulesets on `master` and `v*` tags, secret scanning, CodeQL, 2FA.
