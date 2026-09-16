# Automation Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden the repository's existing automation against silent 6.6 patch failures, ambiguous LTS selection, malformed pin promotion, concurrent history collisions, and corrupted generated device tables without changing build features or release modes.

**Architecture:** Keep the current composite Actions, reusable workflows, and Python utilities in place. Make each boundary fail explicitly before it mutates source or documentation, and add only the smallest helper logic needed for exact validation, collision-resistant history names, and idempotent table edits. Verify with focused temporary fixtures, the existing validators, and a fresh Android 15 / 6.6 LTS artifact build.

**Tech Stack:** Bash embedded in GitHub Actions, GitHub Actions YAML, Python 3 standard library, PyYAML validator, GitHub CLI for the regression dispatch.

**Spec:** `docs/superpowers/specs/2026-09-04-automation-hardening-design.md`

## Global Constraints

- Preserve the successful `6.6.x-android15` + `os_patch_level=lts` build's feature set, root variants, artifact naming, and release-mode semantics.
- Fail before writing when a required patch, numeric LTS sublevel, verified pin assignment, or device-table field is invalid.
- Keep `release_type=Action` behavior unchanged: skip the versioned `rN` release path while retaining the existing `nightly` prerelease behavior.
- Do not enable ABI/defconfig gates, redesign LTS provenance, add YAML shell linting, or redesign SUSFS fake-patch reversion in this plan.
- Do not commit or push changes unless the user explicitly requests it.

---

### Task 1: Make the 6.6 device patch transactional

**Files:**
- Modify: `.github/actions/apply-device-patches/action.yml:14-30`

**Interfaces:**
- Consumes: `version` and `kernel_version` action inputs plus the fetched `kernel_patches/samsung/min_kdp` files.
- Produces: either a fully applied Samsung `min_kdp` fix or a nonzero Action step before copying source or modifying `drivers/Makefile`.

- [ ] **Step 1: Add explicit source and patch preconditions**

Inside the existing Samsung `run` block, keep the working directory and input condition, then add `set -euo pipefail` and explicit `test -f` checks for:

```bash
PATCH="${{ github.workspace }}/kernel_patches/samsung/min_kdp/add-min_kdp-symbols.patch"
MIN_KDP="${{ github.workspace }}/kernel_patches/samsung/min_kdp/min_kdp.c"
test -f "$PATCH" || { echo "Missing Samsung min_kdp patch: $PATCH" >&2; exit 1; }
test -f "$MIN_KDP" || { echo "Missing Samsung min_kdp source: $MIN_KDP" >&2; exit 1; }
```

- [ ] **Step 2: Make dry-run failure fatal and prevent partial mutation**

Replace the current conditional patch application with:

```bash
patch -p1 --dry-run < "$PATCH"
patch -p1 --no-backup-if-mismatch < "$PATCH"
cp "$MIN_KDP" drivers/min_kdp.c
```

The second command must not run if the dry-run fails because strict mode exits first.

- [ ] **Step 3: Make the Makefile append idempotent**

Replace the unconditional append with an exact-line check:

```bash
if ! grep -Fqx 'obj-y += min_kdp.o' drivers/Makefile; then
  printf '%s\n' 'obj-y += min_kdp.o' >> drivers/Makefile
fi
```

- [ ] **Step 4: Run static validation for the Action change**

Run:

```bash
python3 .github/scripts/validate_workflows.py
python3 .github/scripts/validate_python.py
python3 .github/scripts/validate_shell.py
```

Expected: all validators exit 0. The shell validator may still report that no standalone `.sh` files exist; that is expected and does not validate embedded Action shell.

---

### Task 2: Normalize LTS selection and validate extracted sublevels

**Files:**
- Modify: `.github/workflows/prepare.yml:136-154`
- Modify: `.github/actions/extract-sublevel-file-name/action.yml:31-51`

**Interfaces:**
- Consumes: workflow input `os_patch_level`, config entries with `date`/`sublevel`, and the fetched `kernel/common/Makefile`.
- Produces: a normalized matrix `os_patch_level` and a numeric extracted sublevel for exact lowercase `lts` builds.

