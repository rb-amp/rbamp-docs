# Configuration, persistence and calibration

## What survives what

| Data | Survives warm reset | Survives power-off | Survives firmware update | How saved |
|---|---|---|---|---|
| Calibration (per channel, with its sensor) | ✓ | ✓ | ✓ | factory |
| Configuration (`CFG_*` flash rows) | ✓ | ✓ | ✓ | command 0x04 SAVE_CONFIG |
| Cumulative energy + generation | ✓ (resumes from about the last publish window) | ✓ (about the last second of energy lost on an ungraceful cut) | ✓ | automatic, every second ([rules](16_measurement_energy.md#cumulative-counters)) |
| Hourly history + event log | ✓ | ✓ | ✓ (back-fill path exists for store migration) | automatic |
| Boot counter | ✓ | ✓ | ✓ | automatic |
| 15-min history record | ✗ | ✗ | ✗ | RAM only |
| `CFG_RATE`, `CFG_HARM` | ✗ | ✗ | ✗ | RAM only |
| `EVENT_MASK`, `THRESH_I/P`, `PERIOD_ARM` | ✗ | ✗ | ✗ | RAM — re-arm after any reset ([DRDY](15_drdy_events.md)) |
| Device clock | ✗ | ✗ | ✗ | master re-feeds time after every reset |

---

## Saving configuration

1. Write the wanted `CFG_*` registers (list: [register map](13_register_map.md#configuration-0x00700x0085-persisted-by-save_config-unless-marked-ram)).
2. `COMMAND` = 0x04 (SAVE_CONFIG). The write is **read-modify-write** — calibration and the other
   config rows are untouched.
3. Values are validated against one shared legal-value table before anything is written: an illegal
   value → `ERROR` 0xFD on I²C / exception 03 on Modbus, and **nothing is persisted**.
4. Saves are **rate-limited** to about one per 5 s — a repeat answers 0xF9, retry later. A save with
   nothing changed is an OK no-op (it still raises the CONFIG_CHANGED event flag).
5. When it takes effect:
   - **after the next restart:** `CFG_I2C_ADDR`, Modbus unit/baud/frame/tx-delay,
     `CFG_VIRTUAL_IDS` — save, then `COMMAND` 0x02 (on Modbus RESET is UNLOCK-gated);
   - **immediately on write** (but reverting at boot unless saved): `CFG_LED`, `CFG_WORD_ORDER`,
     `CFG_START_UA` (image 0x7B+ — [starting current](16_measurement_energy.md#cumulative-counters));
   - **metadata only** (stored, no behaviour today): `CFG_CLASS`, `CFG_CT_MODEL`,
     `CFG_TZ_OFFSET_MIN`, `CFG_TIME_AUTH`, `CFG_ENERGY_PERSIST_S`.

Defaults after factory/fresh state: `CFG_CLASS` = 0 (unset), `CFG_CT_MODEL` = 0 (unset),
`CFG_TIER` = 0 (regular), I²C 0x50, Modbus unit 1 / 9600 8N1 / ABCD, LEDs on. A master can tell an
unconfigured module by the zeros.

Over Modbus the config rows are holding 0x8000–0x800C; SAVE_CONFIG itself is **not** UNLOCK-gated.
Step-by-step: [Sequences D3](22_sequences.md#d3--configuration).

---

## Calibration — the user-visible part

**Every channel is calibrated at the factory together with its own current sensor**; the pair is
matched (sensors are labelled per channel) and the calibration is stored on the device. It survives
resets, power loss and firmware updates. There is no user calibration procedure, and none is needed
after installation.

What a user/master *can* do:

- **Read health:** `STATUS` bit 0 (cal_ok). From image **0x7A**: cal_ok is set from a **marker
  stored with the calibration** when it is saved — a unit calibrated under older firmware keeps the
  0x79 rule below until its calibration is saved once more. On image **0x79**: cal_ok = 1 only when
  **every** channel carries a measured (non-default) gain and the stored calibration matches the
  running firmware; built-in defaults → 0. On modules **before 0x79** cal_ok could read 1 even on
  defaults — there, judge by `CAL_SRC`. `CAL_SRC` (0x0872) says where the calibration came from:
  1/2 = stored calibration · 0 = built-in defaults · **3 = the stored calibration does not match
  this firmware's measurement range — defaults in use**. 0 or 3 means the module needs service —
  readings are not to calibrated accuracy.
- **Set metadata:** `CFG_CLASS` (sensor class) and `CFG_CT_MODEL` (sensor model code) are
  **reference metadata** — they label what is connected for fleet inventory and do **not** change
  gains. Persist with SAVE_CONFIG.
- **Read the calibration blocks** (CAL_V / CAL_I in the map) for audit — e.g. to verify two modules
  are not accidentally swapped with their sensors.

Writing the calibration blocks and `SAVE_CAL` (0x01) is a **service/factory operation** (UNLOCK-gated
over Modbus; calibration saves are event-logged with a checksum). Incorrect values produce wrong
readings with no warning — do not write these registers outside a service procedure.

The per-channel sensor pairing matters operationally: when moving a module, move its sensors with
it, and keep each sensor on its labelled channel. A swapped sensor does not raise an error — the
channel still measures, but to the other sensor's calibration.
