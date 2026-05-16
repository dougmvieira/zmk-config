# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a ZMK firmware configuration for the **Cradio (Ferris Sweep)** — a 34-key split Bluetooth keyboard. The left half is the BLE central; it pairs with the host and receives input from the right half over a separate split-pairing BLE link. The central can also output HID over USB when plugged in (toggle via `&out OUT_TOG`).

Firmware is built entirely via **GitHub Actions** — there is no local build toolchain in this repo. To get firmware artifacts, push changes and let the workflow run, or trigger it manually.

## Repository Structure

- `build.yaml` — build matrix; defines which board+shield combinations to compile
- `config/cradio.keymap` — the keymap with all layers and custom behaviors
- `config/cradio.conf` — runtime Kconfig overrides (currently empty)
- `config/west.yml` — pins the ZMK upstream dependency (currently `main` branch)

The Cradio shield itself lives upstream in ZMK (`app/boards/shields/cradio/`); no local shield is needed.

## Build Matrix

Defined in `build.yaml`:

| Board            | Shield           | Purpose                                       |
|------------------|------------------|-----------------------------------------------|
| `nice_nano//zmk` | `cradio_left`    | Left split half (BLE central, talks to host)  |
| `nice_nano//zmk` | `cradio_right`   | Right split half (BLE peripheral, talks to left) |
| `nice_nano//zmk` | `settings_reset` | Factory reset firmware (clears BT pairings)   |

The `//zmk` suffix selects the ZMK board variant introduced in Zephyr 4.1 / ZMK 2025-12-09 (see https://zmk.dev/blog/2025/12/09/zephyr-4-1#zmk-board-variant). Without it the board lacks the ZMK compat layer and the build fails. The default revision is 2.0.0 (nice!nano v2); for v1 hardware use `nice_nano@1.0.0//zmk`. The older `nice_nano_v2` and bare `nice_nano` names are no longer accepted.

After flashing new firmware, both halves must be factory-reset once (via `settings_reset.uf2`) before they will re-pair to each other.

## Keymap Architecture

`config/cradio.keymap` defines 5 layers:

1. **Default** — QWERTY with home-row mods. Left hand uses L-side modifiers (Alt/Ctrl/Shift/Gui on A/S/D/F); right hand uses R-side modifiers (Gui/Shift/Ctrl/Alt on J/K/L/;). The L/R split avoids same-hand mod-tap conflicts and OS shortcuts that distinguish L vs R modifiers.
2. **Layer 1** — Numpad
3. **Layer 2** — Arrows, Home/End, PgUp/PgDn, volume
4. **Layer 3** — Brackets, braces, parens, quotes
5. **Layer 4** (Flashing) — Reset, bootloader, BT profile selection, USB/BT output toggle (`&out OUT_TOG`); activated only when layers 2+3 are both held

Custom hold-tap behaviors:
- `qt` (quick_mod_tap) — 200 ms tap term, used on shift positions (D, K) where a faster tap is preferred
- `st` (slow_mod_tap) — 300 ms tap term, used on the other home-row mods
- `bspc_del` (backspace_delete) — mod-morph: Shift+Backspace produces Delete

## Adding or Changing Builds

To add a new board/shield combo, append an entry to the `include` list in `build.yaml`. Extra Kconfig flags go in a `cmake-args` field (e.g. `-DCONFIG_ZMK_USB_LOGGING=y`). Do not override `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` for `cradio_left` — upstream's `Kconfig.defconfig` already sets it correctly.

To change the ZMK upstream version, edit `config/west.yml` (change `revision:` from `main` to a tag or commit SHA).