- [ ] **Step 1: Normalize the selector once in `prepare.yml`**

Replace the current `SELECTED_PATCH_LEVEL` setup with a lowercase normalization that preserves the existing `All` behavior:

```bash
SELECTED_PATCH_LEVEL="$(printf '%s' '${{ inputs.os_patch_level }}' | tr '[:upper:]' '[:lower:]')"
if [ "$SELECTED_PATCH_LEVEL" = "all" ]; then
  SELECTED_PATCH_LEVEL=""
fi
```

Use this normalized value for the date/sublevel filter and ensure the selected matrix entries carry `lts` when the caller supplied `LTS`.

- [ ] **Step 2: Validate exactly one numeric `SUBLEVEL` in the extraction Action**

Replace the `grep | awk` pipeline with a temporary match list and explicit checks. The LTS branch should implement this behavior:

```bash
MAKEFILE="${{ github.workspace }}/kernel/common/Makefile"
if [ "$(printf '%s' '${{ inputs.os_patch_level }}' | tr '[:upper:]' '[:lower:]')" = "lts" ]; then
  if [ ! -f "$MAKEFILE" ]; then
    echo "LTS build requires $MAKEFILE" >&2
    exit 1
  fi
  mapfile -t matches < <(grep -E '^[[:space:]]*SUBLEVEL[[:space:]]*=' "$MAKEFILE" || true)
  if [ "${#matches[@]}" -ne 1 ]; then
    echo "Expected exactly one SUBLEVEL assignment in $MAKEFILE; found ${#matches[@]}" >&2
    exit 1
  fi
  EXTRACTED="${matches[0]#*=}"
  EXTRACTED="$(printf '%s' "$EXTRACTED" | tr -d '[:space:]')"
  if ! [[ "$EXTRACTED" =~ ^[0-9]+$ ]]; then
    echo "Invalid numeric SUBLEVEL '$EXTRACTED' in $MAKEFILE" >&2
    exit 1
  fi
  SUBLEVEL="$EXTRACTED"
fi
```

Keep the existing caller-provided sublevel path for non-LTS builds and keep filename/output writes unchanged.

- [ ] **Step 3: Run a focused extraction fixture**

Run this temporary fixture from the repository root to verify the validation rules without touching the fetched kernel or GitHub state:

```bash
python3 - <<'PY'
import re

def extract(makefile):
    matches = re.findall(r'^[ \t]*SUBLEVEL[ \t]*=', makefile, re.MULTILINE)
    if len(matches) != 1:
        raise ValueError(f"expected one SUBLEVEL, found {len(matches)}")
    value = re.search(r'^[ \t]*SUBLEVEL[ \t]*=\s*([^\s#]+)', makefile, re.MULTILINE).group(1)
    if not value.isdigit():
        raise ValueError(value)
    return value

assert extract('VERSION = 6\nSUBLEVEL = 142\n') == '142'
for bad in ('VERSION = 6\n', 'SUBLEVEL = 142\nSUBLEVEL = 143\n', 'SUBLEVEL = X\n'):
    try:
        extract(bad)
    except (ValueError, AttributeError):
        pass
    else:
        raise AssertionError(bad)
print('LTS extraction fixture: PASS')
PY
```

Expected: `LTS extraction fixture: PASS`.

---

### Task 3: Correct the legacy compiler command

**Files:**
- Modify: `.github/actions/build-kernel/action.yml:20-34`

**Interfaces:**
- Consumes: the existing legacy `build/build.sh` branch.
- Produces: valid `CXX` and `HOSTCXX` values pointing to `/usr/bin/ccache clang++`; Bazel behavior is unchanged.

- [ ] **Step 1: Replace both invalid executable names**

Change only these assignments:

```bash
CXX="/usr/bin/ccache clang++" \
HOSTCXX="/usr/bin/ccache clang++"
```

Do not alter `CC`, `HOSTCC`, Bazel target selection, or disabled defconfig behavior in this task.

- [ ] **Step 2: Assert the typo is gone**

