# Modbus RTU transport

On the Modbus setting of the transport jumper the module is a Modbus RTU slave on RS-485 — a
ready-to-poll electricity meter. The Modbus layer is a **translation onto the same register file**
as I²C: same quantities, same commands, one command queue; only the wire format differs.

---

## Serial

- **Framing:** standard RTU `[addr][function][data][CRC16]`, CRC16 poly 0xA001 (reflected),
  LSB first. A frame with a bad CRC gets **no answer** (standard RTU behaviour).
- **Unit id:** 1..247, default **1** (`CFG_MODBUS_ADDR`). 0 = broadcast (whitelisted writes only,
  see below). Ids 248..255 are rejected on configuration.
- **Baud:** default **9600**. Codes in `CFG_MB_BAUD`: 0=2400 · 1=4800 · 2=9600 · 3=19200 · 4=38400 ·
  5=57600 · 6=115200 · 7=230400 · 8=460800 · 9=921600 · 10=1 000 000. All current builds accept
  codes 0–10. If the stored code is ever invalid, the module boots at 9600.
- **Frame format:** `CFG_MB_FRAME` 0=8N1 · 1=8E1 · 2=8O1 · 3=8N2.
- **Response hold-off:** `CFG_TX_DELAY` 0..100 ms — for auto-direction RS-485 adapters that need
  time to release the line.
