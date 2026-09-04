# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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

