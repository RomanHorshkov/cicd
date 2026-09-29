# cicd

Reusable GitHub Actions workflows for the kyr repositories: one CI family, `c-quality.yml` and `c-release.yml`, parameterised by inputs. Tags `v<major>.<minor>.<patch>`; callers pin the major (`@v2`). Current: v2.1.0 (see Versions).

## c-quality.yml

`kind: lib` (foundation-style: `build_libs.sh`, `app/<lib-name>.c`, UTs, ITs, stress, sanitizers, TSan, `lib<lib-name>{,-dev}` debs) or `kind: app` (template-style: profiles matrix, `build_libs.sh`/`build_bin.sh <profile>`, `build_tests.sh`/`run_tests.sh <profile>`, `run_ITs.sh`, fuzz smoke, every deb). Every kind: kyr-build bootstrapped by the exact version and sha256 in `deps.txt` (no sudo, into `build/sysroot`), `deps.sh sysroot`, `deps.sh check --tiers` (the tier law), `template_check.sh` (no drift from the template), `format.sh --check` and `vendor.sh verify` when present, the ABI gate when `abi/layout.txt` exists (job `abi-gate`: full history and tags, `utils/check_abi.sh`, then the release library and `utils/check_abi.sh --exports`; the package job waits for it), package build, install and smoke test. Artifacts: `uts-coverage`, `test-<profile>`, `integration-results`, `stress-results`, `fuzz-crashes`, `deb-package` (debs, `.buildinfo`, `SHA256SUMS`).

Inputs: `kind` (required), `lib-name`, `strict-cflags`, `apt-packages`, `profiles`, `profile-flag`, `fuzz-seconds`, `tsan-runs-on`, `stress-timeout-minutes`. Secret: `SIBLING_REPOS_PAT` (optional).

```yaml
jobs:
  quality:
    uses: RomanHorshkov/cicd/.github/workflows/c-quality.yml@v2
    with: { kind: lib, lib-name: spscring, strict-cflags: "-std=c11 -O2 -Wall -Wextra -Wpedantic -Werror -DSPSC_REQUIRE_ALWAYS_LOCK_FREE" }
```

## c-release.yml

Tag check against `VERSION`, the same bootstrap on the full history and tags, the ABI gate before packaging and the exports check after it when `abi/layout.txt` exists (a failure fails the build, so nothing is published), `utils/build_deb.sh` dated as on the workstation (kyr-build 2.0.3 on: the last commit that touched a build input; older: the commit), bundle (debs, `.buildinfo`, `SHA256SUMS`; kind lib also `<lib-name>-<ver>-linux-x86_64.tar.gz`). Built twice on separate runners; `compare` diffs the two `SHA256SUMS` and prints the `.buildinfo` differences on a mismatch; `publish` runs only when they agree. `workflow_dispatch` publishes nothing. Inputs: `kind` (required), `lib-name`, `title`, `attest` (default false), `apt-packages`, `double-build` (default true), `extra-assets` (space-separated globs copied into the bundle beside the debs and covered by `SHA256SUMS`; a glob that matches nothing fails the build; how a no-C repository such as kyr_shell ships `build/release/kyr-shell_*.tar` with `kind: app`).

```yaml
jobs:
  quality-gate:
    uses: ./.github/workflows/quality.yml
  release:
    needs: quality-gate
    permissions: { contents: write, id-token: write, attestations: write }
    uses: RomanHorshkov/cicd/.github/workflows/c-release.yml@v2
    with: { kind: lib, lib-name: spscring, title: SPSCring }
    secrets:
      SIBLING_REPOS_PAT: ${{ secrets.SIBLING_REPOS_PAT }}
```

## lint.yml

zizmor + actionlint over this repository's workflows, pinned. Callable from other repositories once D5 closes; not wired into any caller today.

## Security

Actions pinned by SHA, `contents: read` default, `persist-credentials: false`, no `${{ }}` in `run:`, secrets passed by name, the build never touches `/usr/local` (the install step of the package job is the runner-as-VM lane).

Repo settings to set once: read-only workflow permissions, require SHA-pinned actions, rulesets on `master` and `v*` tags, secret scanning, CodeQL, 2FA.

## Versions

- v2.1.0 (to be tagged by the owner; `v2` moves with it): the ABI gate in `c-quality` (job `abi-gate`) and in `c-release` (refuses to package), `fetch-depth: 0` where the version rule or the build-input epoch reads history, the release epoch left to kyr-build 2.0.3's rule, the kind lib tar dated like its debs.
- v2.0.0: one CI family, `kind: lib|app`, kyr-build bootstrap by version and sha256, tier and template checks, double build.

Diagrams: `docs/diagrams/workflow_family_components.puml`, `docs/diagrams/release_double_build_sequence.puml`.
