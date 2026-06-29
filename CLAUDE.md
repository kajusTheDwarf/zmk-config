# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A [ZMK](https://zmk.dev) firmware configuration for a **SplitKB Aurora Lily58** split keyboard running on **nice!nano v2** (`nice_nano`) controllers.

> **Board naming (Zephyr 4.1 / Dec 2025):** ZMK moved to board *revisions*. The old `nice_nano_v2` board name no longer exists — use `nice_nano` (which defaults to revision `2.0.0`), or `nice_nano@2.0.0` to be explicit. Using `nice_nano_v2` causes a CI failure: `No board named 'nice_nano_v2' found`. This repo holds *only* the user config (keymap, conf, build matrix); the firmware itself, the shield definition, and the build toolchain all live in upstream Zephyr/ZMK modules pulled in at build time.

## Building

There is no local build. Firmware is built by GitHub Actions on every push/PR (`.github/workflows/build.yml`, which delegates to `zmkfirmware/zmk`'s `build-user-config.yml`). To get firmware:

1. Push the branch (or open a PR), then download the `firmware` artifact (`.uf2` files) from the Actions run.
2. Flash by putting each half into bootloader mode (double-tap reset) and copying the matching `splitkb_aurora_lily58_left` / `_right` `.uf2` onto the mounted drive.

`settings_reset` is also built — flash it to clear bluetooth/settings state when pairing breaks.

To build locally instead, set up the ZMK toolchain and run `west build -s zmk/app -b nice_nano -- -DSHIELD=splitkb_aurora_lily58_left -DZMK_CONFIG="$PWD/config"` from a west workspace initialized against `config/west.yml`.

## Layout of what matters

- `config/splitkb_aurora_lily58.keymap` — **the file you almost always edit.** Devicetree keymap with three layers: `default_layer` (0, base QWERTY), `lower_layer` (1, numpad + Swedish/German chars + symbols), `raise_layer` (2, function keys + nav arrows). Layers are reached via `&mo 1` / `&mo 2`.
- `config/splitkb_aurora_lily58.conf` — Kconfig feature flags (OLED display widgets are on; RGB underglow and encoders are commented out).
- `build.yaml` — the GitHub Actions build matrix. Add board/shield combos here.
- `config/west.yml` — west manifest. Declares the `zmk` and `zmk-helpers` projects and their remotes.
- `zephyr/module.yml` + `boards/shields/` — make this repo a Zephyr module so custom shields under `boards/shields/` would be discovered. Currently empty (`.gitkeep`).

## zmk-helpers (urob)

The keymap depends on [`urob/zmk-helpers`](https://github.com/urob/zmk-helpers), included two ways:
- as a **git submodule** at `config/zmk-helpers` (so `#include "zmk-helpers/..."` resolves locally), AND
- as a **west project** in `config/west.yml` (so CI fetches it).

After cloning, run `git submodule update --init --recursive` or the `#include`s won't resolve.

The keymap uses helper macros for unicode characters: `&de_ae`, `&de_ue`, `&de_oe`, `&de_eszett` (German) and `&sv_ao`, `&sv_ae`, `&sv_oe` (Swedish), pulled from `zmk-helpers/unicode-chars/german.dtsi` and `swedish.dtsi`. These rely on `#define HOST_OS 1` (set at the top of the keymap) to pick the OS-specific unicode input sequence — change it if the target OS changes (1 = macOS in urob's helpers; see the helper docs for other values).

## Editing the keymap

- It's a 58-key split: each layer's `bindings` is a flat array laid out as 4 rows of 12 (6 left + 6 right) plus a 6-key thumb row. The ASCII comment block above each layer documents the intended physical layout — keep it in sync when you change bindings.
- Key codes and behaviors come from upstream: `<dt-bindings/zmk/keys.h>` and `<behaviors.dtsi>`. Use `&kp`, `&mo`, `&trans`, etc.
- `&trans` = transparent (fall through to lower layer); use it for unused positions on non-base layers.
