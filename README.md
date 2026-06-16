# rbAmp Documentation

Official documentation for the **rbAmp** AC energy-monitoring modules (Basic / Standard) — a compact I²C slave for RMS voltage, current, signed active power, and tariff energy.

Hosted version: <https://www.rbamp.com/docs/modules-basic-standard-overview>

## Contents

- [Overview](overview.md)
- [Quick Start](quickstart.md)
- [Hardware Connection](hardware-connection.md)
- [Initialization](initialization.md)
- [Real-time Polling](realtime-polling.md)
- [Period Metering](period-metering.md)
- [API Reference](api-reference.md)
- [Troubleshooting](troubleshooting.md)

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
