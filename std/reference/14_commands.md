# Commands

All device actions go through the **COMMAND register (`0x0060`, u8)** — on Modbus, holding `0x8113`
(write-only there). A write queues the opcode; execution happens in the device's main loop
(milliseconds for most commands; flash-touching commands take longer).

## Completion and result semantics

- **Busy:** `CMD_STATE` (0x006A) bit 0 = BUSY while a command executes. **BUSY is the completion
  signal** — poll it to 0 before reading the result. A new command written while BUSY is **dropped**
  (bit 1 OVERRUN set; the running command is not clobbered; OVERRUN clears when the next command is
  accepted). On Modbus a COMMAND write while busy → exception 06.
- **Result:** `ERROR` (0x0002; Modbus readback holding 0x8114) holds the result after completion
  (0x00 = OK). From image **0x79** `ERROR` reads 0xF0 (Modbus 0xFFFF) for the whole execution and is
  written together with completion. **On modules before 0x79** `ERROR` may already read 0 while
  BUSY = 1 — judge completion by BUSY only, never by ERROR alone.
- **Freshness:** `CMD_SEQ` (0x0067) increments once per *finished* command — poll it to distinguish
  a fresh result from a stale one.
- **Pattern:** check BUSY = 0 → write parameters, then COMMAND → poll BUSY → 0 (timeouts: ≥1 s for
  simple commands and saves, ≥0.5 s for an update page write, ≥3 s for update COMMIT; on I²C
  tolerate ≥10 ms clock stretching and retry NACKed *reads* — never blindly resend a COMMAND write)
  → read `ERROR`. Full walk-through: [Sequences D2](22_sequences.md#d2--executing-a-command).
- **Commands that restart the module** (0x02 RESET, 0x13 ACTIVATE, 0x16 ADDR_BY_UID on the matching
  module): over Modbus the reply may be truncated or absent (and a configured TX delay can suppress
  it); confirm by re-polling the device after the restart.

## Error codes (`ERROR` 0x0002)

| Code | Name | Meaning |
|---|---|---|
| 0x00 | OK | accepted / completed |
| 0xF0 | PENDING | command still executing (with BUSY = 1) |
| 0xF6 | NVM_IO | a persist write failed (SAVE_CAL / SAVE_CONFIG / history import) |
| 0xF7 | NO_KEY | attestation refused — no provisioned device key |
| 0xF8 | — | operation refused by service policy |
| 0xF9 | NVM_RATE_LIMIT | persist rewrite rate-limited (min ~5 s between saves) — retry later |
| 0xFA | OP_RETRY | operation already in progress · refused in the current state (e.g. energy store FAULT, negative adopt value) |
| 0xFB | KEY_GATE | keyed command without a valid single-shot key (`LT_KEY`) |
| 0xFC | VALUE_CRC | update-page CRC mismatch (resend the page) · time epoch below 1 600 000 000 · UID mismatch (ADDR_BY_UID) · no such history record (HIST_READ) |
| 0xFD | PARAM | invalid parameter / config value — **nothing persisted** |
| 0xFE | UNKNOWN_CMD | opcode not implemented by this build |

## Command set

### Family commands

| Op | Name | Effect |
|---|---|---|
| 0x01 | SAVE_CAL | persist calibration registers (service/factory; UNLOCK-gated on Modbus; rate-limited → 0xF9) |
| 0x02 | RESET | warm reset (UNLOCK-gated on Modbus; expect no/truncated Modbus reply) |
| 0x03 | LATCH_PERIOD | snapshot + reset latch bank `LATCH_BANK` — only on builds with latch banks (capability bit 8); otherwise 0xFE |
| 0x04 | SAVE_CONFIG | persist configuration (read-modify-write; calibration untouched; **not** UNLOCK-gated). Values validated against the legal table; illegal → 0xFD, nothing saved; rate-limited → 0xF9; an unchanged save is an OK no-op (still raises CONFIG_CHANGED) |
| 0x05 | CAL_THD_FLOOR | capture the no-load THD floor (service; builds with THD) — otherwise 0xFE |
| 0x06 | SET_TIME | apply the epoch written to TIME_SET/RTC_EPOCH; epoch < 1 600 000 000 → 0xFC; STATUS bit 4 sets immediately |
| 0x07 | ENERGY_RESET | family opcode, UNLOCK-gated. **Current builds do not implement it** (0xFE / exception 03) — their counter reset is the keyed LT_RESET below |
| 0x0E | IDENTIFY | blink the LEDs for 10 s — locate the module in a cabinet. Not gated |
| 0x0F | ATTEST | sign the 32-B challenge at 0x0B80 ([register map](13_register_map.md#security-block-0x0b000x0bff)); ~55 ms; 0xF7 = no key |
| 0x10 | OTA_BEGIN | open an update session ([Firmware update](18_firmware_update.md)) |
| 0x11 | OTA_BLOCK_WR | program one 256-B page (throttled to one page / ~250 ms — BUSY until its slot) |
| 0x12 | OTA_COMMIT | verify + arm the staged image (no reset); poll BUSY ≥ 3 s |
| 0x13 | OTA_ACTIVATE | reset into the armed image (only when OTA_STATUS = 2; ~4 s re-appear) |
| 0x14 | FW_ABORT | close the update session / disarm an armed image |
| 0x15 | — | reserved (factory/service) |
| 0x16 | ADDR_BY_UID | the module whose UID matches `ADDR_UID` takes the staged address(es), **persists them together with any other pending CFG_ changes** and restarts; other modules answer 0xFC. Both new-address fields zero → 0xFD; illegal address → 0xFD; save rate limit → 0xF9. Works on a Modbus broadcast. Not gated |

### Keyed commands (life-time counters and history)

These change metering-relevant state, so each requires the matching **single-shot key** written to
`LT_KEY` (0x0844 / Modbus holding 0x8142–43, high word first) immediately before the command. The
key is consumed by **any** next command; wrong or absent key → `ERROR` 0xFB. This is
deliberate-action protection on top of the transport (not the Modbus UNLOCK).

| Op | Name | Key (u32) | Effect |
|---|---|---|---|
| 0x08 | HIST_READ | — (not keyed) | fetch the record selected by HIST_PERIOD/HIST_BACK into HIST_REC; no such record → 0xFC |
| 0x09 | HIST_FLUSH | — (not keyed) | flush energy now: the retention copy always; the NVM record only if ≥ 10 s since its last write (also implied by RESET and update activation) |
| 0x0A | LT_RESET | `0x4C545253` "LTRS" | zero life-time energy; generation counter ++ (logged). Store FAULT → 0xFA |
| 0x0B | LT_ADOPT | `0x4C544150` "LTAP" | adopt the staged counter values (board replacement / migration); generation ++ (logged). Staging order: import Wh · export Wh · reactive import varh · reactive export varh (4×i64); a negative value → 0xFA |
| 0x0C | HIST_WRITE | `0x48494D50` "HIMP" | import the 80-B record in HIST_REC into the hourly ring (`HIST_PERIOD` must be 1, else 0xFC); timestamp not newer → 0xFD; NVM I/O → 0xF6 |
| 0x0D | HIST_CLEAR | `0x48434C52` "HCLR" | step-wise erase of the hourly ring; repeat while running → 0xFA (logged) |

Life-time resets, adopts and history writes are recorded in the event log with before/after values —
they cannot happen silently.

## Gating summary

| Mechanism | Scope | How |
|---|---|---|
| Modbus **UNLOCK** | 0x01 · 0x02 · 0x07 · 0x10 · 0x12 · 0x13 · 0x14 · 0x15 | key 0xA5C3 → holding 0x8112; 10-s window, one command; may share one FC16 with the opcode ([Modbus](12_transport_modbus.md#unlock-gated-commands)) |
| **LT_KEY** | 0x0A–0x0D, both transports | per-command magic, single-shot |
| Broadcast whitelist | Modbus unit 0 | only time-set, latch, address-by-UID ([Modbus](12_transport_modbus.md#broadcast-unit-id-0)) |

Not gated on either transport: 0x04, 0x06, 0x08, 0x09, 0x0E, 0x0F, 0x11, 0x16.
I²C has no UNLOCK step: physical access to the board's I²C bus is the installation's trust boundary;
the keyed commands are gated on both transports regardless.
