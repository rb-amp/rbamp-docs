# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## 2026-11-01

### Added
- **STD user documentation** (`std/`): family overview, STD-UI1 and STD-U3I3 pages with measured characteristics for the 60 A and 100 A kits, hardware and wiring, getting started over I²C and Modbus RTU, real-time polling, energy and periods (counters, hourly history, snapshot banks), troubleshooting, examples.
- **STD technical reference**: hardware interface chapter (`std/reference/20_hardware.md`).
- **BASIC CT series** (CT-UI1 / UI2 / UI3 / UI5 / UI7, factory-paired CT clamps) in the overview and hardware pages.

### Changed
- README: one index for both lines (BASIC and STD), raw-register examples and client libraries.
- Diagrams replace the remaining ASCII drawings in the BASIC pages.
- Wording: tariff-accounting and metering terms unified across the BASIC pages.

## 2026-10-23

### Added
- **STD technical reference** (`std/reference/`): register map, transports, commands, measurement and energy model, firmware update, diagnostics and host integration for the STANDARD (STD) module line.
- **Illustrations**: 22 diagrams (PNG + SVG) — module lines, wiring, I²C pull-up rules, timing, period latch, energy accounting, register map, troubleshooting.
- Current-channel accuracy table for SCT-013 sensors (all seven ratings) in the overview.

### Changed
- Measurement window is ~100 ms (images before v1.2 and STD images up to `0x7B`: 200 ms).
- Two module lines in the documentation: BASIC and STANDARD (STD).
- I²C pull-ups: recommended external values by number of modules (connect them to 3.3 V, not to the 5 V supply).
- SCT-013 model codes 1..7 (including -100 and -060) documented as supported.
- Community files are English only; contact address is info@rbamp.com. The draft quick-start page was removed.

### Added
- **Channel access window** (`§4.14`): full documentation for
  `REG_CHANNEL_SELECT` (`0x32`) + `REG_CHANNEL_DATA` (`0x3C..0x3F`) — field
  codes table, four-trap callout, wire-level examples (single-channel read,
  full-module sweep combining flat + window paths).
- **Field 15 `CHFIELD_CT_MODEL`**: per-channel CT model read/write via channel
  window on firmware ≥ v1.4.18. Persistence via `CMD_SAVE_USER_CONFIG` (ungated).
- **SEAL / `ERR_CLONE` forward-compat notes**: capability bit 6 and error
  `0xF9` declared as reserved — mechanism is not implemented in v1.4.x firmware.
- **CAPTURE register refinement**: `CAPTURE_PAGE` page range 0..15 (was 0..7),
  `CAPTURE_WINDOW` 32-byte burst (was 64), `LUT_BYPASS` diagnostic register.
- **Flat-RMS ⛔ STOP trap** in register table: explicit warning that `0x9A` is
  `V03_I0_PEAK`, not `V03_I3_RMS` — channels 3+ live only in the window.
- **Troubleshooting**: "Confirm every CT-model write by read-back" callout —
  silent partial state on `ERR_PARAM` rejection.

### Changed
- `REG_CHANNEL_SELECT` nibble range expanded: channel nibble documented as
  **0..15** (up to 16 channels reserved), validated against `channels()`.

## [0.1.0] — 2026-09-04

### Added
- Initial core product docs (8 pages): overview, quickstart, hardware connection, initialization, real-time polling, period metering, API reference, troubleshooting.
- Raw-register example chapters (6): Arduino, MicroPython/CircuitPython, ESP-IDF, STM32 HAL, Raspberry Pi Pico SDK, Python on Linux SBC.
- Diagrams and infographics across the docs: wiring schematic, register memory map, atomic period-latch timing, multi-module strategies, troubleshooting flowchart, and more.

### Changed
- Core docs updated to wire-protocol **v1.3**: register map, atomic period latch (one write closes + opens), General-Call broadcast latch, two-phase address commit, READ auto-increment / WRITE byte-at-a-time asymmetry, not-sticky `REG_ERROR` model, AC_FREQ on all variants, full CT-model code table.

