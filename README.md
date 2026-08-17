# BERT fatal firmware error record — ASUS TUF GAMING B860M-PLUS WIFI / Intel Core Ultra 7 270K

Supporting evidence for Intel Community case **DGD8008** and the corresponding ASUS ROG forum thread.

## System

| Component | Detail |
|---|---|
| Motherboard | ASUS TUF GAMING B860M-PLUS WIFI, BIOS 3011 |
| CPU | Intel Core Ultra 7 270K (LGA1851, Arrow Lake-S) |
| RAM | 32 GB DDR5 @ 4800 MT/s JEDEC (XMP disabled) |
| GPU | NVIDIA RTX 3070 Ti, driver 580.173.02 |
| Storage | Samsung 980 Pro, Samsung 970 EVO Plus (NVMe) |
| PSU | Super Flower LEADEX III Gold 850W |
| OS | Linux Mint 22.3, kernel 7.0.0-28-generic |

## Crash event

**2026-08-17, approx. 22:24:53 (+07).** Hard freeze requiring manual power-off.
Uptime at crash: 15 minutes.

Kernel signature:

```
watchdog: BUG: soft lockup - CPU#5 stuck for 26s!
```

Conditions at time of crash, from continuous turbostat logging (final sample 22:24:48):

- Package power 80.9 W, PL1 enforced at 125 W via RAPL
- Package temperature 69 °C, peak core temperature 80 °C
- Average frequency 1133 MHz, busy 23.7 %, peak core 4779 MHz
- SMI count 0 throughout the session

No MCE, AER, or EDAC events. No NVIDIA Xid errors. No i915 errors.

This signature differs from earlier crashes in this case, which showed
simultaneous multi-CPU stalls in `smp_call_function_many_cond`. Here a single
CPU stalled while the platform froze completely.

## BERT record

Extracted from `/sys/firmware/acpi/tables/data/BERT` on the post-crash boot.

| Field | Value |
|---|---|
| Section type GUID | `81212a96-09ed-4996-9471-8d729c8e69ed` (UEFI Firmware Error Record Reference) |
| Error severity | 1 (Fatal) |
| Record type | 2 (SoC firmware error record) |
| Record identifier GUID | `8f87f311-c998-4d9e-a0c4-6065518c4f6d` |
| Generic Error Status Block data length | 0xADE0 (44,512 bytes) |
| Vendor-defined payload | 8,224 bytes |
| Total record size | 44,532 bytes |
| FRU ID / FRU text / timestamp | all zero |

The record is a **firmware error record** — not Processor Generic, not Processor
Specific x86/x64, not Platform Memory, not PCI Express. This is consistent with
the persistent absence of MCE, AER, and EDAC events throughout this case: the
fault has never been reported through those paths.

The kernel logs `BERT: [Hardware Error]: Skipped 1 error records` because the
record exceeds the printer's size limit, not because it fails to decode.

The 8,224-byte vendor payload requires vendor-side decoding.

### BERT is logged per crash, not latched

After a clean shutdown and reboot with no intervening crash, the BERT ACPI table
is **absent entirely** — no table, no data region, no dmesg entry. Each
`Total records found: 1` therefore represents a distinct fatal error freshly
logged by firmware after that specific crash, rather than one stale record
re-presented at every boot.

## Hardware configuration at time of crash

A full mechanical rebuild was completed the day before this crash:

- Thermalright LGA1851-BCF anti-bend contact frame removed
- CPU reseated using stock Intel retention hardware and stock backplate
- Fresh thermal paste applied
- Both DIMMs removed, contacts cleaned with isopropyl alcohol, reseated in A2/B2
- CMOS battery removed and CMOS cleared
- NVMe thermal pad residue cleaned (confirmed benign silicone bleed)
- BIOS set to Optimized Defaults with Intel Default Settings

The crash recurred on this configuration.

## Note on power limits

PL1 is not applied by the BIOS despite the menu displaying 125 W — the RAPL MSR
reads 200 W until manually overridden. A systemd oneshot unit now forces 125 W at
every boot. 125 W was confirmed active at the time of this crash.

## Persistent ACPI firmware errors

Present at every boot, identical on BIOS 3011 and 3020:

```
ACPI BIOS Error (bug): Could not resolve symbol
  [\_SB.PC02.RP16.PXSX._DSM.IDSM], AE_NOT_FOUND
ACPI BIOS Error (bug): Failure creating named object
  [\_SB.PC00.RP01.PXSX._DSM.USRG], AE_ALREADY_EXISTS
```

Both reference PCI Express root ports.

## Files

| File | Description |
|---|---|
| `bert_20260817.bin` | Raw BERT data region from `/sys/firmware/acpi/tables/data/BERT` (44,532 bytes) |
| `bert_20260817.hex` | Hex dump of the above |
| `bert_table_20260817.bin` | BERT ACPI table header from `/sys/firmware/acpi/tables/BERT` (48 bytes) |
| `crashboot_20260817.txt` | Full kernel log of the crash boot |
| `turbostat_crash_20260817.txt` | turbostat telemetry, 22:20:00–22:25:30 |
| `SHA256SUMS.txt` | Checksums for the evidence files |

`bert_20260817.bin` SHA-256:
`7842babc17c42b6e36106f93ffacfe21ef1d84f0719c941d679d76ea209c0d26`

## Monitoring currently armed

- `kernel.softlockup_panic = 1`, `kernel.hardlockup_panic = 1`, `kernel.panic = 30`
- kdump armed and reporting `ready to kdump`
- iTCO watchdog module loaded via dedicated systemd unit, 60 s timeout
- Continuous turbostat logging with timestamps, 7-day logrotate retention
- rasdaemon running

### Open question on the watchdog

The iTCO watchdog did not reset the platform during this crash, and approximately
two minutes elapsed before manual power-off. Because only one CPU stalled, the
watchdog daemon may have continued servicing `/dev/watchdog0` from the remaining
23 cores. This cannot currently be distinguished from a genuine reset-path
failure.
