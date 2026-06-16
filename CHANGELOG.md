# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial core product docs (8 pages): overview, quickstart, hardware connection, initialization, real-time polling, period metering, API reference, troubleshooting.
- Raw-register example chapters (6): Arduino, MicroPython/CircuitPython, ESP-IDF, STM32 HAL, Raspberry Pi Pico SDK, Python on Linux SBC.
- Diagrams and infographics across the docs: wiring schematic, register memory map, atomic period-latch timing, multi-module strategies, troubleshooting flowchart, and more.

### Changed
- Core docs updated to wire-protocol **v1.3**: register map, atomic period latch (one write closes + opens), General-Call broadcast latch, two-phase address commit, READ auto-increment / WRITE byte-at-a-time asymmetry, not-sticky `REG_ERROR` model, AC_FREQ on all variants, full CT-model code table.

