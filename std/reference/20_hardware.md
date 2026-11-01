# Hardware interface

Signals a host or installer deals with; this chapter defines the signals and their rules.

The enclosure covers the lower part of the board, where the AC circuits are. The upper part of the board with the interface connector (VIN, GND, bus) is open; it is galvanically isolated from the AC part. The only terminal that comes out of the enclosure is the AC voltage-sensor terminal, in a plastic housing.

> ⚠ **Safety.** The measurement inputs connect to mains. Installation in a closed electrical cabinet by
> qualified personnel only.

## Signal set

| Signal | Direction | Purpose |
|---|---|---|
| Bus: pins 3 and 4 — SDA/SCL (I²C) **or** UART TTL for Modbus (pin 3 = module RX, pin 4 = module TX) | bidir | I²C slave or Modbus RTU — one per boot, selected by the interface mode switch |
| Interface mode switch | input | selects the transport at boot (below) |
| DE | output | driver enable for an external RS-485 transceiver |
| DRDY | output, open-drain | data-ready pulse / alarm ([DRDY](15_drdy_events.md)) |
| RST | input | reset input of the module (below) |
| Status LEDs ×2 | — | green run / red comm-fault ([Diagnostics](19_diagnostics.md#status-leds)) |
| Current-sensor inputs, per channel | input | factory-matched sensors, labelled per channel |
| Voltage inputs | input | per voltage channel; STD-U3I3: four-position terminal N, L1, L2, L3 (neutral required) |

---

## Interface mode switch

A 3-position switch selects the transport **at boot** (a change applies after a restart):

- **Modbus RTU** — Modbus RTU with the saved serial settings;
- **Modbus default** — Modbus RTU at 9600 8N1, unit id 1;
- **I²C** — I²C slave.

The switch is read only at start-up.

The module has a JST XH 7-position connector (2.54 mm), pin 1 → 7: VIN (4.5–5.5 V; 50 mA maximum, 30 mA typical), GND, SDA/TX, SCL/RX (names as printed on the board),
DRDY, DE (driver enable for an external RS-485 transceiver), RST (reset input of the module, for a hardware reset by the master; may be left unconnected). SDA/SCL are shared with
the Modbus UART lines: in Modbus mode pin 3 (SDA) is the module's UART RX and pin 4 (SCL) is the module's UART TX — connect the master's TX to pin 3 and the master's RX to pin 4. Two PCB jumpers enable 4.7 kΩ I²C pull-ups to 3.3 V; the connector logic is 3.3 V and VIN feeds only the on-board regulator.

---

## I²C wiring

- Standard open-drain bus; master provides pull-ups. Recommended bus speed: 50 kHz on ESP32 and ESP8266 hosts (a limitation of the ESP-IDF I²C driver, not of the module), 100 kHz on other hosts; do not use 400 kHz. The speed is the user's choice.
- Budget the master's SCL timeout for ≥ 10 ms of clock stretching (worst case during firmware-update
  flash writes) — see [I²C transport](11_transport_i2c.md#timing-and-robustness).
- DRDY is an open-drain output without a pull-up on the module; when you use DRDY, fit a pull-up on the master side — 10 kΩ to the master's 3.3 V logic supply is recommended. It is a separate line (idle high, a short low pulse per new window, held low on alarm), paired with the module's `WIN_SEQ`.

---

## RS-485 wiring

- The module has no RS-485 transceiver: the connector carries UART TTL on pins 3 (module RX) and 4 (module TX), plus DE for an
  external transceiver. Termination and bias are external.
- If your RS-485 adapter needs turnaround time, use `CFG_TX_DELAY` (0..100 ms response hold-off).
- Defaults out of the box: unit 1, 9600 8N1.

---

## RST

- RST is the reset input of the module, for a hardware reset by the master; it may be left unconnected.
- Uses: recover a silent module; apply an interface-switch change without a power cycle.
- Resetting loses the RAM state a master owns (event mask, thresholds, snapshot arming, clock) — the
  master re-arms per the [diagnostics checklist](19_diagnostics.md#health-checklist-for-a-supervising-master).

---

## Enclosure

The enclosure has a DIN clip and can be installed in a cabinet (DIN rail). The same form factor across STD modules and the planned hub.
