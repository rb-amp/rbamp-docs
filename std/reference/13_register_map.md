# Register map reference

The complete I²C register map (map version `0x36`) with the Modbus equivalents. Unless stated
otherwise: values little-endian, read-only. "Variant diag" marks registers whose meaning is
build-specific — gate on `VARIANT` before interpreting them. All blocks below are implemented by every
current STD build (one code base) unless marked *reserved*.

Access column: r = read, w = write, rw = both. Persistence: regs marked *flash* survive power-off
after SAVE_CONFIG / SAVE_CAL; *RAM* regs reset to defaults at every boot.

## CORE 0x0000–0x00FF

| Addr | Name | Type | Acc | Meaning |
|---|---|---|---|---|
| 0x0000 | ID | 2×u8 | r | `'S','T'` |
| 0x0002 | ERROR | u8 | r | last command result (0x00 = OK; codes in [Commands](14_commands.md)) |
| 0x0003 | VERSION | u8 | r | map version (0x36) |
| 0x0004 | CAPABILITIES | u16 | r | feature bitmap ([Identity](10_identity_discovery.md)) |
| 0x0006 | STATUS | u8 | r | status bits ([Diagnostics](19_diagnostics.md)) |
| 0x0007 | N_VOLTAGE | u8 | r | voltage channels 0..8 |
| 0x0008 | N_CURRENT | u8 | r | current channels 1..64 |
| 0x0009 | VARIANT | u8 | r | member code |
| 0x000A | HW_REV | u8 | r | hardware revision |
| 0x000B | FW_VERSION | u16 | r | firmware line marker (not the image version) |
| 0x0010 | FREQ | u16 | r | mains frequency, cHz |
| 0x0012 | PHASE_SEQ | u8 | r | image 0x7A+: phase sequence from the voltage angles — 0 not available (fewer than 3 voltage inputs, or one below 50 V) · 1 forward L1-L2-L3 · 2 reverse · 3 abnormal (no 120° pattern, e.g. two inputs on one phase) |
| 0x0014 | WINDOW_MS | u16 | r | measurement window, ms (reads 0 on builds that do not publish it) |
| 0x0016 | WIN_SEQ | u16 | r | publish counter: +1 per published snapshot — once per measurement window, ~100 ms on image 0x7C+ (every 200 ms up to 0x7B); keeps stepping without mains on a slower fallback tick; steps in the same instant as the DRDY pulse ([DRDY](15_drdy_events.md)) |
| 0x0020 | FRAME_CNT | u32 | r | measurement frames since boot (liveness) |
| 0x0024 | CRC_ERR | u32 | r | frame CRC errors (reads 0 on builds that report a per-frame flag instead) |
| 0x0060 | COMMAND | u8 | w | command opcode ([Commands](14_commands.md)) |
| 0x0062 | LATCH_BANK | u8 | rw | target bank for LATCH_PERIOD (builds with snapshot banks) |
| 0x0063 | PERIOD_ARM | u8 | rw | snapshot-bank arm mask (RAM) |
| 0x0066 | PD_VALID | u8 | r | snapshot-bank valid mask (sticky) |
| 0x0067 | CMD_SEQ | u16 | r | completed-command counter |
| 0x006A | CMD_STATE | u8 | r | bit0 BUSY · bit1 OVERRUN |
| 0x0090 | SCRATCH | 16 B | w | master scratch window |
| 0x00C0 | EVENT_FLAGS | u8 | rw | sticky event bits, **write-1-to-clear** (RAM) |
| 0x00C1 | EVENT_MASK | u8 | rw | which event bits hold DRDY low (alarm); RAM, default 0 |
| 0x00C4 | THRESH_I | u32 | rw | current threshold µA; 0xFFFFFFFF = off (RAM) |
| 0x00C8 | THRESH_P | u32 | rw | \|total P\| threshold mW; 0xFFFFFFFF = off (RAM) |
| 0x00CC | RESET_CAUSE | u8 | r | cause of this boot ([Diagnostics](19_diagnostics.md)) |
| 0x00CE | BOOT_COUNT | u16 | r | persisted boot counter (low 16 bits) |
| 0x00E0 | ADDR_UID | 12 B | w | target UID for ADDR_BY_UID; cleared after the command |
| 0x00EC | ADDR_UID_I2C | u8 | w | new I²C address 0x08..0x77, 0 = keep |
| 0x00ED | ADDR_UID_UNIT | u8 | w | new Modbus unit 1..247, 0 = keep |
| 0x00F4 | UID | 12 B | r | unique chip id |

