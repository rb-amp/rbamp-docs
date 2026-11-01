# rbAmp STD — the precision metering family

**rbAmp STD** is the precision line of rbAmp energy measurement: true sample-by-sample power, signed
(import/export) energy kept on the module, and **one register map and one firmware core for every member
of the family**. A module works **standalone** — as a meter on a Modbus RTU network — or
as a **slave under your master** (ESP32, Arduino, Raspberry Pi, PLC) over I²C or Modbus.

Members differ only in the number of voltage and current channels and in the number of phases. Registers,
commands, energy accounting, firmware update and transports are identical across the family: a client reads
the discovery block and adapts. There is one client library per platform for the whole line.

Member pages:

- [STD-UI1 — single-phase meter](ui1.md)
- [STD-U3I3 — three-phase meter](u3i3.md)

## Family members

| Member | Channels | Phases |
|---|---|---|
| **STD-UI1** — [single-phase meter](ui1.md) | 1 voltage + 1 current | 1 |
| **STD-U3I3** — [three-phase meter](u3i3.md) | 3 voltage + 3 current | 3 |

The planned rbAmp Hub will bring several modules together.

Relation to the **rbAmp BASIC** line ([documentation](../README.md)): BASIC is the economy / monitoring
line with full galvanic isolation; STD is the precision / three-phase line.

---

## Why STD

- **One sensor per channel, no range switching.** Each channel works with its own factory-matched sensor.
- **Power from every measurement point.** Active, reactive and apparent power are computed from every
  measurement point (u·i). All channels and totals come from one coherent measurement window.
- **Energy you can rely on.** Import and export are kept separately, signed, for active and reactive
  energy. Measurement frames lost during an internal memory write are compensated in the energy counters.
  Energy is saved every second: a power cut loses about a second at most.
- **Standalone from first power-up.** Metering starts by itself; energy accumulates and the event log is
  kept without any master.
- **A Modbus meter.** An SDM630/SDM120-compatible float profile publishes per-phase voltage, current and
  power, total power, frequency, and import and export energy at the addresses an inverter or SCADA reads
  from an Eastron-type meter; other points of that meter read 0.
- **Update without a screwdriver.** Firmware is updated over the working bus; a failed image rolls back by
  itself and metering continues during the update. The image is signed and encrypted.
- **Provable authenticity.** Each module carries a unique device certificate and answers a challenge from
  the master with a signed response.

---

## What you can build

- **The main meter of a home, a building, a production floor or a workshop:** whole-site consumption,
  import and export energy, per-phase readings on three-phase sites.
- **A home with solar:** separate import and export energy, the SDM points for a Modbus master,
  a grid-minus-solar balance in Home Assistant.
- **Sub-metering of tenants or workshops:** per-channel measurements and energy for every line.
- **An EV charging site:** three-phase metering for each charger, import and export energy.
- **Integration into an existing control system:** any Modbus master that supports the listed SDM points can read the module.
- **A smart home:** ESPHome / Home Assistant component, MQTT through the master, per-channel load
  monitoring. Over Modbus the listed SDM points can be read by any Modbus master that supports them.

---

## What every STD member does

### Measurement

- All channels are sampled synchronously, about 3 kHz per channel.
- Measurement windows are locked to the mains period (whole cycles); a complete snapshot is published
  about ten times a second.
- Per channel: RMS voltage, RMS current, active power (signed), reactive power, apparent power, power
  factor (signed), mains frequency. Three-phase members add per-phase and grand totals.
- Phase angle of each voltage input and the phase sequence on three-phase members — an installation check
  before you trust per-phase readings.
- **Signed energy:** import and export, active and reactive; counters in Wh, mWh and µWh.
- **Channel ratings.** The module ships with a current sensor for every channel, calibrated together with
  it and labelled per channel. The kit includes a current sensor in one of two ratings: 60 A or 100 A. Do
  not swap sensors between channels.

---

### Energy accounting and data integrity

- A non-decreasing life-time energy register with journaled storage: a damaged record falls back to the
  previous good copy.
- If the storage becomes unavailable the module keeps metering, writes nothing, and recovers by itself
  when the storage returns.
- Event log with before/after values: counter reset or adopt, calibration save, firmware update.
- **Eight independent snapshot banks:** each master freezes the period it needs (a minute, 15 minutes, an
  hour) without disturbing the others. See [Energy and periods](energy-periods.md).
- Time is set by the master; modules keep no battery clock and never invent a time. Counters accumulate
  without it; hourly history records are written once the time is set.

---

### Interfaces

- **I²C** — the full register map (16-bit addressing; address `0x50` by default, configurable).
- **Modbus RTU** — the module is a Modbus meter:
  - UART from 2400 baud up to 1 Mbaud (over RS-485 the line speed is set by the external transceiver), frames 8N1 / 8E1 / 8O1 / 8N2, configurable word order;
  - **three register views:** the native STD map (fixed-scale integers) · an SDM630/SDM120-compatible
    float profile · per-channel virtual meters, where each current channel of a multi-channel member
    answers as its own single-phase meter on its own unit id;
  - a discovery block (type, channel counts, serial number, firmware version);
  - settings writes are validated; reset, calibration and update commands need an explicit unlock.
- I²C or Modbus is chosen at installation with a interface mode switch.
- A data-ready output gives one pulse per new measurement window for masters that poll on availability.

---

### Firmware update

- Over the working bus (I²C or Modbus); no programmer, no removal from the installation.
- Two firmware copies; a failed or hanging new image rolls back automatically.
- A foreign, corrupted or older image is refused. A damaged page is resent alone, not the whole image.
- Metering continues during the update.

---

### Device identity and security

- A unique device certificate; the module signs a challenge from the master.
- Energy and log records in the module's storage are signed by the device: records copied from another
  device or edited outside it are detected and reported.

---

### Service

- Status LEDs: metering heartbeat, update indication, bus activity, fault.
- An IDENTIFY command blinks the LEDs to locate the module in a cabinet.

---

### Software

One client library per platform covers the whole line; channel count and capabilities are discovered at
run time. Arduino, ESP-IDF, ESPHome / Home Assistant and Python are available. Without our libraries the
module works too: over Modbus it is a standard meter, and the register protocol is published in the
[technical reference](reference/README.md).

SDM630, SDM120 and Eastron are names of a third-party manufacturer's meters, used here only to state
compatibility with their Modbus register protocol and interface.

---

## Documentation

| Page | Covers |
|---|---|
| [Hardware](hardware.md) | signals, wiring, interface mode switch |
| [Getting started](getting-started.md) | discovery and the first reads over I²C and Modbus |
| [Real-time polling](realtime-polling.md) | window cadence, DRDY, reading a snapshot |
| [Energy and periods](energy-periods.md) | counters, hourly history, event log, snapshot banks |
| [Troubleshooting](troubleshooting.md) | symptoms and checks |
| [Examples](examples.md) | ready sketches and scripts |
| [Technical reference](reference/README.md) | every register, command and transport rule |

---

## Enclosure and installation

The enclosure has a DIN clip and can be installed in a cabinet (DIN rail). The same form factor across STD modules.

The enclosure covers the lower part of the board, where the AC circuits are. The upper part of the board with the interface connector (VIN, GND, bus) is open; it is galvanically isolated from the AC part. The only terminal that comes out of the enclosure is the AC voltage-sensor terminal, in a plastic housing.

> **⚠ Safety.** The measurement inputs connect to mains. Install the module in a closed electrical cabinet,
> by qualified personnel only.
