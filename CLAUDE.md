# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ZMK firmware configuration for the [beekeeb Toucan Keyboard](https://beekeeb.com/toucan-keyboard/) — a wireless split 42-key column-stagger board with a Cirque trackpad and nice!view display. Built on top of ZMK v0.3 with several external modules.

Hardware: Seeeduino XIAO BLE (nRF52840) on each half. Left half has the nice!view display. Right half has the Cirque Pinnacle trackpad over SPI.

## Build / Flash

Firmware is built via GitHub Actions (`.github/workflows/build.yml`). The build matrix is defined in `build.yaml`. There is no local build setup — push to trigger CI, or use the ZMK GitHub Actions workflow manually via `workflow_dispatch`.

To add a new build target, edit `build.yaml`.

## Repository Structure

```
config/
  toucan.keymap       # User keymap (edit this to remap keys)
  west.yml            # ZMK + external module dependencies
  toucan.json         # ZMK Studio layout metadata

boards/shields/toucan/
  toucan.dtsi         # Shared hardware definition: kscan matrix, physical layout, trackpad split input
  toucan_left.overlay # Left half: nice!view SPI, column GPIOs
  toucan_right.overlay# Right half: Cirque trackpad SPI, column GPIOs, row GPIOs

boards/shields/nice_view_gem/
  custom_status_screen.c  # Entry point for display; instantiates screen widget
  widgets/screen.c        # Main display widget — handles all ZMK event subscriptions and draw calls
  widgets/{battery,layer,output,profile,sleep}.*  # Individual display widgets
  assets/                 # Fonts (QuinqueFive) and images embedded as C arrays
```

## External Modules (west.yml)

| Module | Purpose |
|--------|---------|
| `zmkfirmware/zmk@v0.3` | ZMK core |
| `geeksville/cirque-input-module@toucan` | Cirque Pinnacle trackpad driver |
| `caksoylar/zmk-rgbled-widget@v0.3` | RGB LED status widget |

## Keymap Architecture

`config/toucan.keymap` defines the active keymap. `boards/shields/toucan/toucan.keymap` is the shield default (not used when a user keymap exists).

Layers: BASE (0), NAV (1), SYM (2), ADJ (3). ADJ activates via tri-layer when NAV+SYM are both held simultaneously.

`studio-rpc-usb-uart` snippet + `CONFIG_ZMK_STUDIO=y` is enabled on the left half only, allowing keymap editing via ZMK Studio over USB without reflashing.

## Display (nice_view_gem)

The display shield is a fork of [nice-view-gem](https://github.com/M165437/nice-view-gem). The main draw loop is in `widgets/screen.c:draw_top()`. Each widget (battery, layer, output, profile, sleep) has its own `.c`/`.h` pair under `widgets/`. The sleep screen takes over the full display when ZMK enters `ZMK_ACTIVITY_SLEEP` state; `lv_task_handler()` + `lv_refr_now()` are called explicitly to flush the frame before deep sleep.

## Trackpad

The Cirque Pinnacle on the right half is connected via SPI and driven by the `cirque-input-module`. The split input proxy (`glidepoint_split` / `glidepoint_listener`) forwards touch events to the left (central) half over BLE. On NAV and SYM layers, `zip_xy_to_scroll_mapper` converts pointer movement to scroll events. Trackpad sleep mode is available but disabled by default in `toucan_right.overlay` due to a ~300ms wake delay.
