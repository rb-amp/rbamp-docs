# 08 · STD examples

Minimum library versions: Arduino [1.6.2](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#energy-source-and-minimum-library-version) and ESP-IDF [1.7.3](https://github.com/rb-amp/rbamp-esp-idf#std-line-minimum-version) on a module with snapshot banks and energy counters; Python 1.6.0; ESPHome 1.9.0. Snapshot-bank calls need Python 1.6.1 and ESPHome 1.9.1. Snapshot banks in the libraries: [Arduino](https://github.com/rb-amp/rbamp-arduino/blob/main/docs/09_api_reference.md#period-snapshot-banks), [ESP-IDF](https://github.com/rb-amp/rbamp-esp-idf#std-snapshot-banks), [ESPHome](https://github.com/rb-amp/rbamp-esphome/blob/main/docs/09_api_reference.md#period_banks) (`period_banks:`); Python has the `std_arm_banks`, `std_latch`, `std_read_latch`, `std_latch_wh` calls.

The client libraries read STD and BASIC modules with the same calls; the STD-only quantities (reactive and
apparent power, health counters, snapshot banks) are available behind a line check.

| Platform | Example | Shows |
|---|---|---|
| Arduino | [rbamp-arduino](https://github.com/rb-amp/rbamp-arduino), `examples/11_StdLine` | one sketch for both lines: `begin()` detects the line; realtime values; health counters; a master-timed period on snapshot bank 1 |
| ESP-IDF | [rbamp-esp-idf](https://github.com/rb-amp/rbamp-esp-idf), `examples/std_meter` | STD meter readout with the ESP-IDF component |
| Python (CPython) | [rbamp-python](https://github.com/rb-amp/rbamp-python), the package examples | the same reads on Raspberry Pi and servers |

## Arduino: the core of the sketch

(`Serial.printf` as on the ESP32 Arduino core; use `Serial.print` on other boards.)

```cpp
#include <Wire.h>
#include <RbAmp.h>

static RbAmp dev(Wire, 0x50);

void setup() {
    Serial.begin(115200);
    Wire.begin();
    while (!dev.begin()) delay(1000);          // detects BASIC or STD
    if (dev.line() == RbAmpLine::Std) {
        dev.stdArmBanks((1u << 0) | (1u << 1)); // arm banks 0 and 1
        dev.stdLatch(1);                        // start the first period of bank 1
    }
}

void loop() {
    RbAmpSnapshot s;
    if (dev.readAll(s)) {
        Serial.printf("U=%.1f V  I=%.2f A  P=%.1f W\n", s.voltage, s.current[0], s.power[0]);
        if (dev.line() == RbAmpLine::Std)
            Serial.printf("Q=%.1f var  S=%.1f VA\n", dev.readReactivePower(0), dev.readApparentPower(0));
    }
    delay(1000);
}
```

A master-timed period: call `dev.stdLatch(bank)` at the boundary, wait a few window lengths, read it with
`dev.stdReadLatch(bank, L)` and convert with `RbAmp::stdLatchWh(...)`. The register-level procedure is
in [Energy and periods](energy-periods.md).

---

## Modbus RTU with Python (pymodbus, STD-UI1)

Reads over the module's UART lines through a serial adapter (with an external RS-485 transceiver where
the link needs one). Set the interface mode switch to **Modbus default** (9600 8N1, unit 1) or to **Modbus
RTU** with your saved settings. Written for pymodbus 3.x (`device_id=`; older 3.x releases call it `slave=`).
All multi-register values are big-endian, high word first (ABCD).

```python
import struct
from pymodbus.client import ModbusSerialClient

def i32(r, i):
    v = (r[i] << 16) | r[i + 1]
    return v - (1 << 32) if v & 0x80000000 else v

def i64(r, i):
    v = 0
    for w in r[i:i + 4]:
        v = (v << 16) | w
    return v - (1 << 64) if v & (1 << 63) else v

def f32(r, i):
    return struct.unpack(">f", struct.pack(">HH", r[i], r[i + 1]))[0]

client = ModbusSerialClient(port="/dev/ttyUSB0", baudrate=9600, parity="N", stopbits=1, bytesize=8)
client.connect()
UNIT = 1

# 1. Identity (input 0x6000): 'ST', map version, variant, status, frequency, window counter
r = client.read_input_registers(0x6000, count=33, device_id=UNIT).registers
assert r[0] == 0x5354                       # 'S','T'
print("map", hex(r[1]), "variant", hex(r[7]), "status", hex(r[9]), "f =", r[0x13] / 100, "Hz")

# 2. Voltage (input 0x6100): U_RMS, i32 mV
u = client.read_input_registers(0x6100, count=6, device_id=UNIT).registers
print("U =", i32(u, 0) / 1000, "V")

# 3. Current channel 0 (input 0x6200): I (µA), P (mW), Q (mvar), S (mVA), PF (Q10, signed)
c = client.read_input_registers(0x6200, count=9, device_id=UNIT).registers
pf = c[8] - 65536 if c[8] & 0x8000 else c[8]
print("I =", i32(c, 0) / 1e6, "A  P =", i32(c, 2) / 1000, "W  Q =", i32(c, 4) / 1000, "var  S =", i32(c, 6) / 1000, "VA  PF =", pf / 1024)

# 4. Energy (input 0x6B00): import / export / net in Wh (i64), then mWh at +0x14
e = client.read_input_registers(0x6B00, count=32, device_id=UNIT).registers
print("import", i64(e, 0), "Wh  export", i64(e, 4), "Wh  net", i64(e, 8), "Wh")

# 5. The same quantities as float32 profile values (input 0x0000)
flt = client.read_input_registers(0x0000, count=76, device_id=UNIT).registers
print("float32: V L1", f32(flt, 0x00), "I L1", f32(flt, 0x06), "P total", f32(flt, 0x34),
      "f", f32(flt, 0x46), "import kWh", f32(flt, 0x48), "export kWh", f32(flt, 0x4A))
client.close()
```

The power factor is meaningful only with load current; at no load it can read any value.

In the float32 profile a single-phase module publishes its channel as L1; L2 and L3 read 0.0, as do the S and PF
points and every address outside the list in the [Modbus transport](reference/12_transport_modbus.md).
The configuration registers (`0x8000`: unit id, baud code, frame format, virtual ids) read the same way
with `read_holding_registers`.

---

## Modbus RTU with mbpoll (STD-UI1)

`mbpoll` reads the module from a command line. Set the interface mode switch to **Modbus default** (9600 8N1, unit 1) and replace `<port>` with your serial port. The options used: `-0` (0-based register numbers, so they match the register map), `-1` (one poll), `-B` (big-endian word order, high word first, for 32-bit values), `-o 2` (2 s timeout), `-t 3` (input registers) or `-t 4` (holding registers). The reference after `-r` is decimal.

```sh
# configuration, holding 0x8000: unit id, baud code, frame format, ...
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 4 -r 32768 -c 10 <port>

# identity, input 0x6000: 'ST', map version, variant, status, frequency, ...
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:hex -r 24576 -c 33 <port>

# U_RMS, input 0x6100: one 32-bit value, mV
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:int -B -r 24832 -c 1 <port>

# current channel 0, input 0x6200: I (µA), P (mW), Q (mvar), S (mVA) as four 32-bit values
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:int -B -r 25088 -c 4 <port>

# float32 profile: V L1, V L2, V L3 (single-phase modules read 0 for L2/L3), I L1 (float)
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:float -B -r 0 -c 4 <port>

# float32 profile: total P (float)
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:float -B -r 52 -c 1 <port>

# float32 profile: frequency, import kWh, export kWh (float)
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:float -B -r 70 -c 3 <port>

# energy in Wh, input 0x6B00: 64-bit values as 4 words each (import, export, ...)
mbpoll -m rtu -b 9600 -P none -a 1 -0 -1 -o 2 -t 3:hex -r 27392 -c 8 <port>
```

What the answers look like:

- **Configuration:** ten registers; on a module with default settings the first three read `1` (unit id), `2` (baud code for 9600) and `0` (frame 8N1).
- **Identity:** `[24576]` reads `0x5354` (`'S','T'`), `[24577]` the register-map version (`0x0036`), `[24583]` the variant (`0x0011` for STD-UI1), `[24585]` the status, `[24595]` the frequency in 0.01 Hz (`0x1388` = 50.00 Hz).
- **32-bit integer reads** (`:int -B`) print one value per two registers: `[24832]` the voltage in mV; `[25088]`, `[25090]`, `[25092]`, `[25094]` the current in µA and the power, reactive power and apparent power in mW, mvar and mVA.
- **Float32 profile reads** print one value per two registers; the points are listed in the [Modbus transport](reference/12_transport_modbus.md#float32-profile-input-0x0000).
- **Energy** is a 64-bit value: `mbpoll` has no 64-bit type, so read four 16-bit words and combine them high word first.

---

## ESPHome: I²C (STD-UI1)

The `rbamp` external component of [rbamp-esphome](https://github.com/rb-amp/rbamp-esphome) reads the module over I²C (the component is placed in `components/` next to the YAML). At start-up it reports the register-map version, the variant, the channels and the energy source. The sensors are published every 10 s.

```yaml
esphome:
  name: std-ui1-i2c

esp32:
  board: esp32dev
  framework:
    type: arduino

logger:

i2c:
  sda: GPIO25
  scl: GPIO26
  frequency: 50kHz

external_components:
  - source:
      type: local
      path: components
    components: [rbamp]

rbamp:
  id: meter1
  address: 0x50
  update_interval: 10s

sensor:
  - platform: rbamp
    rbamp_id: meter1
    voltage:
      name: "Voltage"
    current:
      name: "Current"
    power:
      name: "Power"
    energy_total:
      name: "Energy import"
    energy_exported_total:
      name: "Energy export"
```

`energy_total` and `energy_exported_total` count the energy seen since the component started (they are restored across restarts of the ESP). They are **not** the module's life-time counters; read those from the module (see [Energy and periods](energy-periods.md)). The 50 kHz bus speed is the recommendation for ESP32 hosts.

---

## ESPHome: Modbus RTU (STD-UI1)

The stock `modbus_controller` reads the module over its UART lines: set the interface mode switch to **Modbus default** (9600 8N1, unit 1). Connect the master's TX to pin 3 (SDA, module RX) and the master's RX to pin 4 (SCL, module TX) of the module connector — the module receives on pin 3 and transmits on pin 4 — so `tx_pin` is the wire to pin 3 and `rx_pin` the wire to pin 4. The controller id must not be `std`: it collides with the C++ namespace and the build fails.

```yaml
esphome:
  name: std-ui1-modbus

esp32:
  board: esp32dev
  framework:
    type: arduino

logger:

uart:
  id: mod_bus
  tx_pin: GPIO25        # to the module's pin 3 (SDA, module RX)
  rx_pin: GPIO26        # from the module's pin 4 (SCL, module TX)
  baud_rate: 9600
  parity: NONE
  stop_bits: 1

modbus:
  id: modbus1
  uart_id: mod_bus

modbus_controller:
  - id: std_meter
    address: 1
    modbus_id: modbus1
    update_interval: 10s

sensor:
  # float32 profile points (input registers, ABCD)
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Voltage L1"
    register_type: read
    address: 0x0000
    value_type: FP32
    unit_of_measurement: V
    accuracy_decimals: 1
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Current L1"
    register_type: read
    address: 0x0006
    value_type: FP32
    unit_of_measurement: A
    accuracy_decimals: 3
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Power total"
    register_type: read
    address: 0x0034
    value_type: FP32
    unit_of_measurement: W
    accuracy_decimals: 1
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Frequency"
    register_type: read
    address: 0x0046
    value_type: FP32
    unit_of_measurement: Hz
    accuracy_decimals: 2
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Energy import (float profile)"
    register_type: read
    address: 0x0048
    value_type: FP32
    unit_of_measurement: kWh
    accuracy_decimals: 3
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Energy export (float profile)"
    register_type: read
    address: 0x004A
    value_type: FP32
    unit_of_measurement: kWh
    accuracy_decimals: 3
  # Native map: import energy in Wh, i64 big-endian (input 0x6B00)
  - platform: modbus_controller
    modbus_controller_id: std_meter
    name: "Energy import (native)"
    register_type: read
    address: 0x6B00
    value_type: S_QWORD
    unit_of_measurement: Wh
    accuracy_decimals: 0
```

The float32 profile points and the native import energy report the same energy. If your serial adapter is RS-485, put an external transceiver between the master UART and the module (see [Hardware](hardware.md)).

---

## Without a library

Over Modbus any Modbus stack can read the module's native map or the float32 profile points
(input `0x0000`); see [Getting started](getting-started.md).
