# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A [ZMK](https://zmk.dev) firmware configuration repo for the "KEYPOINT" split keyboard (ZitaoTech), a
split ergonomic board with a nice!view-style display on one half, a trackpad on the left half, and a
trackpoint on the right half. This is a **config-only** repo (`west`/Zephyr build manifest + devicetree
overlays) — there is no application source to build locally; firmware is compiled by ZMK's GitHub Actions
workflow, not on this machine.

## Build / CI (no local build)

- Firmware is built entirely via GitHub Actions — pushing to `main` (or opening a PR) triggers
  `.github/workflows/main.yml`, which calls ZMK's reusable `build-user-config.yml` workflow using the
  ZMK revision pinned in `config/west.yml` (`v0.3.0`).
- `build.yaml` at the repo root defines the two build targets (board+shield combos): the boards are
  `zitaotech_keypoint_left` / `zitaotech_keypoint_right`, each paired with `lpm_view` (display) plus
  `left_bbtrackpad_keypoint` or `right_trackpoint_keypoint` respectively. A commented-out block shows how
  to add `settings_reset` variants for BLE-pairing resets.
- Resulting `.uf2` firmware files come from the Actions run artifact (`firmware.zip`); flash the dongle/
  half by copying the matching `.uf2` onto it in bootloader mode.
- If you ever need to build locally with `west` (not the normal workflow here), the per-shield READMEs
  document the exact invocation, e.g.:
  `west build -p -b zitaotech_keypoint_left -- -DSHIELD="lpm_view;left_bbtrackpad_keypoint"`
- `.github/workflows/draw.yml` auto-regenerates the keymap diagrams (via `caksoylar/keymap-drawer`) into
  `keymap-drawer/` whenever anything under `config/` changes, and commits the result back.

## Editing the keymap

The only file most changes touch is `config/zitaotech_keypoint.keymap` (devicetree keymap for both halves).
Two ways to edit, per `README.md`:
1. Edit `config/zitaotech_keypoint.keymap` directly and push — CI builds the firmware.
2. Use the web-based [keymap-editor](https://nickcoutsos.github.io/keymap-editor/), which reads/writes
   `config/zitaotech_keypoint.json` (the structured/editable form) in this repo via GitHub login.

Keep `config/zitaotech_keypoint.keymap` and `config/zitaotech_keypoint.json` in sync when hand-editing —
the JSON is the source the web editor round-trips against.

### Keymap structure (`config/zitaotech_keypoint.keymap`)

- Custom `behaviors` block defines: tap-dance bindings (`td0`, `tdRMCLK`), homerow mods (`hm`), a
  tap-preferred hold-tap for mouse click+scroll (`LMouse_sc`), a hold-tap combining mouse press + key tap
  (`M_kp`), and a BLE profile next/prev sensor-rotate (`BLE_encoder`).
- Layers (indices referenced by `mo`/`to` in bindings): `default_layer` (0, QWERTY), `lower_layer` (1),
  `raise_layer` (2), `MOUSE_layer` (3), `RES_layer` (4, reserved/unused — all `&none`).
- `MOUSE_LAYER_ID` is defined in the per-half `.dts` files (`config/boards/arm/zitaotech_keypoint/
  zitaotech_keypoint_{left,right}.dts`) and must match the `MOUSE_layer` index above — it's what the
  trackpad/trackpoint `input_processors` (`zip_temp_layer`) use to momentarily engage layer 3 on input.

## Repo layout

- `config/west.yml` — Zephyr west manifest; pins the `zmk` project/revision this config builds against.
- `config/zitaotech_keypoint.keymap` / `.json` — the keymap, in devicetree and keymap-editor JSON form.
- `config/boards/arm/zitaotech_keypoint/` — custom board definitions for the left/right halves
  (`*_left.dts`, `*_right.dts`, shared `zitaotech_keypoint.dtsi`/`-layouts.dtsi`, Kconfig/defconfig,
  `board.cmake`). This is where MCU pinout, matrix transform, and per-half `input_processors` live.
- `config/boards/shields/` — custom shields stacked on top of each board via `-DSHIELD="a;b;c"`:
  - `lpm_view/` — the display shield (nice!view-compatible panel), with its own LVGL widgets
    (`widgets/`, including custom animated art) and display driver (`display_driver/lpm009m360a.*`).
  - `left_bbtrackpad_keypoint/` — left-half trackpad shield; custom driver in `custom_driver_left/`
    (`a320.c/.h` sensor driver, `trackpad_led.c/.h`).
  - `right_trackpoint_keypoint/` — right-half trackpoint shield; custom driver in
    `custom_driver_right/` (`trackpoint_0x15.c`).
  - `settings_reset/` — standard ZMK settings-reset shield, used to clear BLE bonds (see the commented
    block in `build.yaml` for how it gets added to a build).
  - Each shield directory has a `*.zmk.yml` (ZMK module metadata: id, features, requirements) and a
    `Kconfig.shield` / `Kconfig.defconfig` pair — when adding a new shield, mirror that pattern.
- `keymap-drawer/` — auto-generated keymap diagrams (SVG + intermediate YAML); do not hand-edit, they're
  overwritten by the `draw.yml` CI job.

## Conventions when modifying devicetree / driver code

- New custom behaviors go in the `behaviors { ... }` blocks in the keymap file, following the existing
  hold-tap/tap-dance patterns (each behavior gets its own `behaviors { }` wrapper in this file, matching
  existing style — not consolidated into one block).
- Layer numbers are referenced by raw integer (`mo 1`, `to 3`, etc.) throughout the keymap and in
  `MOUSE_LAYER_ID` — if you reorder or insert a layer, update every numeric reference across
  `zitaotech_keypoint.keymap` and both per-half `.dts` files.
- Custom pointing-device drivers (`a320.c`, `trackpoint_0x15.c`) and their LED counterparts are
  Zephyr sensor drivers wired up through each shield's `.overlay`/`.conf`/`Kconfig.defconfig` — changes to
  sensor behavior typically require touching the driver `.c`/`.h` pair plus the shield's `.conf`/overlay
  together.
