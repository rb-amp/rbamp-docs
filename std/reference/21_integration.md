# Host integration

How to run an STD module from your own code — with our libraries or with nothing but an I²C/Modbus
master.

## With the rbAmp libraries

One client library per platform covers the whole line; channel count and features are discovered at
run time:

- **Arduino** — STD support: discovery, realtime, config, events/DRDY, OTA, history; the
  micro-resolution energy, starting-current and phase-angle/sequence features from v1.6.1.
- **ESP-IDF** — component with STD support from v1.6.0; the image 0x7A/0x7B features from v1.7.0.
- **ESPHome / Home Assistant** — external component; on-device energy counters, starting current
  and phase diagnostics from v1.8.0.
- **Python** — STD support from v1.6.0: one client class, the module line is detected at connect.
  Validated on standard Linux Python (smbus2); the MicroPython STD path is implemented but not yet
  hardware-validated.

The libraries cover these flows (discovery gating, busy/err polling, prepared-image OTA,
energy-delta accounting), so prefer them where the platform allows.

The image 0x7A/0x7B features at a glance — every call degrades gracefully on modules that do not
publish the feature (no bus traffic, or values derived from the coarse counters):

| Feature | Arduino | ESP-IDF | ESPHome (YAML) | Python |
|---|---|---|---|---|
| Micro-resolution energy (µWh / µvarh fields are always usable; a `fine` flag says whether they are the module's own counters) | `stdReadEnergy()` → `RbAmpStdEnergy` `*_uwh` / `*_uvarh` fields, `stdEnergyDelta()` | `rbamp_std_energy_t` `*_uwh` / `*_uvarh` fields, `rbamp_std_energy_delta()` | `energy_total` · `energy_exported_total` · `energy_net` · `reactive_energy` / `reactive_energy_exported` | `read_std_energy()` → `RbAmpStdEnergy` `*_uwh` / `*_uvarh` fields, `RbAmp.std_energy_delta()` |
| Starting current for energy | `stdSetEnergyStartCurrent()` / `stdReadEnergyStartCurrent()` | `rbamp_std_set_start_current()` / `rbamp_std_get_start_current()` | `start_current_ua:` | `set_start_current()` / `get_start_current()` |
| Voltage phase angle | `stdReadVoltage(v)` → `angle_deg` / `angle_valid` | `rbamp_std_voltage_t.angle_deg` / `angle_valid` | `voltage_angle_b` / `voltage_angle_c` | `read_std_voltage(phase)` → `angle_deg` / `angle_valid` |
| Phase sequence | `stdReadPhaseSequence()` | `rbamp_std_read_phase_seq()` | `phase_sequence` | `read_phase_sequence()` |

If your application persists a library energy snapshot (NVS, EEPROM, file), store the individual
fields you need in your own fixed format, not the struct as a raw blob — the snapshot structures
gain fields across library versions.

## Without our libraries

Everything the libraries do is plain register I/O. The protocol is documented and stable; map
changes are versioned (`VERSION`) and feature-gated (`CAPABILITIES`). **The canonical step-by-step
flows (init, command, config, address change, energy, history, OTA, unlock) are in
[Sequences](22_sequences.md)** — below are the shortest working patterns.

### Minimal I²C bring-up (pseudocode)

```
# discovery (once)
id   = read(0x0000, 2)            # expect 'S','T'
ver  = read(0x0003, 1)            # map version, expect >= 0x36
caps = read_u16(0x0004)
nv   = read(0x0007, 1); ni = read(0x0008, 1)
uid  = read(0x00F4, 12)           # device identity for your database
fwv  = read_u32(0x0114)           # running image version

# clock (after every module reset: STATUS bit4 == 0)
write_u32(0x08C0, unix_epoch)     # TIME_SET
write(0x0060, 0x06)               # SET_TIME; then poll CMD_STATE 0x006A bit0 -> 0, check ERROR == 0

# realtime poll (every 0.2..1 s, or on the DRDY edge)
u_mv  = read_i32(0x0200 + v*0x20)             # per voltage channel
i_ua  = read_i32(0x1000 + c*0x40)             # per current channel
p_mw  = read_i32(0x1000 + c*0x40 + 0x04)      # signed
pf    = read_i16(0x1000 + c*0x40 + 0x10)      # /1024
freq  = read_u16(0x0010)                      # /100 Hz
tot_p = read_i32(0x0300 + 0x04)               # grand active power
```

### Energy accounting pattern

```
# at each interval boundary (your clock):
blk = read(0x0800, 0x44)          # ONE transaction <= 128 B: Wh + mWh counters + LT_GEN — coherent
e_imp = i64(blk, 0x28); e_exp = i64(blk, 0x30)   # grand import/export, mWh
gen   = u32(blk, 0x40)            # LT_GEN (0x0840)
up    = read_u32(0x08A0)          # UPTIME

interval_import = e_imp - last_imp          # delta of cumulative counters
if gen != last_gen: re-anchor, no delta     # counter was reset/adopted (event-logged)
if up < last_up:    delta = max(0, delta)   # restart: ~last second of energy may be missing; re-feed time
```

Read the energy block in **one transaction of ≤ 128 bytes** — i64 fields read in separate
transactions can tear across a publication ([I²C transport](11_transport_i2c.md#timing-and-robustness)).

Never integrate power over time on the host — the module's counters are gap-compensated and
persisted; your integral is neither. Details: [Measurement & energy](16_measurement_energy.md#energy).

### History back-fill after your own outage

```
write(0x0C00, 1)                  # HIST_PERIOD = hourly
write_u16(0x0C04, n)              # n records back from newest
write(0x0060, 0x08)               # HIST_READ
poll CMD_STATE (0x006A) bit0 -> 0
err = read(0x0002, 1)             # 0xFC = no such record
flags = read(0x0C02, 1)           # bit0 = record valid
rec   = read(0x0C20, 80)          # one transaction; see record layout
```

Records carry the **absolute** life-time Wh at period end plus per-phase period energy, so gaps
close exactly ([layout](16_measurement_energy.md#history-records)).

### Health poll

See the [diagnostics checklist](19_diagnostics.md#health-checklist-for-a-supervising-master) — six
reads cover liveness, storage health, restarts and counter generations.

## Modbus integration without libraries

- **As a ready meter (SDM profile):** point the inverter/SCADA at the module as an Eastron-type
  SDM630 meter — per-phase V/I/P, total P, frequency, import/export kWh at the canonical float32
  input addresses. No register programming needed.
- **Native profile:** FC04 reads at 0x6000 (discovery), 0x6100/0x6200 (channels), 0x6B00+ (energy),
  0x6C00 (totals); configuration and commands over FC16 at 0x8000+ with UNLOCK for state-changing
  commands. Full map: [register map](13_register_map.md#modbus-mapping); rules:
  [Modbus transport](12_transport_modbus.md).
- FC03-only masters read the native data through the 0xA000+ mirror (input + 0x4000).

## Integration rules worth repeating

1. Gate channels on `N_VOLTAGE`/`N_CURRENT`, features on `CAPABILITIES`, map layout on `VERSION`.
2. Identify modules by **UID**, not by bus address.
3. Re-arm RAM state (event mask, thresholds, clock) after any detected module restart.
4. Tolerate NACK/clock-stretching during flash-touching commands; respect settle times.
5. Energy: counter deltas with generation/restart handling — never host-side P·dt.
6. Firmware updates: stream the prepared release file as-is; check `BOARD_ID` first.