Run:

```bash
python3 - <<'PY'
from pathlib import Path
text = Path('.github/actions/build-kernel/action.yml').read_text()
assert 'clang+' not in text.replace('clang++', '')
assert 'CXX="/usr/bin/ccache clang++"' in text
assert 'HOSTCXX="/usr/bin/ccache clang++"' in text
print('legacy compiler assignments: PASS')
PY
```

Expected: `legacy compiler assignments: PASS`.

---

### Task 4: Harden verified-pin promotion and concurrency

**Files:**
- Modify: `.github/scripts/update_verified_pins.py:10-111`
- Modify: `.github/workflows/main.yml:818-868`

**Interfaces:**
- Consumes: the existing `PIN_*` assignments and `NOMOUNT_SHA`/`KERNELSU_SHA`/`RESUKISU_SHA` environment variables.
- Produces: exact-one-match pin reads/replacements, collision-resistant history JSON paths, and serialized `commit_mode=update` jobs.

- [ ] **Step 1: Add collision-resistant history names**

Import `uuid4` from `uuid` and change `history_path()` to retain the current UTC-second prefix while appending an 8-character UUID suffix:

```python
from uuid import uuid4


def history_path():
    ts = datetime.now(timezone.utc).strftime("%Y-%m-%dT%H%M%SZ")
    d = os.path.join(REPO, ".github/pins/history")
    os.makedirs(d, exist_ok=True)
    return os.path.join(d, f"{ts}-{uuid4().hex[:8]}.json")
```

- [ ] **Step 2: Require exactly one assignment when reading pins**

In `read_pins()`, read the file once and use an anchored multiline expression per key:

```python
pattern = re.compile(rf'^[ \t]*{re.escape(var)}="([0-9a-f]{{40}})"[ \t]*$', re.MULTILINE)
matches = list(pattern.finditer(text))
if len(matches) != 1:
    raise ValueError(f"expected exactly one {var} assignment, found {len(matches)}")
pins[var] = matches[0].group(1)
```

This must fail before `history_path()`, `write_history()`, or `apply_pins()` can mutate anything when the expected assignment is missing or duplicated.

- [ ] **Step 3: Make replacements exact and fail before writing**

In `apply_pins()`, read the file once, validate each promoted key with the same anchored exact-assignment pattern, require one match, and build the complete replacement text in memory. Only open `MAIN` for writing after all promoted keys validated. Replace the matched assignment line while preserving its indentation and newline style.

- [ ] **Step 4: Serialize update-mode promotion jobs**

Add job-level concurrency to `update-verified-pins` before `if`/`runs-on`:

```yaml
    concurrency:
      group: update-verified-pins-${{ github.repository }}
      cancel-in-progress: false
```

This queues overlapping update jobs rather than cancelling one or allowing simultaneous commits. Leave all existing `needs`, permissions, approval environment variables, and commit/push steps unchanged.

- [ ] **Step 5: Run a temporary pin-updater fixture**

Run a focused Python check that imports the updater against a temporary repository layout and verifies unique history names plus duplicate-assignment rejection:

```bash
python3 - <<'PY'
import importlib.util
import os
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as root:
    main = Path(root) / '.github/workflows/main.yml'
    main.parent.mkdir(parents=True)
    main.write_text('PIN_NOMOUNT="' + 'a' * 40 + '"\nPIN_KERNELSU="' + 'b' * 40 + '"\nPIN_RESUKISU="' + 'c' * 40 + '"\n')
    old = os.environ.get('GITHUB_WORKSPACE')
    os.environ['GITHUB_WORKSPACE'] = root
    try:
        spec = importlib.util.spec_from_file_location('pin_updater', '.github/scripts/update_verified_pins.py')
        mod = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(mod)
        assert mod.read_pins()['PIN_KERNELSU'] == 'b' * 40
        assert mod.history_path() != mod.history_path()
        main.write_text(main.read_text() + 'PIN_KERNELSU="' + 'd' * 40 + '"\n')
        try:
            mod.read_pins()
        except ValueError as exc:
            assert 'exactly one PIN_KERNELSU' in str(exc)
        else:
            raise AssertionError('duplicate assignment was accepted')
    finally:
        if old is None:
            os.environ.pop('GITHUB_WORKSPACE', None)
        else:
            os.environ['GITHUB_WORKSPACE'] = old
print('pin updater fixture: PASS')
PY
```

