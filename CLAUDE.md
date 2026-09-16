# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository focus

This is the Wild Kernels GKI 2.0 build orchestrator. It builds generic Android kernels from Google's `kernel/common` sources (5.10 and newer) and layers in root implementations, SUSFS, optional kernel features, device fixes, and AnyKernel3 packaging. Kernel trees and most third-party sources are fetched during CI; they are not vendored in this repository. Generic GKI compatibility is broad but not guaranteed for every device.

The first planned build target is Android 15 / Linux 6.6 LTS: the workflow selector is `6.6.x-android15` and the patch selector is `lts`. Device-specific source trees belong in the separate device-oriented projects; keep this repository's changes generic unless the existing 6.6 device-patch action explicitly covers the device family.

## Commands

### Validate changes locally

The repository's validation workflow is `.github/workflows/validate.yml`. From the repository root:

```bash
pip install pyyaml
python3 .github/scripts/validate_workflows.py
python3 .github/scripts/validate_shell.py
python3 .github/scripts/validate_python.py
git diff --check
```

`validate_workflows.py` parses every `.github/workflows/*.yml`/`.yaml` file and checks the expected workflow/job/step structure with PyYAML. It is not `actionlint` or `yamllint`. `validate_shell.py` runs `bash -n` over tracked shell files (there are currently no shell scripts), and `validate_python.py` AST-parses Python files. There is no repository test suite, Makefile, Gradle project, or per-test runner; for a focused change, run the validator corresponding to the file type and then the full validation set before submitting.

### Start the 6.6 LTS build

Builds are designed to run on GitHub-hosted Ubuntu runners through `main.yml`, not from a clean local checkout. With GitHub CLI authenticated to the repository, dispatch the first reproducible 6.6 build with:

```bash
gh workflow run main.yml --ref main \
  -f release_type=Action \
  -f kernel_build_version=6.6.x-android15 \
  -f os_patch_level=lts \
  -f commit_mode=verified \
  -f root_flavor=All \
  -f use_cache=true
```

`lts` is intentional: `.github/actions/extract-sublevel-file-name` reads the actual `SUBLEVEL` from the synced kernel `Makefile` instead of requiring a hard-coded sublevel. `verified` uses the audited component pins and is the default reproducible mode. Use `latest` only when deliberately testing upstream branch tips; `update` is an operational mode that promotes pins after successful builds and can mutate repository state.

Useful GitHub Actions commands:

```bash
gh run list --workflow main.yml --limit 10
gh run watch <run-id> --exit-status
gh run download <run-id>
```

`build.yml` and `prepare.yml` are reusable `workflow_call` workflows and cannot be dispatched directly. `managers.yml` is a separate manual workflow for fetching/building root manager artifacts; `clear-cache.yml` is a destructive GitHub-only cache operation and requires its explicit confirmation input.

## Architecture

- `.github/workflows/main.yml` is the public build dispatcher. It accepts the release type, version family, patch-level filter, feature toggles, root flavor, cache choice, and source `commit_mode`. `resolve-sources` resolves KernelSU/KernelSU-Next/ReSukiSU, SUSFS, NoMount, kernel-patches, AnyKernel3, and DroidSpaces commits, then the version jobs fan out over the selected Android/kernel families. `root_flavor: All` creates builds for KernelSU-Next, KernelSU, and ReSukiSU.
- `.github/config/android*.json` is the build matrix. Each file maps a version such as `android15-6.6` to Android patch dates and kernel sublevels; the `lts` row is a selector for the latest sublevel found in the synced source. Do not manually duplicate matrix entries in workflow YAML when the JSON config is the source of truth.
- `.github/workflows/prepare.yml` converts the selected config and patch-level filter into a build matrix, then calls `build.yml` once per selected date/sublevel/variant. It passes resolved component commits, feature flags, root settings, branding, bypass, and cache settings downstream.
- `.github/workflows/build.yml` runs the per-kernel pipeline on Ubuntu: set up the build environment, initialize and sync the Android GKI manifest, fix and patch the kernel, configure optional features, set up the selected root/SUSFS integration, compile with the source branch's legacy `build/build.sh` or Bazel/kleaf path, and package/upload the resulting kernel artifacts. The composite actions under `.github/actions/` implement these stages; edit or add an action when behavior is reusable across version families.
- `.github/actions/` also contains the feature and compatibility layers: kernel config setters, SUSFS/root setup and patches, NoMount, networking/BPF/performance/NTSync/DroidSpaces, branding, device patches, cache restore/save, manager fetching, and reject scanning. Most actions assume the checked-out/fetched kernel lives under the CI workspace's `kernel/` directory.
- `docs/` describes installation, supported-device reporting, feature behavior, post-install manager/module requirements, KernelFlasher, and manual `magiskboot` repacking. Device-support issues are processed by `.github/workflows/device-support.yml` and update `docs/supported-devices.md`; blank issues are disabled.
- `THIRD_PARTY_NOTICES.md` records licenses and provenance for sources fetched into release ZIPs. Keep third-party attribution aligned when changing fetched components.

## Version and release rules

- Supported matrix families currently include Android 12/13 on 5.10, Android 13/14 on 5.15, Android 14 on 6.1, Android 15 on 6.6, and Android 16 on 6.12. Use the exact `android<release>-<kernel>` naming already used by `.github/config/` and workflow inputs.
- `os_patch_level` accepts a `YYYY-MM` patch date, a kernel sublevel, `lts`, or `All`. Restrict it when validating one source snapshot; use `lts` for the current 6.6 LTS target.
- Keep `commit_mode=verified` for normal/reproducible builds. Pin updates are intentionally separated from ordinary builds; do not silently replace audited SHAs with moving branch tips.
- `release_type=Action` is appropriate for an initial build/artifact test and skips the versioned `rN` release path, but a successful run still creates or updates the `nightly` prerelease. Use `Pre-Release` or `Release` only when the release contents and device compatibility have been checked.
- The root manager, kernel, and module versions must remain compatible. When triaging a device report, collect the exact flashed filename, release, original/flashed kernel versions, manager version, and the required logs described by `.github/ISSUE_TEMPLATE/bug_report.yml`.
