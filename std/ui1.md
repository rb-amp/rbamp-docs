# STD-UI1 — single-phase meter

*Member of the [rbAmp STD family](overview.md); everything on the
[family page](overview.md) applies.*

## What it is

A single-phase precision energy meter module: **one voltage and one current input** with synchronous
sampling, the full STD feature set, and a choice of I²C or Modbus RTU. Variant code `0x11`.

It works as a sub-meter / smart-meter front end for a master (ESP32, Arduino, Raspberry Pi, PLC), or
directly on a Modbus RTU network (through an external RS-485 transceiver) as a meter, including the SDM-compatible profile for inverters and
monitoring systems that read the points of an Eastron-type meter.

---

## Key properties

| | |
|---|---|
| Inputs | 1 voltage, 1 current (factory-matched sensor, labelled for the channel; 60 A or 100 A in the kit) |
| Quantities | U, I, P (signed), Q, S, PF (signed), frequency; import/export energy, active and reactive |
| Energy counters | Wh, mWh, µWh (active) and varh, µvarh (reactive); kept when the power is off |
| Interfaces | I²C (`0x50` by default) or Modbus RTU (unit 1, 9600 8N1 by default), chosen by the interface mode switch |
| Snapshot cadence | a complete snapshot about ten times a second; data-ready output |
| Period snapshots | eight independent banks |
| Update | over the working bus, with automatic rollback |

---

## Measured characteristics

Measured on one production module per kit (60 A and 100 A), each individually calibrated together with its sensor, against a
reference meter on a resistive load. Current is given as the primary current with one pass of the wire
through the sensor.

| Parameter | 60 A kit | 100 A kit |
|---|---|---|
| Current error | ±0.19 % from 1.14 A to 79.85 A | ±0.22 % from 1.13 A to 97.2 A |
| Upper end of the range | the reading saturates near 80 A (−1 % at 82–84 A) | linear up to 128.9 A (±0.23 %) |
| Voltage error | ±0.24 % (measured at 200–224 V) | ±0.14 % (measured at 199–224 V) |
| Power factor, resistive load | 0.999–1.000 | 0.999–1.000 |
| Energy, one 15-minute run (deviation from the reference) | +0.157 % | +0.14 % |
| Reads from | 4.8 mA | 4.8 mA |
| Noise with no current | 0.25 mA | 0.29 mA |

The 100 A figure for the upper end is the limit of the test setup (about 131 A), not a limit of the module.

---

## Typical uses

- The main meter of a single-phase home, apartment or workshop.
- Sub-metering of a circuit, a heater, an EV charger or a heat pump.
- Solar self-consumption: separate import and export energy, SDM points for a Modbus master.
- A measuring node in a Home Assistant / ESPHome installation.

---

## Where to go next

[Hardware](hardware.md) · [Getting started](getting-started.md) ·
[Real-time polling](realtime-polling.md) · [Energy and periods](energy-periods.md) ·
[Technical reference](reference/README.md)
