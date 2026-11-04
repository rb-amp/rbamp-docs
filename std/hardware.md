# 03 · STD hardware and wiring

Signals an installer or integrator deals with on an STD module (STD-UI1, STD-U3I3). The full signal
rules are in the [hardware reference](reference/20_hardware.md); this page is the practical summary.

The enclosure covers the lower part of the board, where the AC circuits are. The upper part of the board with the interface connector (VIN, GND, bus) is open; it is galvanically isolated from the AC part. The only terminal that comes out of the enclosure is the AC voltage-sensor terminal, in a plastic housing.

> **⚠ Safety.** The measurement inputs connect to mains. Install the module in a closed electrical cabinet.
> For work inside an electrical panel, call a qualified electrician.

## Signal set

| Signal | Direction | Purpose |
|---|---|---|
| Bus: pins 3 and 4 — SDA/SCL (I²C) **or** UART TTL for Modbus (pin 3 = module RX, pin 4 = module TX) | bidirectional | I²C slave or Modbus RTU — one per boot, selected by the interface mode switch |
| Interface mode switch | input | selects the transport at boot |
| DE | output | driver enable for an external RS-485 transceiver |
| DRDY | output, open-drain | one pulse per new measurement window; alarm level on a masked event |
| RST | input | reset input of the module |
| Status LEDs ×2 | — | green run / red communication fault |
| Current inputs, per channel | input | the factory-matched sensor, labelled for the channel |
| Voltage inputs | input | one per voltage channel (AC, from the mains) |

---

## Voltage connection

The AC voltage input uses the same wire and fuse rule as BASIC: a thick wire is not required, AWG26–22 is
enough; a fuse in the L line is **mandatory**, rated up to 0.1 A.

- **STD-UI1:** one voltage input.
- **STD-U3I3:** the voltage terminal block has four positions: **N, L1, L2, L3**. The module measures
  phase-to-neutral voltage, so the **neutral is required**. Fuse each L line you connect (up to 0.1 A).

---

## Sensors

Every current channel ships with its own sensor, calibrated together with the channel and labelled with
it. **Connect each sensor to the channel it is labelled for.** Moving a sensor to another channel, or
connecting another sensor, loses its factory calibration; the module does not report it. This also applies when you rewire the panel later. The sensors and the board are marked and the kit comes pre-connected; for the import direction follow the arrow marking on the sensor and the board.

---

## Connector

The module has one **JST XH 7-position connector, 2.54 mm pitch**. Its signals are:

| Pin | Signal (as printed) | Purpose |
|:---:|---|---|
| 1 | VIN | supply input, 4.5–5.5 V; 50 mA maximum, 30 mA typical |
| 2 | GND | ground |
| 3 | SDA / TX | I²C data; in Modbus mode the module's UART RX — connect the master's TX |
| 4 | SCL / RX | I²C clock; in Modbus mode the module's UART TX — connect the master's RX |
| 5 | DRDY | data-ready output ([real-time polling](realtime-polling.md)) |
| 6 | DE | driver enable for an external RS-485 transceiver |
| 7 | RST | reset input of the module, for a hardware reset by the master; may be left unconnected |

SDA and SCL are shared with the Modbus UART lines, so one connector serves both transports.

---

## Choosing the transport

A **3-position interface mode switch** on the board selects the transport **at boot**:

- **Modbus RTU** — Modbus RTU with the saved serial settings;
- **Modbus default** — Modbus RTU at **9600 8N1, unit id 1** (use it to reach a module whose serial settings
  are unknown);
- **I²C** — the module is an I²C slave at `0x50`.

The switch is read **only at start-up**, so a change applies after a restart (power cycle or RST).

In a Modbus mode pins 3 and 4 are the module's UART, not I²C: connect a module set to Modbus to a Modbus line only.

---

## Sharing a bus with BASIC modules

BASIC and STD modules both answer at `0x50` by default. Change the address of one of them before putting both on one bus.

---

## I²C pull-ups

The module has 4.7 kΩ I²C pull-ups to 3.3 V, enabled from the factory. To disable them, cut the two solder jumpers on the PCB. On a bus up to 15 cm with two or three modules, leave them as they are; with four or more modules, cut them on all but one (see the table in the BASIC wiring guide). The connector logic is 3.3 V; VIN (5 V) feeds only the on-board regulator.

---

## I²C wiring

- Open-drain bus; the pull-ups are on the module (see *I²C pull-ups*) or on the master. Recommended bus speed: 50 kHz on ESP32 and ESP8266 hosts (a limitation of the ESP-IDF I²C driver, not of the module), 100 kHz on other hosts; do not use 400 kHz. The speed is the user's choice.
- Set the master's SCL timeout to at least 10 ms: during firmware-update flash writes the module may
  stretch the clock for several milliseconds.
- DRDY is an open-drain output without a pull-up on the module; when you use DRDY, fit a pull-up on the master side — 10 kΩ to the master's 3.3 V logic supply is recommended. It idles high, a short low pulse for each new measurement window, held low on an
  alarm. Pair each pulse with the module's `WIN_SEQ`.
- To remove a module from a shared bus, disconnect its bus wires (or the whole module), not only its power: an unpowered module loads SDA and SCL and can stop the bus for the other modules.

---

## RS-485 wiring

- The module has no RS-485 transceiver: the connector carries UART TTL on pins 3 (module RX) and 4 (module TX), plus DE
  for an external transceiver. Termination and bias are external.
- If your RS-485 adapter needs turnaround time, set `CFG_TX_DELAY` (0…100 ms response hold-off).
- Out of the box: unit 1, 9600 8N1. Change the serial settings and unit id through the configuration
  registers; see [Modbus transport](reference/12_transport_modbus.md#serial).

---

## RST

RST is the reset input of the module, for a hardware reset by the master; it may be left unconnected. Use it to recover a silent module or to apply an interface-switch
change without a power cycle. A reset loses the state your master owns in RAM (event mask, thresholds, snapshot arming, clock);
re-arm per the [diagnostics checklist](reference/19_diagnostics.md#health-checklist-for-a-supervising-master).

---

## Status LEDs

| LED | Pattern | Meaning |
|---|---|---|
| green | short flash every ~2 s | metering runs |
| green | double flash | first ~8 s after boot, or an update session is open |
| red | short blink | a request addressed to this module |
| red | steady | fault: no measurement frames or energy storage fault |
| both | alternating ~5 Hz for 10 s | IDENTIFY — locating the module |

---

## Enclosure

The enclosure has a DIN clip and can be installed in a cabinet (DIN rail).
