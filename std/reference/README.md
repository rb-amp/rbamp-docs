# rbAmp STD — technical reference

Full technical reference for the **rbAmp STD module line**: register map, transports, commands,
measurement and energy model, firmware update, diagnostics and host integration.

> Figures and behaviour reflect firmware image `0x0001007C`, register-map version
> `0x36` (differences of older images are flagged inline as "before 0x79" / "image 0x7A+" /
> "image 0x7B+" / "image 0x7C+" — 0x7C changes the measurement window to ~100 ms; no register changes).

This reference describes the **current firmware core** shared by every STD module. Built and
validated member today: **STD-UI1** (1 voltage + 1 current channel). Further members (more
channels, three-phase) run the same map — they
differ only in `N_VOLTAGE` / `N_CURRENT` and the variant code, so everything here is written
generically by channel count. 

## Contents

| # | File | Covers |
|---|---|---|
| 1 | [Identity & discovery](10_identity_discovery.md) | magic, versions, capabilities, variant codes, channel counts, UID, board id |
| 2 | [I²C transport](11_transport_i2c.md) | 16-bit pointer, read/write sequences, clock stretching, address change |
| 3 | [Modbus RTU transport](12_transport_modbus.md) | framing, profiles, register spaces, exceptions, UNLOCK, behaviour rules |
| 4 | [Register map](13_register_map.md) | every block with address, type, units, scale, access |
| 5 | [Commands](14_commands.md) | command register, busy/result semantics, error codes, gating |
| 6 | [Data-ready output & events](15_drdy_events.md) | DRDY contract, WIN_SEQ, event flags, thresholds, alarm |
| 7 | [Measurement & energy model](16_measurement_energy.md) | windows, units, energy counters, history, event log, time |
| 8 | [Configuration, persistence & calibration](17_config_calibration.md) | what survives reset/power-off, how to save, user-visible calibration |
| 9 | [Firmware update](18_firmware_update.md) | host-side OTA protocol over I²C and Modbus |
| 10 | [Diagnostics](19_diagnostics.md) | status flags, reset cause, counters, health checklist, LEDs |
| 11 | Hardware interface | signals, wiring, transport jumper, DRDY, reset |
| 12 | [Host integration](21_integration.md) | raw register sequences without our libraries; library pointers |
| 13 | [Sequences](22_sequences.md) | canonical step-by-step flows: init, commands, config, address change, energy, history, OTA, unlock |

## Conventions used throughout

- **Addresses.** I²C register addresses are 16-bit, written `0xNNNN`. Modbus register addresses are
  a **separate namespace** (see [Modbus transport](12_transport_modbus.md)); a Modbus address never
  equals "the I²C address of the same value" — mapping tables are given explicitly.
- **Byte order.** Multi-byte values on I²C are **little-endian**. On Modbus the native profile is
  big-endian word order **ABCD** (configurable), the SDM profile is always ABCD.
- **Units** (fixed-scale integers, never float, on I²C and the Modbus native profile):

  | Quantity | Unit | To SI |
  |---|---|---|
  | Voltage | mV | ÷1000 → V |
  | Current | µA | ÷1 000 000 → A |
  | Active power | mW (signed) | ÷1000 → W |
  | Reactive power | mvar (signed) | ÷1000 → var |
  | Apparent power | mVA | ÷1000 → VA |
  | Power factor | Q10 (signed) | ÷1024 → PF |
  | Frequency | cHz | ÷100 → Hz |
  | Cumulative energy | Wh / varh (i64, signed) and mWh (i64); µWh (i64) on image 0x7A+, µvarh (i64) on image 0x7B+ | — |
  | Voltage phase angle | centi-degree (i16, signed, image 0x7A+) | ÷100 → ° |
  | Per-window energy | mW·samples (i64, signed) | see measurement model |
  | THD | centi-percent | ÷100 → % |

- **Signs.** P > 0 = import (consumption); export is negative. Q follows IEC 62053-23
  (Q > 0 inductive). PF carries the sign of P. Import/export energy is kept in separate counters.
- **Sentinels — per transport.** A field a given build does not offer reads, on the **Modbus native
  profile**, a sentinel: `0x8000` (i16), `0x80000000` (i32), `0xFFFF` (u16); in the **SDM profile**
  it reads 0; on **I²C** such fields simply read 0, and unmapped I²C addresses read `0xFF`.
- **Gating rule.** A master must gate channel access on `N_VOLTAGE` / `N_CURRENT` and feature access
  on `CAPABILITIES` — registers of channels ≥ N and of absent features are not data.
