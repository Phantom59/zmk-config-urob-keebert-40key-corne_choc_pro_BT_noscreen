# ZMK Build Failure Post-Mortem

**Repo:** [Phantom59/zmk-config-urob](https://github.com/Phantom59/zmk-config-urob)
**Board:** Corne Choc Pro BT (nRF52840 split keyboard)
**Stack:** urob's ZMK fork v0.3 · Zephyr 3.5.0 · GitHub Actions

## Commits That Fixed It

| Commit | Change |
|--------|--------|
| [`aa6ddcb`](https://github.com/Phantom59/zmk-config-urob/commit/aa6ddcb) | Fix keymap include path: `../../` → `../../../base.keymap` |
| [`4228745`](https://github.com/Phantom59/zmk-config-urob/commit/4228745) | Switch both workflows to `urob/zmk-actions`, remove invalid `run-in-container` input |
| [`4e25415`](https://github.com/Phantom59/zmk-config-urob/commit/4e25415) | Remove `CONFIG_ZMK_BOARD_COMPAT=y` — undefined symbol in ZMK v0.3 Kconfig |

---

## Root Cause Analysis

Four independent defects compounded to produce build failures. Each would block the build on its own; they surfaced together because editing `build.yaml` to enumerate explicit board targets caused the matrix to expand from zero entries to two, executing code paths that had previously been dead.

### 1. Workflow version mismatch (primary blocker)

`build.yml` referenced `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main` — the upstream ZMK community workflow tracking Zephyr 4.1. A recent upstream commit added a compatibility-check step that calls `west boards --format "{qualifiers}"`. The `qualifiers` field was introduced in Zephyr 4.1's `boards.py`; it does not exist in the Zephyr 3.5.0 `BoardFile` dataclass. When the step runs against the repo's pinned Zephyr 3.5 environment (declared in `config/west.yml`), Python raises `KeyError: 'qualifiers'` and the job aborts before any compilation begins.

**Fix:** switch to `urob/zmk-actions`, which is versioned against ZMK v0.3 / Zephyr 3.5 and omits this check entirely.

### 2. Incorrect keymap include depth

`config/boards/arm/corne_choc_pro/corne_choc_pro.keymap` contained `#include "../../base.keymap"`. Two levels of `../` from that file's directory resolves to `config/boards/base.keymap`, which does not exist. The actual target is `config/base.keymap`, three levels up. The C preprocessor cannot recover from a missing include; CMake configuration aborts with a fatal error.

**Fix:** correct the path to `#include "../../../base.keymap"`.

### 3. Undefined Kconfig symbol in defconfig

`CONFIG_ZMK_BOARD_COMPAT=y` was added to both `*_defconfig` files as an attempted workaround for the compat check. This symbol does not exist in ZMK v0.3's Kconfig tree. Zephyr's Kconfig infrastructure treats assignment to an undefined symbol as a warning, and with `CONFIG_WARN_MISSING_SYMBOLS=y` (the default in ZMK builds) promotes it to a fatal error: `Aborting due to Kconfig warnings`.

**Fix:** remove the symbol — it is neither defined nor needed when using the correct workflow.

### 4. Unknown `with:` parameter in build-nix.yml

`build-nix.yml` passed `run-in-container: false` to `urob/zmk-actions`. That parameter is not declared in the reusable workflow's `inputs:` block. Recent GitHub Actions runner versions hard-fail on undeclared `with:` inputs. Earlier runs appeared green in 1–2 seconds only because the build matrix was empty (no targets listed before `build.yaml` was populated), so the reusable workflow was never actually invoked and the validation was skipped. Once the matrix became non-empty, the runner validated inputs before execution and immediately failed.

**Fix:** remove `run-in-container: false` from the `with:` block.

---

## Debugging Process

The investigation began with four GitHub Actions annotations: two "Check if building a board without explicit ZMK compat" failures and two `west build` exits with code 1. The Actions UI compounded the problem immediately — "There was an error while loading" blocked access to the full logs, so the only visible signal was the annotation summaries.

**Dead end: Kconfig hypothesis.** The compat check annotation pointed squarely at the board defconfigs. The obvious read was that `corne_choc_pro_left` and `corne_choc_pro_right` were missing `CONFIG_ZMK_BOARD_COMPAT=y`. That fix went in — and was wrong. The annotation was a symptom of a workflow mismatch, not a missing Kconfig symbol, but that wouldn't be clear for another two log reads.

**Getting the real log.** Once the raw `west build` output was obtained, two distinct failures separated cleanly. The first: `fatal error: ../../base.keymap: No such file or directory` at `corne_choc_pro.keymap:84`. The include path `../../base.keymap` resolves from `config/boards/arm/corne_choc_pro/` to `config/boards/` — one directory level short. Corrected to `../../../base.keymap`. The second failure in the same log: `KeyError: 'qualifiers'` in the compat check step — a Python crash with no obvious Kconfig cause.

**The Kconfig fix bites back.** The next build surfaced: `warning: attempt to assign the value 'y' to the undefined symbol ZMK_BOARD_COMPAT` followed by `error: Aborting due to Kconfig warnings`. The fix applied in step one was now the active breakage — `CONFIG_ZMK_BOARD_COMPAT` simply does not exist in ZMK v0.3.

**Version mismatch surfaces.** Correlating the runner environment (`zephyr_version: 4.1.0`) against the build output (`Zephyr version: 3.5.0`) identified the root cause of both the `qualifiers` crash and the undefined symbol: the `zmkfirmware/zmk` workflow was written for Zephyr 4.1 features against a repo pinned to 3.5. Switching both workflows to `urob/zmk-actions` resolved the toolchain alignment.

**The silent no-op.** One workflow had been reporting green in 1–2 seconds for months. That duration should have been suspicious — it was doing nothing. The job matrix was empty before `build.yaml` received real targets. Once populated, an invalid input (`run-in-container: false`) triggered a hard startup failure, exposing the workflow as having never run.

The consistent throughline: annotation summaries pointed at symptoms, not causes. Each actual fix only became possible after obtaining the raw log and treating each error independently.

---

## Key Lessons

- **"Succeeds in 1 second" is not success** — verify the matrix wasn't empty and work actually ran.
- **Annotation summaries lie** — always get the raw log before forming a hypothesis.
- **Multiple layered failures can share one error message** — the compat check fired for two different underlying reasons across different runs.
- **Workflow version ≠ toolchain version** — `@main` on a community workflow means "latest upstream", which drifts away from pinned dependencies silently.
- **Zephyr Kconfig aborts on undefined symbols** — `CONFIG_WARN_MISSING_SYMBOLS=y` is on by default; unknown symbols in defconfig are fatal, not ignored.
