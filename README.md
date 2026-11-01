# rbAmp — User Documentation

Hosted version: <https://www.rbamp.com/docs/modules-basic-standard-overview> (BASIC) · <https://www.rbamp.com/docs/modules-std-overview> (STD)

**rbAmp** modules measure AC mains parameters — voltage, current, active and reactive power, power factor, line frequency and energy — and report them to a host over I²C (and, on STD, Modbus RTU). The documentation is organised by product line; each line has its own complete set of pages, including its own I²C description, because the register maps and commands differ.

> ⚠ **Mains safety**: connecting the measurement inputs and fitting a current sensor on a panel conductor must be done by a qualified person with the circuit de-energised — see the hardware page of the line you use and [Safety — Safe by Design](safety.md).

## Product lines

| Line | What it is | Start here |
|---|---|---|
| **rbAmp BASIC** | single-phase modules for SCT-013 type sensors and factory-paired CT clamps; I²C slave, 8-bit register map | [BASIC documentation](overview.md) |
| **rbAmp STD** | single-phase (STD-UI1) and three-phase (STD-U3I3) modules; on-module energy counters, I²C 16-bit register map and Modbus RTU | [STD documentation](std/overview.md) |

Both lines are read by the same client libraries; `begin()` detects which line is connected.

---

## BASIC pages

| Page | Covers |
|---|---|
| [Overview](overview.md) | variants, what the module measures, SCT-013 characteristics |
| [Hardware](hardware-connection.md) | wiring, I²C pull-ups, bus speed and length, sensors |
| [Initialization](initialization.md) | first start, sensor model, address change |
| [Real-time polling](realtime-polling.md) | reading values, data-ready pin |
| [Period metering](period-metering.md) | energy from period snapshots |
| [Troubleshooting](troubleshooting.md) | symptoms and checks |
| Examples | [Arduino](arduino-examples.md) · [MicroPython](micropython-examples.md) · [ESP-IDF](esp-idf-examples.md) · [STM32 HAL](stm32-hal-examples.md) · [Pico SDK](pico-sdk-examples.md) · [Python on Linux SBC](python-sbc-examples.md) — raw-register, no client library |
| [API reference](api-reference.md) | every register and command |

---

## STD pages

| Page | Covers |
|---|---|
| [Family overview](std/overview.md) | members, what every STD module does |
| [STD-UI1](std/ui1.md) · [STD-U3I3](std/u3i3.md) | the two members |
| [Hardware](std/hardware.md) | connector, interface mode switch, wiring |
| [Getting started](std/getting-started.md) | discovery and the first reads over I²C and Modbus |
| [Real-time polling](std/realtime-polling.md) | window cadence, DRDY |
| [Energy and periods](std/energy-periods.md) | counters, hourly history, snapshot banks |
| [Troubleshooting](std/troubleshooting.md) | symptoms and checks |
| [Examples](std/examples.md) | ready sketches and scripts |
| [Technical reference](std/reference/README.md) | every register, command and transport rule |

---

## Client libraries

| Platform | Repository |
|---|---|
| Arduino | [rbamp-arduino](https://github.com/rb-amp/rbamp-arduino) |
| ESP-IDF | [rbamp-esp-idf](https://github.com/rb-amp/rbamp-esp-idf) |
| ESPHome / Home Assistant | [rbamp-esphome](https://github.com/rb-amp/rbamp-esphome) |
| Python | [rbamp-python](https://github.com/rb-amp/rbamp-python) |

---

## License

See [LICENSE](LICENSE).
