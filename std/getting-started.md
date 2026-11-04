# 04 · Getting started with STD

The first reads from an STD module over I²C and over Modbus RTU. Register addresses are I²C 16-bit
pointers; the Modbus addresses are given in brackets. Full flows are in the
[sequences reference](reference/22_sequences.md).

## 0. Select the transport

Set the 3-position interface mode switch before power-up: **Modbus RTU** (saved serial settings),
**Modbus default** (Modbus RTU at 9600 8N1, unit id 1) or **I²C**. The switch is read only at start-up, so a change applies after a restart. See
[Hardware](hardware.md#choosing-the-transport).

---

## 1. Identify the module

Read **12 bytes at 0x0000 in one transaction** (I²C: write the 16-bit pointer `0x0000`, then read 12 bytes):

| Offset | Field | Expect |
|---|---|---|
| 0x0000 | `ID` (2 bytes) | `'S'`, `'T'` |
| 0x0002 | `ERROR` | result of the last command |
| 0x0003 | `VERSION` | `0x36` or higher |
| 0x0004 | `CAPABILITIES` (u16) | bit 9 (16-bit addressing) set; bit 8 (snapshot banks) set when the module has them |
| 0x0006 | `STATUS` | bit 1 sensor ok, bit 2 mains ok |
| 0x0007 / 0x0008 | `N_VOLTAGE` / `N_CURRENT` | STD-UI1: 1 / 1 · STD-U3I3: 3 / 3 |
| 0x0009 | `VARIANT` | `0x11` STD-UI1 · `0x63` STD-U3I3 |
| 0x000A | `HW_REV` | hardware revision |

The exact byte layout is in [Identity & discovery](reference/10_identity_discovery.md). Decide what the
module can do from `N_VOLTAGE`, `N_CURRENT` and the `CAPABILITIES` bits — never from hardcoded variant
codes.

Then read the unique `UID` (0x00F4, 12 bytes) for your database and the firmware version `FW_VER`
(0x0114, u32): the high word and the low word in decimal give the version, for example `0x0001007E` = STD firmware 1.126.

---

## 2. Wait for live data

After power-up allow about one second: `WIN_SEQ` (0x0016) must be non-zero and stepping, and
`STATUS` bit 1 (sensor ok) set. See [boot readiness](reference/16_measurement_energy.md#boot-readiness).

---

## 3. First reads (I²C)

Read the realtime blocks in one transaction each (up to 128 bytes, coherent):

- voltage channel *v*: base `0x0200 + v·0x20` — RMS voltage;
- current channel *c*: base `0x1000 + c·0x40` — current, P, Q, S, PF;
- totals: `0x0300 + k·0x40` — `k = 0` grand total, `k = 1..3` per phase;
- frequency `FREQ` (0x0010, cHz).

The complete block layouts are in the [register map](reference/13_register_map.md). Values are fixed-scale
integers: voltage in mV, current in µA, power in mW / mvar / mVA.

---

## 4. First reads (Modbus RTU)

Defaults: unit 1, 9600 8N1. The discovery block is readable as input registers from `0x6000` (magic,
map version, capabilities, channel counts, variant, UID). Live values are in the native profile (input
`0x6000+`) or, for an inverter or SCADA, in the float32
profile (input `0x0000`: voltage, current and power per phase, total power, frequency, import and
export energy). Details: [Modbus transport](reference/12_transport_modbus.md).

---

## 5. Set the time (optional but recommended)

Modules keep no battery clock. Counters accumulate without it; hourly history records are written only after the time is set. Set the time by having the
master write the epoch to `TIME_SET` (0x08C0) and issues command `0x06`. Do it at discovery and whenever
`STATUS` bit 4 (time valid) reads 0 after a restart.

---

## In the client libraries

The libraries do all of the above in `begin()` and expose the same read calls on STD and BASIC modules; see
[Examples](examples.md). On a module with snapshot banks and energy counters use Arduino [1.6.2 or later](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#energy-source-and-minimum-library-version) or ESP-IDF [1.7.3 or later](https://github.com/rb-amp/rbamp-esp-idf#std-line-minimum-version).
