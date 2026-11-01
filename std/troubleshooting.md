# 07 · STD troubleshooting

Symptoms, what to read, and what it means. Registers are in the
[diagnostics reference](reference/19_diagnostics.md).

## The module does not answer

- **Wrong transport.** The interface mode switch selects I²C or Modbus **at boot**; after moving it, restart
  the module (power cycle or RST).
- **I²C address.** Default `0x50`. After an address change the module answers only at the new address; the
  unique `UID` identifies it.
- **Modbus settings.** Defaults are unit 1, 9600 8N1. A mismatch in baud, frame format or unit id gives no
  answer. Read `LAST_EXCEPTION` (0x0158 / holding 0x830A) after an error answer.
- **After a firmware activation reboot** allow about 4 s before the module reappears.
- **Pull-ups (I²C) and termination / bias (RS-485)** — check them first on a new installation.

---

## Which firmware do I have

Read `FW_VER` (0x0114, u32): the high word and the low word in decimal are the version, for example `0x0001007E` = STD firmware 1.126. The firmware is updated over the working bus (OTA, I²C or Modbus); see the [firmware update protocol](reference/18_firmware_update.md).

---

## NACKs and retries on I²C

During flash writes (saving configuration, firmware update) the module may stretch the clock for several
milliseconds or NACK briefly. Set the master's SCL timeout to at least 10 ms and retry until the
command's settle time has passed instead of aborting on the first NACK.

---

## All readings are zero

- Right after boot: measurements read 0 until the first window is published (~100 ms); allow ~1 s.
- `STATUS` bit 2 (mains ok) clear: the channel-0 voltage is below ~50 V — no mains, or the voltage input
  is not connected.
- `STATUS` bit 1 (sensor ok) clear, or `FRAME_CNT` (0x0020) not increasing: the measurement path is not
  delivering frames; the red LED stays on.

---

## Current or power looks wrong

- **A sensor on the wrong channel.** Each sensor is calibrated with the channel it is labelled for.
- **Sensor direction.** Follow the arrow marking on the sensor and the board. A reversed sensor gives a power
  with the opposite sign.
- **No load.** A small reading of a few tenths of a milliamp with nothing connected is the reading floor,
  not a load.
- **Reversed sensor or reversed voltage.** P is negative and the energy goes to export. There is no software sensor inversion; turn the sensor or the voltage input.
- **Voltage channel phase.** On a three-phase member read `PHASE_SEQ` (0x0012): 1 = forward, 2 = reverse,
  3 = abnormal (two inputs on one phase). A wrong phase order gives 2; on a correct three-phase supply L2 reads about −120° and L3 about +120° relative to L1.

---

## Energy does not increase

- The channel's current is below the starting current (`CFG_START_UA`, default 400 µA): nothing is added
  by design.
- A counter went back after a restart by up to about a second of energy: normal; take the difference as
  `max(0, new − last)`.
- `LT_GEN` changed: someone reset or adopted the counters. Check the event log.

---

## `LTA_AGE_S` is large

Energy is written only when the counters changed, so large values at idle are normal. Judge the energy
storage by `LTA_STATUS` and `LTA_FAIL_CNT`, not by the age.

---

## DRDY does not pulse

- No new window: the fallback publish every 500 ms has no pulse.
- DRDY needs a pull-up on the master side (the module has none; 10 kΩ to the master's 3.3 V is recommended).
- DRDY held **low** is the alarm level: an event bit is set in both `EVENT_FLAGS` and `EVENT_MASK`. Clear
  the flag (write 1) or unmask it.

---

## Snapshot banks read zero or old data

- `CAPABILITIES` bit 8 is 0: the module has no banks; use counter differences.
- Read the bank only after `WIN_SEQ` advanced by 2 following the command; `PD_VALID` is not a freshness
  flag.
- A reset clears the banks; arm them again.

---

## Health checklist

Poll cheaply and alert on: `STATUS` (cal ok, sensor ok, mains ok set; storage fault clear; time valid),
`FRAME_CNT` advancing, `RING_DROP` = 0, `LOST_FRAMES` flat, unexpected `BOOT_COUNT` steps, `LT_GEN`
unchanged, `LTA_STATUS` ≤ 1. See the
[full checklist](reference/19_diagnostics.md#health-checklist-for-a-supervising-master).
