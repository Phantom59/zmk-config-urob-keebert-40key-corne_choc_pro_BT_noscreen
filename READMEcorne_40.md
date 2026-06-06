# Corne Choc Pro BT — 40-Key urob Keymap Port

**Hardware:** Keebart Corne Choc Pro BT · nRF52840 · 40 physical keys (36 + 4 inner display columns)
**Firmware:** ZMK via [urob's fork](https://github.com/urob/zmk) at `v0.3` · Zephyr 3.5.0
**Forked from:** [stphn/zmk-config-urob](https://github.com/stphn/zmk-config-urob)

---

## What This Is

This repo is a complete port of [urob's 34/36-key Colemak-DH keymap](https://github.com/urob/zmk-config) onto the Corne Choc Pro BT — a 40-key wireless split keyboard with inner display columns and optional encoders. The original stphn repo had a 46-key QWERTY layout. We replaced it entirely with urob's layered system: homerow mods, magic shift, combos, mouse layer, adaptive keys, tri-state, unicode, and a 3-thumb outer thumb cluster.

Everything below describes what we actually did, in order, and exactly what went wrong and how we figured it out.

---

## The Full Commit History — What We Did and Why

### Phase 1 — Setup (June 5, 2026)

**[`8aeb55a`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/8aeb55a) — Update build.yaml**

Changed the build targets from a single generic board to explicit left/right variants:

```yaml
include:
  - board: corne_choc_pro_left
  - board: corne_choc_pro_right
```

This was necessary because the two halves need separate firmware — different matrix transforms, different encoder pin assignments, different column offsets for the right side. This single change is what set off the entire debugging chain that followed, because it caused the GitHub Actions matrix to expand from zero entries to two, hitting code paths that had never been exercised before.

---

**[`9c20716`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/9c20716) [`0c3b85d`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/0c3b85d) [`415840e`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/415840e) [`e5d4d16`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/e5d4d16) — Update west.yml (×4)**

Four iterations tuning the west manifest to correctly pin urob's ZMK modules at `v0.3`:

- `zmk-adaptive-key`
- `zmk-auto-layer`
- `zmk-helpers`
- `zmk-leader-key`
- `zmk-tri-state`
- `zmk-unicode`

These modules are what make urob's keymap tick — they provide the behaviours (`adaptive_key`, `auto_layer`, `tri_state`, etc.) that the base keymap depends on. Getting the pinning right took several attempts because the module API changed between versions.

---

**[`0b53ffe`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/0b53ffe) — Port urob 40-key keymap to Corne Choc Pro BT**

The core work. Seven files changed. This is where the actual keyboard engineering happened.

#### `corne_choc_pro.dtsi` — Matrix transform fix

The original transform declared `columns = <12>` but the Corne Choc Pro has 14 columns — 6 standard columns per side plus 2 inner display columns (`RC(r,6)` and `RC(r,7)` for the OLED/display area). These inner keys are what give the board 40 physical keys instead of 36. Without this fix the inner 4 keys were invisible to ZMK.

```dts
// Before
columns = <12>;
// After
columns = <14>;
```

#### `corne_choc_pro-layouts.dtsi` — Position map validation

The `five_col_map` was referencing key positions 40–45 — positions that don't exist in a 40-key layout. This would have caused ZMK to silently drop or misroute those keys at runtime. Corrected every position reference to stay within 0–39. Also removed the `complete` keyword from the position map since the physical layout doesn't cover every matrix position.

#### `corne_choc_pro_right.dts` — Column offset correction

The right half had `col-offset = <5>` in its `five_col_transform`, which conflicted with the `default_transform` offset. With 14 columns (7 per side) the correct offset is `<7>`:

```dts
// Before
col-offset = <5>;
// After
col-offset = <7>;
```

Without this the right half's key matrix would collide with the left half's columns, causing phantom keypresses and dropped keys.

#### Encoders disabled

Both `.dts` files had encoders set to `"okay"` but the keymap had no `sensor-bindings` defined for them. ZMK aborts at build time if an encoder is enabled but has no bindings. Disabled both:

```dts
status = "disabled";
```

#### `corne_choc_pro.keymap` — Complete rewrite

Replaced the original 128-line QWERTY keymap with a full urob-style 40-key implementation. The keymap defines position labels for all 40 keys, enables wireless Bluetooth bindings, and uses a `ZMK_BASE_LAYER` macro that injects the 4 inner keys (encoder area: `C_MUTE`, `C_PP`, `LALT`, `RALT`) and the 2 outer thumb keys (`LGUI`, `smart_mouse`) around urob's standard `LT/RT/LM/RM/LB/RB/LH/RH` groups. The actual layer definitions are pulled in from `base.keymap` — urob's shared keymap file.

#### Defconfig additions (both halves)

Added to both `*_defconfig` files:

```kconfig
CONFIG_ZMK_POINTING=y          # mouse / trackpad support
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y # +8 dBm TX power for range
CONFIG_BT_MAX_CONN=4            # up to 4 simultaneous connections
CONFIG_BT_MAX_PAIRED=4          # 4 pairing slots
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=1800000  # 30-minute sleep timeout
```

---

### Phase 2 — The Build Failures (June 6, 2026)

After pushing the keymap port, GitHub Actions started failing. What followed was four rounds of debugging across six commits — each one peeling back another layer.

---

**[`8351e88`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/8351e88) — Add CONFIG_ZMK_BOARD_COMPAT=y (dead end)**

The Actions UI showed an annotation: *"Check if building a board without explicit ZMK compat."* The obvious read was that the board's defconfigs were missing a Kconfig flag. Added `CONFIG_ZMK_BOARD_COMPAT=y` to both.

This was wrong. The annotation was a symptom of a workflow version mismatch, not a missing Kconfig symbol. And because `CONFIG_ZMK_BOARD_COMPAT` does not exist in ZMK v0.3's Kconfig tree at all, Zephyr would abort on it later. But that wasn't visible yet.

---

**[`aa6ddcb`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/aa6ddcb) — Fix keymap include path**

The raw build log (not the annotation — the raw log) showed:

```
fatal error: ../../base.keymap: No such file or directory
```

The keymap at `config/boards/arm/corne_choc_pro/corne_choc_pro.keymap` was including `../../base.keymap`. Two levels up from that directory is `config/boards/` — `base.keymap` isn't there. It lives at `config/base.keymap`, which requires three levels:

```
Before: #include "../../base.keymap"
After:  #include "../../../base.keymap"
```

Off by one directory level. A classic include path bug — invisible until you actually trace the filesystem tree by hand.

---

**[`4228745`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/4228745) — Switch both workflows to urob/zmk-actions**

The next log exposed the real root cause. Two things visible simultaneously:

1. The runner environment said `zephyr_version: 4.1.0`
2. The actual build said `Zephyr version: 3.5.0`

The build workflow (`build.yml`) was using `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main` — the upstream community workflow, which had been updated to require Zephyr 4.1 features. One of those features was `west boards --format "{qualifiers}"`. The `qualifiers` field was added to Zephyr 4.1's `boards.py`; it doesn't exist in 3.5. Result:

```
KeyError: 'qualifiers'
```

The compat check was always going to crash against this repo's Zephyr 3.5 environment. No Kconfig flag was ever going to fix it.

The fix: both workflows switched to `urob/zmk-actions` — urob's own GitHub Actions workflow, built and versioned for ZMK v0.3 / Zephyr 3.5, with no `{qualifiers}` check.

Also fixed in this commit: `build-nix.yml` was passing `run-in-container: false` — a parameter that `urob/zmk-actions` doesn't accept. GitHub Actions had started hard-failing on unknown `with:` inputs to reusable workflows. The nix workflow had been silently "succeeding" in 1–2 seconds for months because the build matrix was empty before `build.yaml` had real targets — the workflow was being invoked with nothing to do. Once the matrix had real entries, the unknown parameter caused an immediate startup failure.

---

**[`4e25415`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/4e25415) — Remove CONFIG_ZMK_BOARD_COMPAT from defconfigs**

The dead-end fix from `8351e88` came back to bite. The next build log showed:

```
warning: attempt to assign the value 'y' to the undefined symbol ZMK_BOARD_COMPAT
error: Aborting due to Kconfig warnings
```

`CONFIG_ZMK_BOARD_COMPAT` doesn't exist anywhere in ZMK v0.3's Kconfig tree. Zephyr treats assignment to an undefined symbol as a warning, and with `CONFIG_WARN_MISSING_SYMBOLS=y` (the default in ZMK builds), it promotes that warning to a fatal abort. The symbol was removed from both defconfigs. With `urob/zmk-actions` now in place, it was never needed anyway — urob's workflow has no compat check.

---

**[`97d5463`](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/commit/97d5463) — Add build failure post-mortem doc**

Detailed technical post-mortem written to `docs/build-debug-postmortem.md` documenting all four compounding failures and the debugging process.

---

## The Eureka Moments

There were three of them.

**First:** The annotation said "ZMK compat" — which pointed straight at Kconfig and defconfigs. That was the wrong trail. The real signal was buried in the raw log: `KeyError: 'qualifiers'` in a Python traceback. That crash had nothing to do with Kconfig. Once we saw that `zephyr_version: 4.1.0` in the runner env vs `Zephyr version: 3.5.0` in the build output, the whole workflow mismatch snapped into focus.

**Second:** The include path bug. `../../base.keymap` looks right at a glance — two levels up from the keymap file. But the file is at `config/boards/arm/corne_choc_pro/corne_choc_pro.keymap`, so two levels up lands at `config/boards/`, not `config/`. You have to trace the actual path segment by segment to see it. The compiler error was precise — it just required reading it carefully.

**Third:** The nix workflow "succeeded" in 1–2 seconds for months. That should have been the first red flag — a ZMK firmware build cannot complete in 2 seconds. It was a ghost: the matrix was empty, no jobs ever ran, GitHub just reported the workflow as passed. The startup failure that eventually appeared wasn't a regression — it was the first time the workflow had been asked to do real work.

---

## Lessons

- **"Succeeds in 1 second" is not success.** Check whether the matrix was empty and work actually ran.
- **Annotation summaries lie.** Always get the raw log before forming a hypothesis.
- **Multiple layered failures can share one error message.** The compat check fired for different underlying reasons across different runs.
- **`@main` on a community workflow means drift.** It tracks latest upstream, which moves away from pinned dependencies silently. Pin to a fork that matches your toolchain.
- **Zephyr Kconfig aborts on undefined symbols.** `CONFIG_WARN_MISSING_SYMBOLS=y` is on by default. Unknown symbols in defconfig are fatal, not ignored.
- **Trace include paths segment by segment.** `../../` looks like "go up two", but the starting directory matters entirely.

---

## Build

Builds run automatically on push to `config/**` or `build.yaml` via both workflows:

- `build.yml` — uses `urob/zmk-actions` with `toolchain: zephyr`
- `build-nix.yml` — same, with Nix dev shell support

Artifacts (`.uf2` files for both halves) are available in the [Actions tab](https://github.com/Phantom59/zmk-config-urob-keebert-40key-corne_choc_pro_BT_noscreen/actions) after a successful run.

## Flash

1. Download `firmware.zip` from the latest successful Actions run
2. Put each half into bootloader mode (double-tap reset)
3. Drag `corne_choc_pro_left-zmk.uf2` to the left half's USB drive
4. Drag `corne_choc_pro_right-zmk.uf2` to the right half's USB drive