Expected: `pin updater fixture: PASS` and no repository files changed.

---

### Task 5: Validate device-support table fields before writing

**Files:**
- Modify: `.github/scripts/update-supported-devices.py:41-73`

**Interfaces:**
- Consumes: parsed issue-form fields for manufacturer, device, codename, GKI, firmware, and status.
- Produces: a valid single-row Markdown table update or a nonzero validation error before `MD.write_text()`.

- [ ] **Step 1: Add a table-cell validator**

Add a helper before field extraction:

```python
def validate_cell(name, value):
    if any(ch in value for ch in ('|', '\r', '\n')):
        print(f"invalid {name}: Markdown delimiters and line breaks are not allowed", file=sys.stderr)
        raise SystemExit(1)
    return value
```

Apply it to every value interpolated into `row`, including `device`, `codename`, `gki`, `firmware`, and `status`; validate `manufacturer`/`custom_oem` before deriving a heading as well. Preserve legitimate punctuation other than table delimiters and line breaks.

- [ ] **Step 2: Test invalid and valid fields in a temporary copy**

Use a temporary directory containing a minimal `docs/supported-devices.md` and run the updater with an issue body containing `Device: Foo | Bar`; assert nonzero exit and unchanged Markdown. Repeat with `Device: Foo 5G`, `Codename: foo`, `GKI Kernel: 6.6`, and `Firmware: stock` and assert one correctly delimited row. Do not run the updater against the repository's real docs file during this focused test.

- [ ] **Step 3: Run Python validation**

```bash
python3 .github/scripts/validate_python.py
```

Expected: exit 0 and all Python files parse successfully.

---

### Task 6: Run repository validation and inspect the complete diff

**Files:**
- Inspect: all files modified by Tasks 1–5

**Interfaces:**
- Consumes: the completed hardening edits.
- Produces: a clean, reviewable diff with all existing validators passing.

- [ ] **Step 1: Run all repository validators**

```bash
python3 .github/scripts/validate_workflows.py
python3 .github/scripts/validate_shell.py
python3 .github/scripts/validate_python.py
git diff --check
```

Expected: all commands exit 0. `validate_shell.py` may report `No shell scripts found`; that is an existing repository fact, not a failure.

- [ ] **Step 2: Inspect changed paths and ensure exclusions stayed untouched**

```bash
git status --short
git diff -- .github/actions/apply-device-patches/action.yml \
  .github/workflows/prepare.yml \
  .github/actions/extract-sublevel-file-name/action.yml \
  .github/actions/build-kernel/action.yml \
  .github/scripts/update_verified_pins.py \
  .github/workflows/main.yml \
  .github/scripts/update-supported-devices.py
```

Confirm the diff does not modify ABI/defconfig gates, SUSFS reversion policy, LTS source pinning, release conditions, or unrelated workflow jobs.

- [ ] **Step 3: Dispatch the approved 6.6 LTS regression build**

After the user-visible diff review, dispatch:

```bash
gh workflow run main.yml --ref main \
  -f release_type=Action \
  -f use_cache=true \
  -f bypass=false \
  -f kernel_build_version=6.6.x-android15 \
  -f os_patch_level=lts \
  -f brand_name=Wild \
  -f commit_mode=verified \
  -f root_flavor=All \
  -f use_susfs=true
```

- [ ] **Step 4: Verify the regression result**

Run `gh run view <run-id> --json status,conclusion,jobs` and confirm the three Android 15 / 6.6 LTS root jobs succeed. Confirm AnyKernel3 and BuildInfo artifacts exist for KernelSU-Next, KernelSU, and ReSukiSU. The versioned release job must remain skipped; the existing Action-mode `nightly` prerelease behavior is expected.