- Serial settings are validated on write (exception 03). **A write only stages the value in RAM** —
  to make it effective: save (`COMMAND` 0x04), then restart (`COMMAND` 0x02, UNLOCK-gated); the
  module boots with the saved values. Step-by-step: [Sequences D4](22_sequences.md#d4--address--serial-change).
- **Reply latency:** the requested range is copied atomically out of the published image, so even a
  125-register read is internally coherent; the device answers in **< 20 ms** at any baud (suited to
  inverter polling at 3–5 Hz).
- **Replies to restart-causing commands** (RESET, firmware ACTIVATE, address change by UID) may be
  truncated or absent — the device may restart before the reply completes, and a configured TX delay
  can suppress it entirely. Confirm by re-polling after the restart, not by the reply.

---

## Function codes

**FC03** read holding · **FC04** read input · **FC06** write single · **FC16** write multiple.
Any other function code → exception 01.

Limits: FC03/FC04 quantity 1..125, FC16 1..123 — outside → exception 03.
**FC16 is all-or-nothing:** the whole frame is validated first; one bad register → nothing applied.
`COMMAND` (holding 0x8113) is write-only — reading it → exception 02.

---

## Register spaces

Modbus addresses are **their own namespace** — independent of I²C addresses. Layout:

| Space | Range | Content |
|---|---|---|
| Input | 0x0000–0x03FF | **SDM630-compatible float32 profile** (13 live points, below) |
| Input | 0x6000–0x6FFF | **native integer-scale profile** (discovery, channels, energy, totals) |
| Holding | 0x8000–0x83FF | native config · time/commands · firmware update |
| Holding | 0x8400–0x847F | security window (device certificate / attestation; byte stream) |
| Holding | 0xA000–0xAFFF | read-only mirror of native input (= input address + 0x4000) for FC03-only masters |
| — | everything else | unmapped → exception 02 |

The per-block native and holding addresses are listed with the I²C map in the
[register map](13_register_map.md#modbus-mapping).

---

### Native profile (input 0x6000+)

Fixed-scale integers, same units as I²C. **Word order ABCD** (high word first) by default;
`CFG_WORD_ORDER` = 1 swaps to low-word-first for all multi-register numbers in the native input
profile (i64 = all four words reversed), and — from image 0x79 — the absent-channel sentinels swap
the same way. **The holding windows (0x81xx/0x83xx) are always high-word-first**, regardless of
`CFG_WORD_ORDER`. i32 = 2 registers, i64 = 4 registers. Totals P/Q/S are published as i64 over
Modbus (sign-extended from the device values).

---

### SDM630-compatible profile (input 0x0000)

IEEE-754 float32, ABCD always, physical units (V/A/W/kWh), export as positive magnitude — a drop-in
for inverters and SCADA that expect an Eastron-type meter. **Live points (13):**

| Input addr | Point |
|---|---|
| 0x0000 / 0x0002 / 0x0004 | voltage L1 / L2 / L3, V |
| 0x0006 / 0x0008 / 0x000A | current L1 / L2 / L3, A |
| 0x000C / 0x000E / 0x0010 | active power L1 / L2 / L3, W |
| 0x0034 | total active power, W |
| 0x0046 | frequency, Hz |
| 0x0048 / 0x004A | import / export energy, kWh |

Every other address in 0x0000–0x03FF reads 0. Phases the module does not have read 0.0 (from image
0x79). Single-phase members publish their channel as L1. There are no SDM configuration aliases —
configuration is done through the native holding registers.

---

### Virtual unit ids

With `CFG_VIRTUAL_IDS` = 1 (saved + restart; takes effect at boot) a module with more than one
current channel also answers on unit ids **base+1 … base+N_CURRENT** (clamped at 247): each id is a
**read-only single-channel SDM-style view** of one current channel — its voltage, current, power,
total power = that channel's power, frequency; energy points read 0. Any write to a virtual id →
exception 02. Pick the base unit id so the derived ids do not collide with other meters on the bus.

(The METER_GROUP configuration table in the map is **reserved** — group composition is not
configurable on current firmware.)

---

## Broadcast (unit id 0)

Only these writes are accepted on broadcast; everything else is silently refused (no exception
response on broadcast, per the Modbus standard):

- time set — **two frames**: FC16 to 0x8100–0x8101 (epoch, high word first), then FC06
  `COMMAND` 0x8113 = 0x0006 — bus-wide clock sync;
- `LATCH_BANK` / `PERIOD_ARM` 0x8110–11 and `COMMAND` 0x03 — synchronized snapshot across modules
  (on builds with snapshot banks);
- address-by-UID staging 0x8118–0x811E + `COMMAND` 0x16 — commissioning on a shared bus.

A broadcast FC16 must stay within the whitelisted registers — a frame that touches anything else is
refused as a whole.

---

## UNLOCK (gated commands)

Commands that change device state must be preceded by writing the unlock key **0xA5C3** to holding
**0x8112**:

- The key is valid for **10 s** and is consumed by **one** gated command.
- Writing a wrong value **clears** an active unlock.
- A gated command without a valid unlock → exception 03.
- Reading 0x8112 returns the unlock flag (0/1) as last set — treat it as informational only.
- Unlock and command may go in **one FC16 frame** at 0x8112–0x8113 (key first, opcode second).

Gated set: `0x01` SAVE_CAL · `0x02` RESET · `0x07` ENERGY_RESET (where implemented) ·
`0x10/0x12/0x13/0x14` (firmware update) · `0x15` (service). Not gated: `0x04` SAVE_CONFIG · `0x06`
SET_TIME · `0x08–0x0F` · `0x11` page write · `0x16` address by UID. On modules that do not implement
an opcode the refusal is exception 03 before the gate is even checked. The life-time energy commands
use a different, single-shot key mechanism (`LT_KEY`), not UNLOCK — see [Commands](14_commands.md).

---

## Behaviour rules (family contract)

- `COMMAND`, `LATCH_BANK`, `PERIOD_ARM` values > 0xFF → exception 03 (no silent low-byte
  truncation).
- A `COMMAND` write while a command is still executing → exception 06 (device busy).
- **Byte-pair windows** (security 0x8400+, firmware payload 0x8380+, history record 0x8180+):
  register k carries bytes 2k (high) and 2k+1 (low). Writes that touch read-only bytes → exception
  02. The security window is a **little-endian byte stream** — numeric fields inside it decode LE
  from the bytes, not as big-endian registers.
- One table of legal configuration values serves both transports: an illegal value over Modbus →
  exception 03; the same value over I²C is refused at SAVE_CONFIG with `ERROR` 0xFD — in both cases
  nothing is persisted.
- Fields the build does not offer (absent channels; THD and displacement quantities with
  CAPABILITIES bit 2 = 0) read the sentinel in the native profile (`0xFFFF` / `0x8000…`); in the SDM
  profile they read 0.

---

## Exceptions

| Code | Meaning |
|---|---|
| 01 | illegal function (anything but FC03/04/06/16) |
| 02 | address outside the map · write to an input register, read-only holding or virtual unit id · read of a write-only register |
| 03 | illegal value (bad unit id / baud code / config value · invalid command opcode · oversized COMMAND/LATCH value · quantity out of range · gated command without unlock) |
| 04 | internal device failure |
| 06 | device busy (command or firmware operation in progress) |

The last exception is kept for diagnostics in `LAST_EXCEPTION` (`FC<<24 | code<<16 | address`),
readable on **both** transports: I²C 0x0158, Modbus holding 0x830A–0B.
