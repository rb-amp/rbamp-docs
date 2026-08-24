# rbAmp Documentation

Official documentation for the **rbAmp** AC energy-monitoring modules (Basic / Standard) — a compact I²C slave for RMS voltage, current, signed active power, and tariff energy.

Hosted version: <https://www.rbamp.com/docs/modules-basic-standard-overview>

**Safe by design.** rbAmp is entirely low-voltage — the current sensor (CT) clamps around the outside of the conductor with no electrical contact, and the only mains reference (on voltage-sensing variants) is a galvanically isolated tap. The side you wire to your ESP32, Arduino or Raspberry Pi is never at mains potential. See [Safety — Safe by Design](safety.md).

## Contents

- [Overview](overview.md)
- [Quick Start](quickstart.md)
- [Hardware Connection](hardware-connection.md)
- [Initialization](initialization.md)
- [Real-time Polling](realtime-polling.md)
- [Period Metering](period-metering.md)
- [API Reference](api-reference.md)
- [Troubleshooting](troubleshooting.md)
- [Safety — Safe by Design](safety.md)

## Raw-register examples

Worked examples that drive the rbAmp I²C protocol directly (no client library), per platform:

- [Arduino](arduino-examples.md)
- [MicroPython & CircuitPython](micropython-examples.md)
- [ESP-IDF](esp-idf-examples.md)
- [STM32 HAL](stm32-hal-examples.md)
- [Raspberry Pi Pico SDK](pico-sdk-examples.md)
- [Python on Linux SBC](python-sbc-examples.md)

## Client libraries

Drop-in libraries that speak the rbAmp I²C protocol:

- [Arduino](https://github.com/rb-amp/rbamp-arduino) — AVR / ESP32 / ESP8266 / STM32duino
- [ESP-IDF component](https://github.com/rb-amp/rbamp-esp-idf) — native C, IDF ≥ 5.2
- [Python](https://github.com/rb-amp/rbamp-python) — CPython (smbus2) + MicroPython
- [ESPHome external component](https://github.com/rb-amp/rbamp-esphome) — YAML / Home Assistant

## License

See [LICENSE](LICENSE).
