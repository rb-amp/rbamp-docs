# Sequences — step-by-step control flows

Canonical host-side flows. Every step names the register (I²C address; Modbus in brackets) and what
to check. The command-polling rules used everywhere here are defined once in
[D2](#d2--executing-a-command).

## D1 — Power-up / discovery

1. Read **12 bytes at 0x0000 in one transaction**: `ID` = 'S','T' · `VERSION` ≥ 0x36 ·
   `CAPABILITIES` (expect bit 9 set; on current builds THD bit 2 = 0, snapshot-bank bit 8 = 0, on-device
   energy bit 13 = 1) · `STATUS` · `N_VOLTAGE` / `N_CURRENT` · `VARIANT` (UI1 0x11 = 1/1,
   0x63 = 3/3, reserved code) · `HW_REV` / `FW_VERSION`.
2. Read `UID` 0x00F4 (12 B) — the device identity for your database; `FW_VER` 0x0114 (u32 image
   version); `BOARD_ID` 0x017C.
3. Wait for live data: `WIN_SEQ` (0x0016) ≠ 0 and stepping, `STATUS` bit 1 (sensor_ok) = 1 — allow
   ~1 s after a boot ([boot readiness](16_measurement_energy.md#boot-readiness)).
4. If `STATUS` bit 4 (time_valid) = 0 → set the clock ([D2](#d2--executing-a-command), SET_TIME).
5. Health gate: cal_ok (image 0x79+) or `CAL_SRC` (0x0872) ∈ {1, 2}; `LTA_STATUS` (0x0870) ≤ 1.
6. Re-arm RAM state: `EVENT_MASK` (0x00C1), `THRESH_I/P` (0x00C4/0x00C8); acknowledge the boot by
   clearing RESET_OCCURRED — write 0x20 to `EVENT_FLAGS` (0x00C0).

Modbus: the same identity set is input 0x6000–0x601F ([mirror](10_identity_discovery.md#modbus-discovery-mirror));
`FW_VER` holding 0x8315–16, `BOARD_ID` holding 0x831B–1C.

---

## D2 — Executing a command

1. **Idle check:** `CMD_STATE` (0x006A; holding 0x8116) bit 0 = 0.
2. Write the command's parameter registers, then the opcode to `COMMAND` (0x0060; holding 0x8113 —
   for a Modbus **gated** opcode send one FC16 `{0x8112 = 0xA5C3, 0x8113 = opcode}`; exception 06 =
   busy, exception 03 = unknown opcode or missing unlock).
3. **Poll `CMD_STATE` bit 0 → 0.** Timeouts: ≥ 1 s simple commands and saves · ≥ 0.5 s update page
   write · ≥ 3 s update COMMIT. On I²C tolerate ≥ 10 ms clock stretching; retry NACKed **reads**;
   never blindly resend a COMMAND write (a duplicate would be dropped as OVERRUN, but you lose track
   of which write took).
4. Read `ERROR` (0x0002; holding 0x8114) **after** BUSY = 0: 0x00 = OK, codes in
   [Commands](14_commands.md#error-codes-error-0x0002). Optionally verify `CMD_SEQ` (0x0067)
   stepped by 1 and OVERRUN (bit 1) = 0.

**Time set** (the most common command): write epoch u32 LE to `TIME_SET` 0x08C0 (Modbus FC16
0x8100 = high word, 0x8101 = low word — on broadcast this is its own frame, the command follows as a
second frame) → `COMMAND` 0x06 → poll → `ERROR` 0xFC = epoch below 1 600 000 000. `STATUS` bit 4
sets immediately.

---

## D3 — Configuration

1. Write the wanted `CFG_*` registers (0x0070–0x0085; holding 0x8000–0x800C).
2. `COMMAND` 0x04 SAVE_CONFIG (not UNLOCK-gated) → [D2](#d2--executing-a-command).
   `ERROR` 0xFD = illegal value, nothing saved · 0xF9 = rate limit, retry after ~5 s · 0xF6 = store
   I/O error.
3. If the change needs a restart (I²C address, Modbus unit/baud/frame/tx-delay, `CFG_VIRTUAL_IDS`):
   `COMMAND` 0x02 RESET (UNLOCK-gated on Modbus; expect no/truncated Modbus reply) and re-poll the
   device after ~1 s.

`CFG_LED` / `CFG_WORD_ORDER` act on write but revert at boot unless saved. `CFG_CLASS`,
`CFG_CT_MODEL`, `CFG_TZ_OFFSET_MIN`, `CFG_TIME_AUTH`, `CFG_ENERGY_PERSIST_S` are stored metadata.

---

## D4 — Address / serial change

**I²C address (known device):** `CFG_I2C_ADDR` 0x0073 = 0x08..0x77 → SAVE_CONFIG → BUSY 0 and
`ERROR` 0 → RESET → wait ~1 s → probe `ID` = 'S','T' at the **new** address.

**Modbus serial:** holding 0x8000 unit (1–247) / 0x8001 baud (0–10) / 0x8002 frame / 0x8003 TX delay
(validated on write — exception 03) → FC06 0x8113 = 0x0004 → poll 0x8114 = 0 → FC16
`{0x8112 = 0xA5C3, 0x8113 = 0x0002}` (reply may be lost) → reconnect at the new settings.

**By UID (several modules on one address, both transports):**
1. I²C: one write of 0x00E0: UID (12 B) + new I²C address (0x00EC, or 0) + new Modbus unit (0x00ED,
   or 0). Modbus: FC16 0x8118–0x811E (UID byte pairs + 0x811E = addr<<8 | unit) — broadcast allowed.
2. `COMMAND` 0x16 (FC06 0x8113 = 0x0016; broadcast allowed).
3. The matching module persists (together with any staged CFG_ values) and restarts; every other
   module answers `ERROR` 0xFC. Both new-address fields 0 or an illegal address → 0xFD; save rate
   limit → 0xF9.
4. Probe the module at its new address/unit; identify by UID.

---

## D5 — Energy: read, reset, adopt

**Read (each interval boundary, your clock):**
1. One I²C transaction ≤ 128 B over 0x0800–0x0843: Wh counters, mWh counters, `LT_GEN`
   (coherent snapshot). Plus `UPTIME` 0x08A0. Modbus: input 0x6B00/0x6B14 + holding 0x8140.
   Capability bit 15 (image 0x7A+): the same counters at µWh resolution — grand at 0x08D0
   (3×i64; Modbus input 0x6BB0), per channel at 0x0C80 + g·0x10 (Modbus 0x6BC0 + g·8) — same
   snapshot as the mWh fields (imp/exp mWh = floor(µWh/1000); net = imp − exp in the same unit),
   same delta/generation/restart rules. Image 0x7B+ adds grand reactive µvarh at 0x08E8/0x08F0
   (Modbus 0x6BAC/0x6BBC, not contiguous): the coherent fine-resolution read is one transaction
   over 0x08D0–0x08F7 (40 B).
2. Interval energy = delta of the mWh counters (µWh counters where finer resolution is wanted).
3. `LT_GEN` changed → do not take the delta; re-anchor (someone reset/adopted — see the event log).
4. `UPTIME` decreased → module restarted: delta = `max(0, Δ)` (up to about the last second of
   energy may be missing), re-feed time ([D2](#d2--executing-a-command)).

**Reset (deliberate, logged):** `LT_KEY` 0x0844 = `0x4C545253` ("LTRS"; Modbus FC16 0x8142–43, high
word first) → **immediately** `COMMAND` 0x0A (any command consumes the key) →
[D2](#d2--executing-a-command). `ERROR` 0xFB = key wrong/missing · 0xFA = store FAULT. No Modbus
UNLOCK needed — the key is the gate. (The family opcode 0x07 ENERGY_RESET is not implemented on
current builds: 0xFE / exception 03.)

**Adopt (board swap / migration):** stage 4×i64 into `LT_ADOPT` 0x0848 (Modbus 0x8144–0x8153) in the
order import Wh · export Wh · reactive import varh · reactive export varh → `LT_KEY` = `0x4C544150`
("LTAP") → `COMMAND` 0x0B. Negative value → 0xFA. Both reset and adopt bump `LT_GEN` and land in the
event log with before/after values.

---

## D6 — History and the event log

1. `HIST_PERIOD` 0x0C00 (holding 0x8160): 0 = last 15-min (then `HIST_BACK` must be 0) · 1 = hourly
   ring · 2 = event log.
2. `HIST_BACK` 0x0C04 (holding 0x8163) = n, 0 = newest.
3. `COMMAND` 0x08 HIST_READ → [D2](#d2--executing-a-command). `ERROR` 0xFC = no such record.
4. On success: `HIST_FLAGS` 0x0C02 bit 0 = 1; read `HIST_COUNT` / `HIST_RECSZ` (80 history / 32
   event) / `HIST_OLDEST` / `HIST_NEWEST` to map timestamps to indices.
5. Read the record at 0x0C20 (`HIST_RECSZ` bytes, one transaction; Modbus holding 0x8180+ byte
   pairs). Layouts: [Measurement & energy](16_measurement_energy.md#history-records).
6. Iterate `HIST_BACK` = 0 … COUNT−1 for a back-fill; diff rules per the record FLAGS.

---

## D7 — Firmware update

1. `FW_ACTIVE_BANK` 0x0156 (holding 0x8301) → pick the release file for the **other** bank.
2. Gate: `BOARD_ID` 0x017C == the file's board word; file version ≥ `FW_VER` 0x0114; read
   `OTA_BANK_SIZE` 0x014C (never hardcode it).
3. (Modbus: UNLOCK) `COMMAND` 0x10 BEGIN → [D2](#d2--executing-a-command) → `OTA_STATUS` 0x0140
   (holding 0x8300) = 1.
4. Per non-blank 256-B page, ascending, **page 0 first**: `OTA_IDX` 0x0144 = k · payload → 0x0500
   (Modbus: two FC16 of 64 regs at 0x8380/0x83C0) · optional `OTA_PCRC_EXP` 0x0170 = page CRC-32 ·
   `COMMAND` 0x11 (not gated) · poll BUSY (≤ ~0.5 s) · `ERROR` 0xFC → rewrite payload, repeat page ·
   check `OTA_STATUS` still 1 (3 → read `OTA_DIAG` 0x0180, restart from BEGIN).
5. (UNLOCK) `COMMAND` 0x12 COMMIT → poll BUSY ≥ 3 s → `OTA_STATUS`: 2 = armed
   (`FW_STAGED_VERSION` 0x0150 = new version) · 3 = refused (reasons 7–15 in `OTA_DIAG`) ·
   4 = lower version.
6. Only when status = 2: (UNLOCK) `COMMAND` 0x13 ACTIVATE — no/truncated Modbus reply; the module is
   back in ~4 s. Or (UNLOCK) `COMMAND` 0x14 FW_ABORT to drop the armed image.
7. **Do not reset the module for ~10 s** (trial + self-confirm), then verify: `FW_ACTIVE_BANK`
   flipped · `FW_VER` = new version · `SWAP_PENDING` 0x011C = 0. Old `FW_VER` = it rolled back —
   read `OTA_DIAG`.

---

## D8 — Modbus unlock

FC06 holding 0x8112 = 0xA5C3 → valid 10 s, consumed by **one** gated command; a wrong value clears
an active unlock; missing/expired → exception 03 on the gated command. Key and opcode may share one
FC16 at 0x8112–0x8113.

Gated: 0x01 · 0x02 · 0x07 · 0x10 · 0x12 · 0x13 · 0x14 · 0x15.
Not gated: 0x04 · 0x06 · 0x08–0x0F · 0x11 · 0x16.
