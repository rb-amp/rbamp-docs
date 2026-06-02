<!-- DRAFT placeholder. Full content TBD by writing skill (technical-writer-en) in Week 0-1.
     Sheff approves before promotion to status: approved. -->


# Quick Start

## What you'll achieve in 15 minutes

By the end of this guide, you'll have an rbAmp module reading voltage and current on your microcontroller, with values flowing into Home Assistant (via ESPHome) or your own Arduino sketch.

## What you need

- 1× rbAmp module (e.g., **Basic UI1** with SCT-013 CT clamp)
- 1× ESP32 dev board (or Arduino Uno, or Raspberry Pi)
- 4× jumper wires (VCC, GND, SDA, SCL)
- 1× SCT-013-100A current clamp (included or sold separately)
- USB cable for programming

:::warning
**Working with mains voltage is dangerous.** Always disconnect AC power before installing voltage taps. If unsure, hire a qualified electrician.
:::

## Wiring

[TBD: schematic diagram with VCC/GND/SDA/SCL connections]

## Choose your path

:::choose-your-path
- **ESPHome (Home Assistant):** [Jump to ESPHome setup ↓](#esphome-setup)
- **Arduino IDE:** [Jump to Arduino setup ↓](#arduino-setup)
- **Raspberry Pi (Python):** [Jump to RPi setup ↓](#raspberry-pi-setup)
:::

## ESPHome setup

[TBD: minimal YAML config, test command, expected output]

## Arduino setup

[TBD: library install, sketch, serial monitor output]

## Raspberry Pi setup

[TBD: smbus2 install, Python script, expected output]

## Verify

After setup, you should see:
- Voltage: ~230 V (or ~120 V depending on region)
- Current: 0 A initially (no load on CT clamp)
- When you turn on a known appliance (e.g., 100W lamp): current spikes to ~0.4 A

## Next steps

- Browse [Hardware Connection guide](hardware-connection.md) for permanent installation
- Read [API Reference](api-reference.md) для register-level details
- Check [Tutorials](https://www.rbamp.com/docs/modules-basic-standard-tutorials/solar-self-consumption) for solar self-consumption setup

## Troubleshooting quick fixes

If you see all zeros or NAK errors:
1. Check I²C wiring (SDA/SCL not swapped)
2. Verify pull-ups (4.7 kΩ on SDA and SCL)
3. Run I²C scanner to confirm address `0x50`

For deeper troubleshooting see [FAQ — Troubleshooting](/faq/troubleshooting).


---

[← Overview](overview.md) | [Contents](README.md) | [Hardware Connection →](hardware-connection.md)