### Configuration 0x0070–0x0085 (persisted by SAVE_CONFIG unless marked RAM)

| Addr | Name | Type | Acc | Default | Meaning |
|---|---|---|---|---|---|
| 0x0070 | CFG_CLASS | u8 | rw | 0 | sensor class metadata (0 = unset) |
| 0x0071 | CFG_CT_MODEL | u8 | rw | 0 | CT model **reference only** — does not change gains |
| 0x0072 | CFG_TIER | u8 | r | 0 | 0 = regular · 1 = industrial; read-only (build/factory property) |
| 0x0073 | CFG_I2C_ADDR | u8 | rw | 0x50 | 7-bit address 0x08..0x77; applies on reset |
| 0x0074 | CFG_RATE | u16 | rw | — | report-rate hint ms (**RAM**) |
| 0x0076 | CFG_HARM | u8 | rw | 0 | bit0 = harmonics enable (**RAM**; effective only with CAPABILITIES bit 2) |
| 0x0077 | CFG_MODBUS_ADDR | u8 | rw | 1 | Modbus unit id 1..247 |
| 0x0078 | CFG_MB_BAUD | u8 | rw | 2 | baud code 0..10 ([Modbus](12_transport_modbus.md)) |
| 0x0079 | CFG_MB_FRAME | u8 | rw | 0 | 0=8N1 · 1=8E1 · 2=8O1 · 3=8N2 |
| 0x007A | CFG_TX_DELAY | u8 | rw | 0 | RS-485 response hold-off, 0..100 ms |
| 0x007B | CFG_WORD_ORDER | u8 | rw | 0 | 0=ABCD · 1=word-swapped (native profile only) |
| 0x007C | CFG_TIME_AUTH | u8 | rw | 0 | 0=master clock · 1=device clock (stored; not applied on current builds) |
| 0x007D | CFG_ENERGY_PERSIST_S | u16 | rw | auto: 60 without battery clock / 3600 with | energy milestone period, s (stored + persisted; not applied on current builds) |
| 0x007F | CFG_TZ_OFFSET_MIN | i16 | rw | 0 | timezone offset, min, −720..+840 (stored for future tariffs) |
| 0x0081 | CFG_VIRTUAL_IDS | u8 | rw | 0 | publish per-channel virtual Modbus unit ids (applied at boot; [Modbus](12_transport_modbus.md#virtual-unit-ids)) |
| 0x0082 | CFG_LED | u8 | rw | 1 | status LEDs: 1 = on, 0 = off (IDENTIFY still works) |
| 0x0084 | CFG_START_UA | u16 | rw | 400 | image 0x7B+: starting current for energy, µA — while a channel's current is below it, that channel adds no active or reactive energy to any counter; readings (I/P/Q/S/PF) unaffected. 0 = off; 10 µA steps, max 2540 (out of range → rejected); applies at once, SAVE_CONFIG persists |

---

## DIAG / OTA 0x0100–0x018F

`0x0100–0x013F` is **variant-defined diagnostic space**, except the family-contract registers
ADS_ID, RING_DROP, WIN_NSAMP, WIN_PERIOD. Gate on `VARIANT` before interpreting the rest.

| Addr | Name | Type | Acc | Meaning |
|---|---|---|---|---|
| 0x0100 | ADS_ID | u16 | r | measurement front-end device id |
| 0x0102 | — | — | r | variant diag |
| 0x0104 | FRAME_OK | u8 | r | variant diag (three-phase): last-frame-CRC-OK flag 0/1 |
| 0x0106 | RING_DROP | u16 | r | acquisition ring overflows; must stay 0 |
| 0x0108/0x010C | — | u32 | r | variant diag |
| 0x0110 | WIN_NSAMP | u16 | r | samples in the last finalized window |
| 0x0112 | WIN_PERIOD | u16 | r | measured mains period, samples |
| 0x0114 | FW_VER | u32 | r | running image version |
| 0x0118 | ACTIVE_BANK | u32 | r | running bank base (opaque id) |
| 0x011C | SWAP_PENDING | u32 | r | update swap state: 0 confirmed · 1 armed · 2 trial |
| 0x0120 | RAW_IRMS | 6 B | r | variant diag (three-phase): 3×u16 raw current codes |
| 0x0126 | ACQ_DIAG | 12 B | r | variant diag: acquisition counters |
| 0x012E | LOST_FRAMES | u32 | r | measurement frames lost since boot (energy is gap-compensated) |
| 0x0132 | CPU_DIAG | 8 B | r | variant diag: processing load |
| 0x013A | I2C_DIAG | 6 B | r | variant diag: u16 address matches · u16 bus errors · u16 overruns |
| 0x0140 | OTA_STATUS | u8 | r | update status ([Firmware update](18_firmware_update.md)) |
| 0x0144 | OTA_IDX | u32 | rw | 256-B page index for the next page write |
| 0x0148 | OTA_ACTIVE | u32 | r | running bank base (same as 0x0118) |
| 0x014C | OTA_BANK_SIZE | u32 | r | bank size, device-reported — **read at runtime, never hardcode** |
| 0x0150 | FW_STAGED_VERSION | u32 | r | version of the staged image (0 = nothing armed) |
| 0x0154 | — | — | — | reserved (deprecated page-CRC register; do not use) |
| 0x0156 | FW_ACTIVE_BANK | u8 | r | running bank: 0 = A, 1 = B |
| 0x0158 | LAST_EXCEPTION | u32 | r | last Modbus exception `FC<<24 \| code<<16 \| addr` |
| 0x0160 | — | — | — | reserved |
| 0x0170 | OTA_PCRC_EXP | u32 | w | optional expected CRC-32 of the next page |
| 0x0174 | OTA_PCRC_RX | u32 | r | CRC-32 of the page as received |
| 0x0178 | OTA_KEYID | u32 | r | image key id this firmware accepts |
| 0x0180 | OTA_DIAG | 16 B | r | update diagnostics: reason · status · offset · page/len · … |
| 0x01A0–0x01E7 | — | — | — | reserved (service) |
| 0x0500 | OTA_PAYLOAD | 256 B | rw | firmware page buffer |

---

## Measurement blocks

### VOLTAGE — base 0x0200, stride 0x20, v = 0..7 (gate on N_VOLTAGE)

| Off | Name | Type | Unit |
|---|---|---|---|
| +0x00 | U_RMS | i32 | mV |
| +0x04 | U_PEAK | i32 | mV (reads 0 — not computed on current builds) |
| +0x08 | THD_U | u16 | centi-% (THD capability = 0: reads 0 on I²C, sentinel on Modbus native) |
| +0x0A | ANGLE | i16 | centi-degree (image 0x7A+): fundamental phase angle vs voltage 0, range (−18000, +18000]; voltage 0 reads 0; `0x8000` = no angle (that input, or voltage 0, below 50 V). Forward sequence: v1 ≈ −12000, v2 ≈ +12000 |

---

### CURRENT (I + P channels) — base 0x1000, stride 0x40, c = 0..63 (gate on N_CURRENT)

| Off | Name | Type | Unit |
|---|---|---|---|
| +0x00 | I_RMS | i32 | µA |
| +0x04 | P | i32 | mW, signed (import > 0) |
| +0x08 | Q | i32 | mvar, signed |
| +0x0C | S | i32 | mVA |
| +0x10 | PF | i16 | Q10, signed |
| +0x14 | I_PEAK | i32 | µA (reads 0 — not computed on current builds) |
| +0x18 | E_POS | i64 | per-window import energy, mW·samples |
| +0x20 | E_NEG | i64 | per-window export energy, mW·samples (≤0) |
| +0x28 | THD_I | u16 | centi-% (gated: 0 on I²C / sentinel on Modbus native) |
| +0x2A | PF1 | i16 | displacement PF, Q10 (gated, as THD_I) |
| +0x2C | Q1 | i32 | displacement reactive, mvar (gated, as THD_I) |

---

### TOTALS — base 0x0300, stride 0x40, k = 0..3; same field layout as CURRENT

k = 0 grand total (all channels) · k = 1..3 per-phase totals (phase = channel when
N_VOLTAGE = N_CURRENT; single-voltage builds sum all channels into k = 1, k = 2/3 read 0; totals
PF = P/S). **TOTALS carry I/P/Q/S/PF only — the E_POS/E_NEG field positions read 0** (energy lives
in the per-channel blocks and the cumulative counters). Grand import = positive part of the
aggregate power per window (net at the service entry), which is not the same as the sum of
per-channel imports — both are available. Note: the first field is I (µA); total P sits at +0x04.
Over Modbus the totals P/Q/S are i64.

---

### Calibration blocks (factory-written; see [Calibration](17_config_calibration.md))

| Block | Base | Stride | Fields |
|---|---|---|---|
| CAL_V | 0x0400 | 0x08, v=0..7 | U_GAIN i32 (codes→mV, Q16.16) · NF_U u16 · THD_FLOOR_U u16 |
| CAL_I | 0x0440 | 0x08, c=0..63 | I_GAIN i32 (codes→µA, Q16.16) · NF_I u16 · THD_FLOOR_I u16 |
| CHMAP_ASSOC | 0x0640 | 1, c=0..63 | reserved — current channel → voltage channel index (reads 0 on current builds) |
| CHMAP_MODEL | 0x0680 | 1, c=0..63 | reserved — per-channel CT model metadata (reads 0 on current builds) |

---

### Snapshot banks — base 0x2000, **stride 0x800**, b = 0..7 *(capability-gated: bit 8)*

Period snapshots armed by `PERIOD_ARM` and taken by command 0x03; banks are read **directly** at
`0x2000 + b·0x800` (LATCH_BANK selects the command target only, it is not a read multiplexer).
Fields: P/Q/S (i32, period averages) · E_POS/E_NEG (i64, mW·samples) · NSAMP u32 · WIN_CNT u32 ·
MAX_P/MAX_I/MAX_U/MIN_U (i32) · per-channel energy sub-block at +0x40 + c·0x10 (E_POS/E_NEG i64).
**Current firmware reports CAPABILITIES bit 8 = 0 — the block is unmapped (command 0x03 → unknown
command) and energy accounting uses the cumulative counters below.**

---

## Energy / time region 0x0800–0x0FFF

### ENERGY (grand cumulative) — base 0x0800 *(ONDEVICE_ENERGY capability)*

| Addr | Name | Type | Unit |
|---|---|---|---|
| 0x0800 | E_ACT_IMP | i64 | Wh |
| 0x0808 | E_ACT_EXP | i64 | Wh (magnitude ≥ 0) |
| 0x0810 | E_ACT_NET | i64 | Wh (native profile only) |
| 0x0818 | E_REA_IMP | i64 | varh |
| 0x0820 | E_REA_EXP | i64 | varh |
| 0x0828 | E_ACT_IMP_MWH | i64 | mWh |
| 0x0830 | E_ACT_EXP_MWH | i64 | mWh |
| 0x0838 | E_ACT_NET_MWH | i64 | mWh |
| 0x08A0 | UPTIME | u32 | s |
| 0x08A4 | RTC_EPOCH | u32 | unix epoch (read = live clock) |
| 0x08A8 | RTC_BCD | 14 B | calendar mirror (reads 0 on modules without a battery clock) |
| 0x08C0 | TIME_SET | u32 (w) | write epoch, then COMMAND 0x06; epoch < 1 600 000 000 (2020-09-13) → ERROR 0xFC |
| 0x08D0 | E_ACT_IMP_UWH | i64 | µWh (image 0x7A+, capability bit 15) |
| 0x08D8 | E_ACT_EXP_UWH | i64 | µWh (magnitude ≥ 0) |
| 0x08E0 | E_ACT_NET_UWH | i64 | µWh; = IMP − EXP (the floor-to-mWh rule applies to IMP/EXP only) |
| 0x08E8 | E_REA_IMP_UVARH | i64 | µvarh (image 0x7B+; varh = floor(µvarh / 1 000 000)) |
| 0x08F0 | E_REA_EXP_UVARH | i64 | µvarh (magnitude ≥ 0). Grand only — there is no per-channel reactive energy |

---

### Life-time counter management

| Addr | Name | Type | Acc | Meaning |
|---|---|---|---|---|
| 0x0840 | LT_GEN | u32 | r | life-time generation counter (++ on reset/adopt) |
| 0x0844 | LT_KEY | u32 | w | single-shot key for keyed commands; consumed by any command |
| 0x0848 | LT_ADOPT | 32 B | rw | staging for counter adopt/import (4×i64) |

---

### NVM / store diagnostics

| Addr | Name | Type | Meaning |
|---|---|---|---|
| 0x0870 | LTA_STATUS | u8 | energy store: 0 OK · 1 new · 2 FAULT · 3 CORRUPT |
| 0x0871 | LTA_SRC | u8 | restore source at boot: 0 none · 1 NVM · 2 retention · 3 legacy (4 reserved) |
| 0x0872 | CAL_SRC | u8 | calibration source: 0 defaults · 1 flash · 2 NVM · 3 stored cal not for this firmware (defaults in use) |
| 0x0873 | LTG_STATUS | u8 | per-channel energy store: 0 ok · 1 new · 2 FAULT · 3 restarted at 0 (transient — clears after the first per-channel write, ~1 s) |
| 0x0874/0x0876/0x0878 | EE_RD_ERR / EE_WR_ERR / EE_RECOVER | u16 | NVM I/O counters |
| 0x087A | LTA_AGE_S | u16 | seconds since the last successful energy write |
| 0x087C/0x0880 | LTA_OK_CNT / LTA_FAIL_CNT | u32 | energy persist counters |
| 0x0892 | KEY_STATE | u8 | record signing: 0 not provisioned · 1 key present · 0xFF build without it |
| 0x0894 | KEY_ID | u32 | provisioned key id (identifier, not a secret) |
| 0x0898 | NVM_REJ_CNT | u16 | NVM records rejected as foreign since boot |
| 0x089A | NVM_REJ_ADDR | u16 | address of the first rejected record |

---

### GROUP_ENERGY — base 0x0900 (Wh) and 0x0A80 (mWh), stride 0x10, g = 0..15 / 0..7

Per-group cumulative energy (E_IMP i64 / E_EXP i64). On current firmware the groups are implicit:
**group g = current channel g** (on three-phase members — phase L(g+1)), g < N_CURRENT. Persisted
every second; **informative** — the grand ENERGY block is the canonical cumulative counter; the per-channel
counters restart at 0 with `LT_GEN` or on new/foreign storage (`LTG_STATUS` = 3).

---

### GROUP_ENERGY_UWH — base 0x0C80, stride 0x10, g = 0..7 *(image 0x7A+, capability bit 15)*

The same per-group counters at µWh resolution: +0x00 E_IMP (i64 µWh) · +0x08 E_EXP (i64 µWh).
Published in the same snapshot as the mWh fields (mWh = floor(µWh / 1000)); the same restart and
generation rules apply ([Measurement & energy](16_measurement_energy.md#cumulative-counters)).

---

### METER_GROUP (group configuration) — base 0x0A00, stride 0x08, g = 0..15 *(reserved)*

**Not implemented on current firmware** (reads 0, not writable): reserved group configuration —
CH_A/CH_B/CH_C (channel indices, 0xFF unused) · V_ASSOC · VIRT_ID · FLAGS (enable, per-channel
invert). Today's virtual unit ids are derived per channel from `CFG_VIRTUAL_IDS`
([Modbus](12_transport_modbus.md#virtual-unit-ids)).

---

## Security block 0x0B00–0x0BFF

| Addr | Name | Type | Meaning |
|---|---|---|---|
| 0x0B00 | SEC_CERT | 128 B | device certificate (0xFF when absent): magic `CRT1` · version · variant · hw_rev · UID · batch · issue date · device public key · manufacturer key id · signature (covers certificate bytes 0..63) |
| 0x0B80 | SEC_CHALLENGE_SIG | 64 B rw | write a 32-B challenge → COMMAND 0x0F → read the 64-B signature |
| 0x0BC0 | SEC_ATT_DATA | 24 B | attested data: firmware version · LT generation · import Wh · export Wh |
| 0x0BE0 | SEC_CERT_STATE | u8 | 0 none · 1 OK · 2 bad · 3 not checked yet (check runs ~3 s after boot) |
| 0x0BE1 | SEC_ATT_STATE | u8 | 0 none · 1 done · 2 refused |
| 0x0BE2 | SEC_ATT_MS | u16 | last attestation duration, ms (~55) |
| 0x0BE4 / 0x0BE8 | SEC_IMG_KEY_ID / SEC_MFR_KEY_ID | u32 | image / manufacturer key ids |
| 0x0BEC–0x0BF3 | SEC_ANCHOR_* / SEC_ROLLBACK | — | **reserved — rollback-anchor mechanism not implemented on current firmware** (read 0) |
| 0x0BF4 | SEC_CERT_MS | u16 | certificate check duration, ms |

Attestation: the device signs a fixed ASCII domain-separation tag (the `ATTEST_MSG_TAG` constant of the register header) `‖ challenge(32) ‖ UID(12) ‖ attested-data(24)` with
Ed25519; the host verifies the certificate against the manufacturer public key, then the signature
against the device public key from the certificate.

---

## History / event-log query 0x0C00–0x0C6F

| Addr | Name | Type | Acc | Meaning |
|---|---|---|---|---|
| 0x0C00 | HIST_PERIOD | u8 | rw | 0 = 15-min · 1 = hourly · 2 = **event log** |
| 0x0C01 | HIST_TYPE | u8 | rw | reserved selector (ignored) |
| 0x0C02 | HIST_FLAGS | u8 | r | bit0 record valid · bit1 from RAM; cleared when HIST_READ is accepted, filled at its completion |
| 0x0C04 | HIST_BACK | u16 | rw | records back from newest (0 = newest) |
| 0x0C06 | HIST_COUNT | u16 | r | records in the ring |
| 0x0C08 | HIST_CAP | u16 | r | ring capacity: the number of records the module keeps (0 for the 15-min selection) — read it from this register |
| 0x0C0A | HIST_RECSZ | u16 | r | record size: 80 (history) / 32 (event log) |
| 0x0C0C / 0x0C10 | HIST_OLDEST / HIST_NEWEST | u32 | r | oldest / newest record timestamp |
| 0x0C20 | HIST_REC | 80 B | rw | read result / import staging |

Record layouts (history and event records) are in
[Measurement & energy](16_measurement_energy.md#history-records).

---

## Modbus mapping

| Content | I²C | Modbus |
|---|---|---|
| Discovery block | 0x0000-core | input 0x6000–0x601F (0x6019–1C CPU diag · 0x601D–1E LOST_FRAMES · 0x601F WIN_SEQ) · 0x6020 PHASE_SEQ (image 0x7A+) |
| Voltage channels | 0x0200 + v·0x20 | input 0x6100 + v·0x10: +0 U_RMS (2 regs) · +2 U_PEAK (2) · +4 THD_U · +5 ANGLE (image 0x7A+) |
| Current channels | 0x1000 + c·0x40 | input 0x6200 + c·0x20: +0 I (2) · +2 P (2) · +4 Q (2) · +6 S (2) · +8 PF · +9 I_PEAK (2) · +0xB THD_I · +0xC PF1 · +0xD Q1 (2). Per-channel E_POS/E_NEG are **not** on Modbus; reserved slots read 0 |
| Cumulative energy + time | 0x0800 block | input 0x6B00 (Wh) · 0x6B14 (mWh) · 0x6BB0/0x6BB4/0x6BB8 (µWh imp/exp/net, 4 regs each, image 0x7A+) · 0x6BAC / 0x6BBC (µvarh imp / exp, image 0x7B+ — **not contiguous** with the µWh regs) · UPTIME 0x6BA0 · RTC 0x6BA2+ |
| Per-group energy | 0x0900 / 0x0A80 | input 0x6B20 + g·0x10 (+0/+4 Wh · +8/+0xC mWh), g=0..7 |
| Per-group energy, µWh | 0x0C80 + g·0x10 | input 0x6BC0 + g·8: +0 E_IMP · +4 E_EXP (i64), g=0..7 (image 0x7A+) |
| Totals (P/Q/S as i64) | 0x0300 + k·0x40 | input 0x6C00 + k·0x20: +0 I (2) · +2 P (4) · +6 Q (4) · +0xA S (4) · +0xE PF |
| Config | 0x0070–0x0085 | holding 0x8000–0x800C (order: unit id · baud · frame · tx-delay · word order · time-auth · persist-s · virtual-ids · class · ct-model · tz-offset · LED · starting current, image 0x7B+). `CFG_I2C_ADDR` has **no** Modbus register |
| Time set | 0x08C0 + cmd 0x06 | holding 0x8100 (hi) / 0x8101 (lo) + COMMAND 0x06 |
| Command window | 0x0060-block | holding 0x8110 LATCH_BANK · 0x8111 PERIOD_ARM · 0x8112 UNLOCK · 0x8113 COMMAND · 0x8114 ERROR_RB (0xFFFF = busy) · 0x8115 PD_VALID · 0x8116 CMD_STATE · 0x8117 CMD_SEQ |
| Address by UID | 0x00E0–0x00ED | holding 0x8118–0x811D (UID byte pairs) · 0x811E (new I²C addr hi / unit lo) |
| Events | 0x00C0–0x00CE | holding 0x8120 EVENT_FLAGS · 0x8121 EVENT_MASK · 0x8122–23 THRESH_I · 0x8124–25 THRESH_P · 0x8126 RESET_CAUSE · 0x8127 BOOT_COUNT |
| Life-time keyed ops | 0x0840–0x0867 | holding 0x8140–41 LT_GEN · 0x8142–43 LT_KEY · 0x8144–53 LT_ADOPT |
| History query | 0x0C00–0x0C13 | holding 0x8160–0x816A (same field order) |
| History record | 0x0C20 (80 B) | holding 0x8180–0x81A7 (byte pairs; writable for import) |
| Firmware update | 0x0140–0x0180 | holding 0x8300–0x831C ([Firmware update](18_firmware_update.md)) |
| Firmware page buffer | 0x0500 (256 B) | holding 0x8380–0x83FF (byte pairs) |
| Security window | 0x0B00–0x0BFF | holding 0x8400–0x847F (byte stream: cert 0x8400 · challenge/signature 0x8440 · attested data 0x8460 · diag 0x8470) |
| RO mirror of native input | — | holding 0xA000+ = input + 0x4000 |

u32 values in the holding windows = 2 registers, **high word first**; i64 = 4 registers.
