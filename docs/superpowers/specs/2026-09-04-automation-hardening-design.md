# Automation Hardening Design

## Goal

Harden the existing GitHub Actions automation without changing the successful Android 15 / Linux 6.6 LTS build's intended feature set, root variants, artifact naming, or release-mode semantics. The first regression target is the already-proven `6.6.x-android15` + `os_patch_level=lts` path.

## Scope

### 1. Fail closed on the Samsung 6.6 device patch

Update `.github/actions/apply-device-patches/action.yml` so the Samsung `min_kdp` path is transactional:

1. Check the patch file and source paths before use.
2. Run the existing `patch --dry-run` check.
3. If the dry-run fails, exit nonzero with the patch command's diagnostic; do not copy `min_kdp.c` or modify `drivers/Makefile`.
4. Apply the patch only after the dry-run succeeds.
5. Append the `drivers/Makefile` entry only if an exact entry is not already present.

The Xiaomi symbol addition remains unchanged. This prevents a successful-looking build with a partially installed Samsung fix.

### 2. Make LTS selection case-insensitive and diagnostic

Update `.github/workflows/prepare.yml` to normalize the patch selector to lowercase before filtering the config matrix and pass the normalized value to downstream jobs. Existing `YYYY-MM`, numeric-sublevel, `All`, and exact `lts` behavior remains unchanged; uppercase forms such as `LTS` become equivalent to `lts`.

Update `.github/actions/extract-sublevel-file-name/action.yml` so the LTS path:

- checks that `kernel/common/Makefile` exists;
- extracts exactly one numeric `SUBLEVEL` assignment;
- fails with a clear diagnostic if it is missing, duplicated, or nonnumeric; and
- continues using the caller-provided sublevel for non-LTS builds.

The resulting filename and cache key continue to use the numeric sublevel read from the synced source.

### 3. Correct the legacy compiler variable

Update `.github/actions/build-kernel/action.yml` so the legacy build path uses `/usr/bin/ccache clang++` for `CXX`/`HOSTCXX`, not the invalid `clang+` executable name. The Bazel path and all other compiler settings remain unchanged.

### 4. Make verified-pin updates safe under malformed or concurrent input

Update `.github/scripts/update_verified_pins.py` to:

- match the audited scalar assignments with line-anchored, exact assignment patterns;
- require exactly one match for every assignment it intends to update before changing any file;
- fail before writing if a key is missing or duplicated;
- preserve the existing pin-selection logic and history payload; and
- use a collision-resistant history filename that retains the UTC timestamp prefix plus a short UUID suffix.

Update the `update-verified-pins` job in `.github/workflows/main.yml` with a repository-scoped concurrency group and `cancel-in-progress: false`, so manual promotion jobs queue instead of racing. This prevents simultaneous pin promotions from overwriting history or pushing conflicting commits; the unique filename also protects provenance if two processes reach the script close together.

### 5. Preserve generated device-support tables

Update `.github/scripts/update-supported-devices.py` to reject or safely escape Markdown table delimiters (`|`) and line breaks in every field that becomes a table cell. Keep legitimate device names, codenames, firmware strings, and free-form notes valid. Perform validation before writing `docs/supported-devices.md`, and retain the existing duplicate detection and generated-PR workflow.

## Explicit exclusions

These are intentionally deferred to separate designs because they can change build/release policy or require source fixtures:

- Complete reversion of all 6.6 SUSFS fake patches.
- Linting shell embedded in YAML Actions/workflows.
- Enabling `check_defconfig` or ABI validation gates.
- Adding build-log/warning artifacts.
- Pinning the moving LTS source branch or redesigning source-derived timestamps/provenance.
- Changing `Action`/`Pre-Release`/`Release` behavior or the `nightly` prerelease contract.

## Verification

Before the regression build, run from the repository root:

```bash
python3 .github/scripts/validate_workflows.py
python3 .github/scripts/validate_shell.py
python3 .github/scripts/validate_python.py
git diff --check
```

Also run focused, non-destructive checks for the Python replacement/table-field helpers and inspect the generated diff to confirm no writes occur on invalid input. Then dispatch the existing artifact-mode regression with `kernel_build_version=6.6.x-android15`, `os_patch_level=lts`, `commit_mode=verified`, and `root_flavor=All`. Confirm all three root variants complete successfully and that AnyKernel3/BuildInfo artifacts remain present. Do not promote a formal versioned release as part of this regression.
