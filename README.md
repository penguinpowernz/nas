# NAS research

The purpose is to research how to setup/run my network attached storage system.

- Preferably 4TB usable space
- Recover from >2 disk fails 
- NAS will not be powered on all the time, only when needed (I'm paying almost 50c/kWh here)
- Need to stagger drive startup
- At least 6x2TB disks wanted
- 2 RAID mirrors, one for data, one for backup
- Maintainer guide with hard copy
- Debian based OS

## Target PC

- Intel(R) Core(TM) i7-4790 CPU @ 3.60GHz
- 16GB DDR3 RAM
- Nvidia Quadro K200
- 6x SATA 6GB/s ports on Motherboard
- Fractal R3 with 8x 3.5" drive slots
- Lots of cooling

### Eqiupment coming soon

- $150 StarTech 4x PCIe card with 6 SATA Ports using ASM1611
- $110 2x 2TB (5400rpm)
- $165 3x 2TB (7200rpm)

### RAID Plan

#### Option 1: 4TB space, with a backup raid, handle 2-4 or 3-6 drive failure

The first two mirrors will be placed into an LVM volume group to give 4TB of useable
 space, a more flexible kind of RAID 10.

| Mirror | Count | Type | Drives | Purpose |
|---|---|---|---|---|
| /dev/md0 | 4 | RAID1 | W1H31BLJ, WD-WCC4M7YANCD5, Z52BBV0P, WD-WCC4M1RYFSH9 | LVM PV1 (Data) |
| /dev/md1 | 4 | RAID1 | W520VJFQ, WFL3ZBBC, Z1E7BC0E, 5YD5VWL1 | LVM PV2 (Data) |
| /dev/md3 | 4 | RAID1 | ZFL0TF34, Z1E46C17, Z1E9K96R, 5YD5PQE7 | OS and restic snapshots |

| PVs | VGs | LVs | Size |
|---|---|---|---|
| /dev/md0 | raid10 | data | 4T |
| /dev/md1 |  |  | |
| /dev/md3 | system | root | 80G |
| | | swap | 20G |
| | | bkp | 1.8T |
| | | boot | 1G |

> Note: I didn't want to do real RAID10 cos you can't grow or shrink the mirrors

> **Important:** LVM stripes across md0 and md1 (RAID0 at the VG level). Each mirror individually tolerates up to 3 drive failures, but if an entire mirror is lost, **all data in the VG is lost** — there is no VG-level redundancy. This is the key tradeoff vs. real RAID10. Both data mirrors must be kept equally healthy; see the Optimum RAID layout section for how CAUTION drives are distributed to reflect this.

#### Option 2: big redundo

12 disks in a RAID1 - 12 copies of everything, full redunancy.

Then an LVM goes over that:

| PVs | VGs | LVs | Size |
|---|---|---|---|
| /dev/md0 | system | root | 80G |
| | | swap | 20G |
| | | data | 1.8T |
| | | boot | 1G |

> Note: the lack of a backup drive in this could lead to data loss

## Tests to run

| Test | Tool | Command | What it checks |
|---|---|---|---|
| SMART health + attributes | `smartctl` | `sudo smartctl -a /dev/sdX` | Overall drive health, error counts, temperature, reallocated sectors, self-test history |
| SMART extended self-test | `smartctl` | `sudo smartctl -t long /dev/sdX` then `sudo smartctl -a /dev/sdX` | Full surface scan via drive firmware; takes several hours |
| Bad blocks (destructive read-write) | `badblocks` | `sudo badblocks -wsv -b 4096 -c 262144 /dev/sdX 2>&1 \| tee sdX_bb.log` | Writes 4 patterns (0xaa, 0x55, 0xff, 0x00) then reads back to detect media errors |
| SMART conveyance self-test | `smartctl` | `sudo smartctl -t conveyance /dev/sdX` then `sudo smartctl -a /dev/sdX` | Short firmware test designed to catch damage from transport/handling |

## Disk pool

These are the drives I have available (with some more on the way):

### Drive comparison


| Size | Model | Serial | SH | SE | BB | SC | Hours | Status |
|---|---|---|---|---|---|---|---|---|
| 2TB | Seagate NAS ST2000VN000-1H3164 | W1H31BLJ | :white_check_mark: | :white_check_mark: | 0 | — | ~6.4 yrs | OK |
| 2TB | :warning: WD Red WD20EFRX-68EUZN0 | WD-WCC4M1RYFSH9 | :white_check_mark: | :white_check_mark: | 16 | — | ~6.4 yrs | CAUTION |
| 2TB | WD Red WD20EFRX-68EUZN0 | WD-WCC4M7YANCD5 | :white_check_mark: | :white_check_mark: | 0 | — | ~6.4 yrs | OK |
| 2TB | Seagate NAS ST2000VN000-1HJ164 | W520VJFQ | :white_check_mark: | :white_check_mark: | 0 | — | ~6.4 yrs | OK |
| 2TB | Seagate Barracuda ST2000DM008-2FR102 | WFL3ZBBC | :white_check_mark: | :white_check_mark: | — | :white_check_mark: | ~0.3 yrs | PENDING |
| 2TB | Seagate IronWolf ST2000VN004-2E4164 | Z52BBV0P | :white_check_mark: | :white_check_mark: | — | :white_check_mark: | ~0.06 yrs | PENDING |
| 2TB | Seagate Barracuda ST2000DM008-2FR102 | ZFL0TF34 | :white_check_mark: | :white_check_mark: | — | :white_check_mark: | ~4.2 yrs | PENDING |
| 2TB | :warning: Seagate Barracuda Green ST2000DL003-9VT166 | 5YD5PQE7 | :white_check_mark: | :white_check_mark: | :hourglass: | — | ~0.7 yrs | PENDING |
| 2TB | :warning: Seagate Barracuda Green ST2000DL003-9VT166 | 5YD5VWL1 | :white_check_mark: | :white_check_mark: | :hourglass: | — | ~5.7 yrs | PENDING |
| 2TB | :warning: Seagate Barracuda ST2000DM001-1CH164 | Z1E9K96R | :white_check_mark: | :white_check_mark: | :hourglass: | — | ~0.6 yrs | PENDING |
| 2TB | Seagate Barracuda 7200.14 ST2000DM001-1CH164 | Z1E46C17 | :white_check_mark: | :hourglass: | — | — | ~1.5 yrs | PENDING |
| 2TB | Seagate Barracuda 7200.14 ST2000DM001-1CH164 | Z1E7BC0E | :white_check_mark: | :hourglass: | — | :white_check_mark: | ~3.0 yrs | PENDING |

I have about 9x 1TB drives but probably won't use them.

## Optimum RAID layout

The disks must be assigned to mirrors in a way that ensures we spread out potential failures. This is
informed by the health data and tests surfaced by this document.  Assuming RAID plan 1, the optimum
pool layout is as follows.

| RAID1 | Serial | Reasoning |
|---|---|---|
| /dev/md0 | W1H31BLJ | Fully tested, 0 bad blocks, clean error log — strong anchor for this mirror |
| | WD-WCC4M7YANCD5 | Fully tested, 0 bad blocks, clean log — strong anchor; split from its CAUTION twin |
| | Z52BBV0P | IronWolf NAS drive (purpose-built RAID), essentially new, all tests clean; check SATA cable first |
| | WD-WCC4M1RYFSH9 | CAUTION: 16 bad blocks + 33 UNC SMART errors; 3 healthy drives carry it; split from its healthy twin WD-WCC4M7YANCD5 |
| /dev/md1 | W520VJFQ | Fully tested, 0 bad blocks, clean log — strong anchor; split from twin W1H31BLJ |
| | WFL3ZBBC | All SMART tests clean, nearly new (0.3 yrs); badblocks pending — complete before activating |
| | Z1E7BC0E | Cleanest of the Z1E batch; conveyance passed, extended test pending; no TLER (CC27 firmware) |
| | 5YD5VWL1 | CAUTION: confirmed UNC errors at ~818 GB; 3 healthy drives carry it; one CAUTION per data mirror keeps both mirrors equally reliable |
| /dev/md3 | ZFL0TF34 | Extended + conveyance tests clean; badblocks pending — complete before activating |
| | Z1E46C17 | High Load_Cycle_Count (185k), extended test still running; weakest non-CAUTION drive — suits the lower-stakes OS/backup mirror |
| | Z1E9K96R | CAUTION: 82°C thermal history, 1 runtime bad block; conveyance + badblocks still needed; md3 failure loses only OS/backup |
| | 5YD5PQE7 | CAUTION: 104,030 CRC errors (interface, not media); conveyance + badblocks still needed; replace SATA cable first |

**md0 and md1 are striped at the LVM level (poor-man's RAID10)** — if either mirror fails completely, all data is lost. Both mirrors therefore carry exactly one CAUTION drive, backed by three healthy members each, keeping their failure probability equal and low. The two worst CAUTION drives go to **md3** (OS + restic), where a total mirror failure is recoverable from backups. Before assembling, complete badblocks on WFL3ZBBC, Z1E7BC0E, ZFL0TF34, and Z1E46C17, and swap SATA cables on Z52BBV0P and 5YD5PQE7.

## Drive Status Reports

| Attribute | …BLJ | …SH9 | …CD5 | …FQ | …BBC | …V0P | …F34 | …QE7 | …WL1 | …C17 | …C0E | …96R |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **Mfr Year** | ~2019 | ~2019 | ~2019 | ~2019 | ~2016 | ~2020s | ~2016 | ~2024 | ~2019 | ~2024 | ~2022 | ~2024 |
| **Mfr Country** | 🇨🇳 | 🇹🇭 | 🇹🇭 | 🇨🇳 | 🇨🇳 | 🇨🇳 | 🇨🇳 | — | — | 🇨🇳 | 🇨🇳 | 🇨🇳 |
| **Hours** | 6.4y | 6.4y | 6.4y | 6.4y | 0.3y | 0.06y | 4.2y | 0.7y | 5.7y | 1.5y | 3.0y | 0.6y |
| **NAS Rated** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Bad Blocks** | 0 | ⚠️ 16 | 0 | 0 | — | — | — | — | — | — | — | — |
| **UNC Errors** | 0 | ⚠️ 33 | 0 | 0 | 0 | 0 | 0 | 0 | ⚠️ 15 | 0 | 0 | 0 |
| **UDMA CRC** | 0 | 0 | 0 | 0 | 0 | ⚠️ 42 | 0 | ⚠️ 104k | 0 | 0 | 0 | 0 |
| **Load Cycles** | 88 | 1,175 | 1,068 | 2,393 | — | — | ⚠️ 283k | — | ⚠️ 31k | ⚠️ 186k | 34k | 13k |
| **Start/Stop** | — | ⚠️ 77k | ⚠️ 77k | — | 1,695 | — | — | — | ⚠️ 31k | — | — | — |
| **High Fly Writes** | ⚠️ 256 | — | — | ⚠️ 37 | 0 | ⚠️ 9 | — | — | — | ⚠️ 1 | 0 | — |
| **Cmd Timeout** | — | — | — | — | — | ⚠️ 4 | — | ⚠️ 3 | — | ⚠️ 2 | 0 | — |
| **Runtime Bad Block** | — | — | — | — | — | — | — | ⚠️ 1 | — | — | — | ⚠️ 1 |
| **Max Temp** | 44°C | 44°C | 45°C | 43°C | 39°C | 35°C | 50°C | ⚠️ 55°C | 40°C | 45°C | 43°C | ⚠️ 82°C |
| **TLER** | — | — | — | — | ❌ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |

### W1H31BLJ — Seagate NAS HDD ST2000VN000-1H3164 (sdb)

**Serial number decode — `W1H31BLJ`:**
| Part | Meaning |
|---|---|
| `W` | Factory: Wuxi, China |
| `1` | Year code (Seagate fiscal year, exact mapping not public) |
| `H3` | Week code (alphanumeric — Seagate's fiscal week encoding) |
| `1BLJ` | Unit sequence number |

Seagate encodes manufacture date using a fiscal year/week scheme (fiscal year starts first Saturday of July). The exact character-to-date mapping is not publicly documented, but drive age (~6.4 years at time of testing) points to manufacture circa 2011.

✅ **Overall: OK — usable**

- ✅ SMART health: PASSED
- ✅ Bad blocks: 0 (clean pass across all 4 patterns)
- ✅ Power-on hours: 56,370 (~6.4 years)
- ✅ Current temperature: 28°C (lifetime max: 44°C, well within limits)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (most recent at 56,281 hours)

**Notable attributes:**
- ⚠️ `High_Fly_Writes` (ID 189) is at raw value 256 with a normalized value of 1 — this is a cosmetic flag on Seagate drives indicating the head flew too high during writes at some point, but with 0 reallocated sectors and a clean error log it has not caused data loss. Monitor but not a disqualifier.
- ✅ `Raw_Read_Error_Rate` raw value is large (36,214,552) but this is normal for Seagate; the normalized value (111) is healthy and above threshold (6).
- ✅ `Seek_Error_Rate` raw value is large (836,472,984) — same Seagate encoding; normalized value (89) is above threshold (30), no concern.

**Verdict:** Healthy drive with ~6.4 years of runtime. Suitable for NAS use. Worth running a periodic SMART extended test going forward.

### WD-WCC4M1RYFSH9 — WD Red WD20EFRX-68EUZN0 (sdc)

**Serial number decode — `WD-WCC4M1RYFSH9`:**
| Part | Meaning |
|---|---|
| `WD-` | Western Digital prefix (present on retail/OEM label; omitted in some system views) |
| `WCC` | Plant code: Korat, Thailand (WD's main HDD manufacturing site) |
| `4M` | Line/sub-facility identifier |
| `1RYFSH9` | Date + sequence encoding — WD's exact year/week character mapping is not publicly documented |

Drive age (~6.4 years at time of testing) points to manufacture circa 2011.

⚠️ **Overall: CAUTION — media errors present**

- ✅ SMART health: PASSED
- ❌ Bad blocks: **16** (all read-comparison errors during the 0xaa write/read pass)
- ✅ Power-on hours: 56,393 (~6.4 years)
- ✅ Current temperature: 27°C (lifetime max: 44°C)
- ⚠️ Reallocated sectors: 0 (drive has not yet remapped the affected sectors)
- ⚠️ Pending sectors: 0 (sector status may have cleared after badblocks rewrote them)
- ✅ Uncorrectable errors: 0 (SMART attribute)
- ❌ **SMART error log: 33 UNC (Uncorrectable) errors recorded** — log contains the most recent 24
- ⚠️ Start/Stop count: 76,660 — very high; indicates this drive was in a system that powered it on and off constantly (e.g. a desktop or spin-down-heavy NAS)

**Error log summary:**
All 33 logged errors are `UNC` (Uncorrectable Read) errors, occurring at disk power-on lifetime 56,316 hours, clustered around two LBA regions:
- ❌ Around LBA `0x0C4C_xxxx` – `0x0C9D_xxxx` (~206–211 million, ~105–108 GB into the disk)
- ❌ Around LBA `0x1170_0xxx` (~292 million, ~149 GB into the disk)

These are real, pre-existing unreadable sectors that existed before the badblocks run. Badblocks found 16 of them; SMART logged 33 UNC events (some may be retries of the same sectors). The drive's firmware did not remap them (Reallocated_Sector_Ct = 0), which means either the sectors recovered on rewrite (badblocks writes before reading) or the drive has exhausted its spare pool — though Reallocated_Sector_Ct being 0 makes the latter unlikely.

**Notable attributes:**
- ⚠️ `Start_Stop_Count` and `Power_Cycle_Count` both at 76,660 — far higher than the other drives (~88). This drive has been cycled tens of thousands of times more, suggesting a very different prior use case.
- ✅ `Load_Cycle_Count`: 1,175 — low relative to start/stop count, suggesting spin-down was not the main cause of cycling.

**Verdict:** This drive has confirmed media errors backed by SMART UNC history. It passed SMART overall health because no attributes crossed their thresholds, but it has demonstrated unreadable sectors. **Do not use as a sole copy of data.** In a RAID-Z or mirrored pool it can still contribute, but it should be treated as suspect and monitored closely. Consider replacing when possible.

### WD-WCC4M7YANCD5 — WD Red WD20EFRX-68EUZN0 (sdd)

**Serial number decode — `WD-WCC4M7YANCD5`:**
| Part | Meaning |
|---|---|
| `WD-` | Western Digital prefix |
| `WCC` | Plant code: Korat, Thailand |
| `4M` | Line/sub-facility identifier |
| `7YANCD5` | Date + sequence encoding — same WD scheme as the twin drive; exact character-to-date mapping not public |

Same age as WD-WCC4M1RYFSH9 (~6.4 years), both manufactured circa 2011 in the same facility.

✅ **Overall: OK — usable**

- ✅ SMART health: PASSED
- ✅ Bad blocks: 0 (clean pass across all 4 patterns)
- ✅ Power-on hours: 56,393 (~6.4 years)
- ✅ Current temperature: 27°C (lifetime max: 45°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (most recent at 56,306 hours)

**Notable attributes:**
- ⚠️ `Start_Stop_Count` and `Power_Cycle_Count`: 76,659 — same very high count as WD-WCC4M1RYFSH9, confirming these two drives came from the same system/use case.
- ✅ `Load_Cycle_Count`: 1,068 — low, consistent with the twin drive.
- ✅ `Raw_Read_Error_Rate`: 0 — cleaner raw read history than its twin.

**Verdict:** Despite the very high power cycle count (same history as the sdc drive), this drive shows no media errors and a clean SMART log. Suitable for NAS use. Monitor alongside WD-WCC4M1RYFSH9 since they share the same age and usage profile — if one is degrading, the other may follow.

### W520VJFQ — Seagate NAS HDD ST2000VN000-1HJ164 (sde)

**Serial number decode — `W520VJFQ`:**
| Part | Meaning |
|---|---|
| `W` | Factory: Wuxi, China |
| `5` | Year code (Seagate fiscal year encoding) |
| `20` | Week code (fiscal week — appears numeric here, suggesting week 20) |
| `VJFQ` | Unit sequence number |

Seagate's fiscal year starts the first Saturday of July. Week 20 of a fiscal year maps to roughly November/December of the prior calendar year or early in the new one. Drive age (~6.4 years at time of testing) points to manufacture circa 2011.

✅ **Overall: OK — usable**

- ✅ SMART health: PASSED
- ✅ Bad blocks: 0 (clean pass across all 4 patterns)
- ✅ Power-on hours: 56,368 (~6.4 years)
- ✅ Current temperature: 28°C (lifetime max: 43°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (most recent at 56,279 hours)

**Notable attributes:**
- ⚠️ `High_Fly_Writes` (ID 189): raw value 37, normalized 63 — elevated but not critical. Some head excursions have occurred; worth monitoring.
- ✅ `Raw_Read_Error_Rate` raw value large (49,389,592) — normal Seagate encoding; normalized 112 is healthy.
- ⚠️ `Load_Cycle_Count`: 2,393 — higher than the twin Seagate (W1H31BLJ at 88), suggesting more head park/unpark cycles in its history.

**Verdict:** Clean drive with no errors across all tests. The slightly elevated High_Fly_Writes and Load_Cycle_Count compared to the other Seagate are worth noting but not alarming. Suitable for NAS use.

### WFL3ZBBC — Seagate Barracuda ST2000DM008-2FR102

**Serial number decode — `WFL3ZBBC`:**
| Part | Meaning |
|---|---|
| `W` | Factory: Wuxi, China |
| `F` | Year code (Seagate uses letters to extend past digit 9 in their fiscal year scheme) |
| `L3` | Week code (alphanumeric fiscal week — L is the 12th letter, indicating a later-in-year week) |
| `ZBBC` | Unit sequence number |

Drive has only ~2,450 power-on hours at time of testing (~0.3 years of active use), so it was relatively lightly used before arriving here regardless of manufacture date.

⏳ **Overall: PENDING — extended self-test in progress**

- ✅ SMART health: PASSED
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 2,442 (~0.3 years — nearly new)
- ✅ Current temperature: 27°C (lifetime max: 39°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (at 2,444 hours)
- ✅ Conveyance self-test: Completed without error (at 2,451 hours)

**Notable attributes:**
- ✅ `Start_Stop_Count`: 1,695 / `Power_Cycle_Count`: 1,450 — low and consistent, normal use history
- ✅ `Seek_Error_Rate` raw value 16,377,844 — normal Seagate encoding; normalized 72 is above threshold (45)
- ✅ `Raw_Read_Error_Rate` raw value 31,474,907 — normal Seagate encoding; normalized 75 is healthy (threshold 6)
- ✅ `High_Fly_Writes`: 0 — clean

**Note:** This is a desktop Barracuda (7200 rpm), not a NAS-rated drive like the others. It lacks vibration compensation (TLER, rotational vibration sensors) present on NAS/Red drives. Fine for single-drive or low-vibration use; worth considering if used alongside other spinning drives.

**Verdict:** All SMART tests clean. Badblocks still to run — do that before putting the drive into service.

### Z52BBV0P — Seagate IronWolf ST2000VN004-2E4164

**Serial number decode — `Z52BBV0P`:**
| Part | Meaning |
|---|---|
| `Z` | Factory: Zhongshan, China (newer Seagate facility) |
| `5` | Year code |
| `2B` | Week code (alphanumeric fiscal week) |
| `BV0P` | Unit sequence number |

Drive has only ~487 power-on hours at time of testing, consistent with a recent manufacture date (2020s era given the Zhongshan factory designation and IronWolf product line).

⏳ **Overall: PENDING — badblocks still to run**

- ✅ SMART health: PASSED
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 487 (~0.06 years — essentially new)
- ✅ Current temperature: 20°C (lifetime max: 35°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (at 484 hours)
- ✅ Conveyance self-test: Completed without error (at 487 hours)

**Notable attributes:**
- ⚠️ `UDMA_CRC_Error_Count`: 42 — non-zero, indicating interface errors at some point (bad cable, loose connector, or hot-plug event). Not a drive fault, but worth checking the SATA cable/port before putting into service.
- ⚠️ `High_Fly_Writes`: 9 — minor, normalized value 91 is well above threshold (0); monitor but not a concern.
- ⚠️ `Command_Timeout`: 4 — a few commands timed out historically, likely related to the same interface events as the CRC errors.
- ✅ `Raw_Read_Error_Rate` raw value 55,436,976 — normal Seagate encoding; normalized 113 is healthy.
- ✅ `Seek_Error_Rate` raw value 467,072 — low and clean; normalized 100.
- ✅ SCT Error Recovery Control: Read/Write timeout set to 7.0s — correctly configured for RAID use.

**Note:** This is a Seagate IronWolf NAS drive (5900 rpm), purpose-built for NAS/RAID use. Has TLER (via SCT ERC) and is rated for 24/7 multi-drive operation.

**Verdict:** Conveyance test passed. Check/replace the SATA cable before use given the CRC error count. All other indicators clean. Run badblocks before putting into service.

### ZFL0TF34 — Seagate Barracuda ST2000DM008-2FR102

**Serial number decode — `ZFL0TF34`:**
| Part | Meaning |
|---|---|
| `Z` | Factory: Zhongshan, China |
| `F` | Year code (same letter as WFL3ZBBC — consistent with the same ~2016 era) |
| `L0` | Week code (alphanumeric fiscal week) |
| `TF34` | Unit sequence number |

Drive has ~36,600 power-on hours (~4.2 years of active use at time of testing). The `F` year code matches WFL3ZBBC, suggesting both Barracudas were manufactured in the same fiscal year.

⏳ **Overall: PENDING — badblocks to run**

- ✅ SMART health: PASSED
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 36,592 (~4.2 years)
- ✅ Current temperature: 26°C (lifetime max: 50°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (most recent at 36,595 hours)
- ✅ Conveyance self-test: Completed without error (at 36,592 hours)

**Notable attributes:**
- ⚠️ `Load_Cycle_Count`: 283,127 with normalized value of **1** (threshold 0) — extremely high head park count, indicating aggressive power management or APM head parking in a prior system. The drive has been unloading heads constantly throughout its life.
- ⚠️ `Power-Off_Retract_Count`: 1,370 — consistent with a system that powered off frequently rather than spinning down via APM.
- ⚠️ `Number of Mechanical Start Failures`: 3 (from Device Statistics) — a small number of spin-up failures in its history; not alarming at this count but worth noting.
- ✅ `Seek_Error_Rate` raw value 298,809,582 — normal Seagate encoding; normalized 85 is above threshold (45).
- ✅ `Raw_Read_Error_Rate` raw value 9,812,397 — normal Seagate encoding; normalized 70 is above threshold (6).
- ✅ `UDMA_CRC_Error_Count`: 0 — clean interface history.

**Note:** Desktop Barracuda (7200 rpm), not NAS-rated. No SCT ERC / TLER support. The very high `Load_Cycle_Count` suggests it lived in a desktop with aggressive APM head parking — common in systems using Linux default APM settings. The heads themselves show 36,437 flying hours consistent with the power-on time, so actual spinning time is normal.

**Verdict:** Clean SMART history despite heavy head park usage. Extended and conveyance tests both passed. Run badblocks before putting into service.

### 5YD5PQE7 — Seagate Barracuda Green ST2000DL003-9VT166

**Serial number decode — `5YD5PQE7`:**
| Part | Meaning |
|---|---|
| `5` | Year code (Seagate fiscal year encoding) |
| `YD` | Week code (alphanumeric fiscal week) |
| `5PQE7` | Unit sequence number |

Drive has ~6,393 power-on hours (~0.7 years of active use). The Barracuda Green line was discontinued; this is an older low-power desktop drive.

⚠️ **Overall: CAUTION — serious interface error history**

- ✅ SMART health: PASSED
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 6,400 (~0.7 years)
- ✅ Current temperature: 34°C (lifetime max: 55°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (at 6,398 hours)
- ⏳ Conveyance self-test: not yet run

**Notable attributes:**
- ❌ `UDMA_CRC_Error_Count`: 104,030 — extremely high. This is an order of magnitude beyond what is seen on other drives here. Indicates severe and sustained interface problems in its prior life — likely a bad cable, enclosure, or controller. Not a drive media fault, but this history demands a cable/port swap and re-check before trusting the drive.
- ⚠️ `Runtime_Bad_Block`: 1 — one runtime bad block has been logged by the drive firmware. Monitor; may be benign, but combined with the CRC history warrants caution.
- ⚠️ `Command_Timeout`: 3 — consistent with the CRC error history (interface stalls leading to command timeouts).
- ⚠️ `Airflow_Temperature_Cel` worst: 044, threshold: 045 — the worst recorded airflow temperature came within 1 point of the SMART threshold. The drive has operated near its thermal limit at some point in its history.
- ✅ `Raw_Read_Error_Rate` raw value 15,588,384 — normal Seagate encoding; normalized 108 is healthy.
- ✅ `Seek_Error_Rate` raw value 4,322,273,481 — large raw value typical of Seagate; normalized 074 is above threshold (030).

**Note:** Seagate Barracuda Green (5900 rpm), not NAS-rated. No TLER/SCT ERC support. APM is unavailable on this model. Sector sizes are 512 bytes logical / 4096 bytes physical (Advanced Format).

**Verdict:** The 104,030 CRC error count is a serious red flag for the interface environment this drive lived in. Replace the SATA cable and port, then re-run smartctl to confirm the count is not growing. Extended self-test completed clean. Run conveyance then badblocks before considering for use.

### 5YD5VWL1 — Seagate Barracuda Green ST2000DL003-9VT166

**Serial number decode — `5YD5VWL1`:**
| Part | Meaning |
|---|---|
| `5` | Year code (Seagate fiscal year encoding) |
| `YD` | Week code (alphanumeric fiscal week) |
| `5VWL1` | Unit sequence number |

Same model family as 5YD5PQE7. Drive has ~50,144 power-on hours (~5.7 years of active use). The much higher power-on hours relative to its twin suggests a very different use history.

⚠️ **Overall: CAUTION — confirmed media errors**

- ✅ SMART health: PASSED (marginal attributes flagged)
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 50,151 (~5.7 years)
- ✅ Current temperature: 32°C (lifetime max: 40°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ⚠️ Uncorrectable errors: 2 (`Reported_Uncorrect` = 2)
- ❌ **SMART error log: 15 UNC (Uncorrectable) errors recorded**
- ✅ Extended self-test: Completed without error (at 50,149 hours)
- ⏳ Conveyance self-test: not yet run

**Error log summary:**
All 15 logged errors are `UNC` (Uncorrectable Read) errors, occurring at disk power-on lifetime 434 hours (very early in the drive's life), clustered tightly around a single LBA region:
- ❌ LBA `0x5f42689c`–`0x5f42689f` (~1.598 billion, ~818 GB into the disk) — 4 distinct physical sectors repeatedly failing across multiple retry attempts.

The errors occurred during a surface scan (`READ VERIFY SECTOR(S) EXT` commands), and the pattern of write-then-verify suggests a badblocks-style tool was running. The drive firmware never reallocated these sectors (Reallocated_Sector_Ct = 0), meaning the bad area persists in the sector map.

**Notable attributes:**
- ⚠️ `Start_Stop_Count`: 31,268 / `Load_Cycle_Count`: 31,269 — very high and nearly equal, indicating this drive was parked and unparked (or power-cycled) an extraordinary number of times, likely from aggressive APM head parking in a prior desktop system.
- ⚠️ `Seek_Error_Rate` normalized: 069, worst: 060 (threshold 030) — degraded but above threshold. The large raw value (60,269,278,047) is unusually high even by Seagate encoding standards; worth watching.
- ✅ `UDMA_CRC_Error_Count`: 0 — clean interface history.
- ✅ `Raw_Read_Error_Rate` normalized 114, healthy.

**Note:** Seagate Barracuda Green (5900 rpm), not NAS-rated. No TLER/SCT ERC support. Advanced Format (512 logical / 4096 physical). Same caveats as 5YD5PQE7 regarding desktop-only suitability.

**Verdict:** This drive has confirmed unreadable sectors at ~818 GB into the disk, logged at 434 hours of age. The drive has never remapped them. **Do not use as a sole copy of data.** In a redundant array it can contribute, but it should be treated as a degraded member. Consider replacing when possible. Extended self-test completed clean — the problem appears localized to the original LBA region. Run conveyance then badblocks before putting into service.

### Z1E46C17 — Seagate Barracuda 7200.14 ST2000DM001-1CH164

**Serial number decode — `Z1E46C17`:**
| Part | Meaning |
|---|---|
| `Z` | Factory: Zhongshan, China |
| `1` | Year code (Seagate fiscal year encoding) |
| `E4` | Week code (alphanumeric fiscal week) |
| `6C17` | Unit sequence number |

Drive has ~13,191 power-on hours (~1.5 years of active use). Same model family as Z1E9K96R; both carry the `Z1E` prefix suggesting manufacture in the same fiscal year and week range.

⏳ **Overall: PENDING — extended self-test in progress, badblocks not yet run**

- ✅ SMART health: PASSED
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 13,191 (~1.5 years)
- ✅ Current temperature: 32°C (lifetime max: 45°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ⏳ Extended self-test: In progress at time of capture (90% remaining at 13,190 hours)
- ➖ Conveyance self-test: Not supported on this firmware (HP33)

**Notable attributes:**
- ⚠️ `Load_Cycle_Count`: 185,601 with normalized value of **8** (threshold 0) — extremely high head park count, consistent with aggressive APM head parking in a prior desktop system. Same pattern as ZFL0TF34.
- ⚠️ `Seek_Error_Rate` worst: 060 (threshold 030) — degraded worst-case but current normalized value (078) is above threshold; monitor.
- ⚠️ `Command_Timeout`: 2 events — minor; not alarming at this count.
- ⚠️ `High_Fly_Writes`: 1 — cosmetic; no reallocated sectors, no concern.
- ✅ `Raw_Read_Error_Rate` normalized 111, healthy (threshold 6).
- ✅ `UDMA_CRC_Error_Count`: 0 — clean interface history.
- ✅ `SCT Error Recovery Control`: Read/Write supported — can configure TLER for RAID use.

**Note:** Desktop Barracuda (7200 rpm), not NAS-rated. No TLER configured by default, but SCT ERC is supported (unlike some other drives here). The very high Load_Cycle_Count is the main concern — same pattern as ZFL0TF34.

**Verdict:** Clean SMART history so far. Extended self-test was still running at capture time — wait for completion before drawing conclusions. Run conveyance (if firmware supports it) then badblocks before putting into service.

### Z1E7BC0E — Seagate Barracuda 7200.14 ST2000DM001-1CH164

**Serial number decode — `Z1E7BC0E`:**
| Part | Meaning |
|---|---|
| `Z` | Factory: Zhongshan, China |
| `1` | Year code (Seagate fiscal year encoding) |
| `E7` | Week code (alphanumeric fiscal week — slightly later than Z1E46C17) |
| `BC0E` | Unit sequence number |

Drive has ~26,433 power-on hours (~3.0 years of active use). Shares the `Z1E` year prefix with Z1E46C17 and Z1E9K96R, placing manufacture in the same fiscal year. The later week code (`E7` vs `E4`) suggests it was built a few weeks after Z1E46C17.

⏳ **Overall: PENDING — extended self-test in progress, badblocks not yet run**

- ✅ SMART health: PASSED
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 26,433 (~3.0 years)
- ✅ Current temperature: 32°C (lifetime max: 43°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ⏳ Extended self-test: In progress at time of capture (80% remaining at 26,433 hours)
- ✅ Conveyance self-test: Completed without error (at 26,432 hours)

**Notable attributes:**
- ✅ `Load_Cycle_Count`: 34,195 (normalized 083) — moderate; not alarming.
- ✅ `Seek_Error_Rate` normalized 090, worst 060 (threshold 030) — healthy current value.
- ✅ `High_Fly_Writes`: 0 — clean.
- ✅ `Command_Timeout`: 0 — no timeout events.
- ✅ `UDMA_CRC_Error_Count`: 0 — clean interface history.
- ✅ `Raw_Read_Error_Rate` normalized 119 (threshold 6) — healthy.
- ✅ `Head_Flying_Hours`: 26,553 — consistent with power-on hours; normal flying time.
- ➖ `SCT Error Recovery Control`: Not supported on this firmware (CC27) — cannot configure TLER for RAID use.

**Note:** Desktop Barracuda (7200 rpm), not NAS-rated. No SCT ERC / TLER support (same as Z1E9K96R with firmware CC27). `Wt Cache Reorder` also unavailable. The CC27 firmware variant appears to lack these features entirely.

**Verdict:** All available tests clean. Conveyance passed. Extended self-test was still running at capture — wait for completion. Run badblocks before putting into service. Cleanest of the three Z1E-series drives so far.

### Z1E9K96R — Seagate Barracuda 7200.14 ST2000DM001-1CH164

**Serial number decode — `Z1E9K96R`:**
| Part | Meaning |
|---|---|
| `Z` | Factory: Zhongshan, China |
| `1` | Year code (Seagate fiscal year encoding) |
| `E9` | Week code (alphanumeric fiscal week) |
| `K96R` | Unit sequence number |

Drive has ~5,157 power-on hours (~0.6 years of active use). The Barracuda 7200.14 is a desktop-class drive.

⚠️ **Overall: CAUTION — severe overheating history**

- ✅ SMART health: PASSED (marginal attributes flagged)
- ⏳ Bad blocks: not yet run
- ✅ Power-on hours: 5,163 (~0.6 years)
- ✅ Current temperature: 33°C (lifetime max: 82°C)
- ✅ Reallocated sectors: 0
- ✅ Pending sectors: 0
- ✅ Uncorrectable errors: 0
- ✅ SMART error log: No errors logged
- ✅ Extended self-test: Completed without error (at 5,160 hours)
- ⏳ Conveyance self-test: not yet run

**Notable attributes:**
- ❌ `Airflow_Temperature_Cel` worst value: 018, threshold: 045 — the worst normalized value (018) is **below** the failure threshold (045), meaning this attribute has historically been in a PAST FAIL state. The lifetime max temperature of **82°C** confirms the drive experienced extreme overheating at some point. Seagate's max recommended operating temp for this model is 60°C; 82°C is 22°C beyond that limit. Thermal damage to heads and platters is a real risk at these temperatures.
- ⚠️ `Runtime_Bad_Block`: 1 — one runtime bad block logged; may be a consequence of the thermal event.
- ✅ `Load_Cycle_Count`: 12,905 — moderate; consistent with some head parking but not extreme.
- ✅ `UDMA_CRC_Error_Count`: 0 — clean interface history.
- ✅ `Raw_Read_Error_Rate` raw value 227,729,008 — normal Seagate encoding; normalized 119 is healthy.
- ✅ `Seek_Error_Rate` raw value 10,715,533 — low; normalized 070 is above threshold (030).

**Note:** Desktop Barracuda (7200 rpm), not NAS-rated. No SCT ERC / TLER support. The `Wt Cache Reorder` feature is listed as unavailable on this model.

**Verdict:** The 82°C lifetime maximum temperature is a serious concern — this drive was cooked at some point. Current media metrics are clean (no reallocated sectors, no error log), and the extended self-test completed without error. Thermal damage can still be latent. Run conveyance then badblocks before making any decision about using this drive. It should be treated as high-risk until those tests pass clean. Do not use in a critical RAID position without monitoring temperature closely.

## Drive Information

Research into each model code found in this pool — target market, key specifications, and real-world customer experience.

---

### Seagate NAS HDD — ST2000VN000

**Target market:** Small-to-medium NAS systems, 1–8 bay enclosures, up to 20 users. Positioned as Seagate's dedicated NAS line before IronWolf replaced it.

| Spec | Value |
|---|---|
| RPM | 5900 |
| Cache | 64 MB |
| Interface | SATA 6 Gb/s |
| Max sustained transfer | 180 MB/s |
| MTBF | 1,000,000 hours |
| Workload rating | 180 TB/year |
| Power-on hours rated | 8,760 hr/yr (24/7) |
| Warranty | 3 years |
| TLER / ERC | Yes (NASWorks Error Recovery Control) |
| Advanced Format | No (512n) |

**Real-world experience:** Generally well regarded as a purpose-built NAS drive from its era. The NASWorks feature set (TLER, vibration compensation, power management tuning) made it genuinely better suited to multi-drive enclosures than contemporary desktop drives. No widespread catastrophic failure patterns reported. Common failure mode in data recovery cases is burnt PCB, consistent with age rather than a design defect. Largely superseded by IronWolf but remains a solid performer for the use case it was designed for.

---

### WD Red — WD20EFRX

**Target market:** Home and SOHO NAS, 1–8 bay systems. WD's first drive line marketed specifically at NAS users, launched 2012.

| Spec | Value |
|---|---|
| RPM | 5400 (marketed as "IntelliPower") |
| Cache | 64 MB |
| Interface | SATA 6 Gb/s |
| Max sustained transfer | ~112 MB/s |
| MTBF | 1,000,000 hours |
| Load/unload cycles rated | 600,000 |
| Non-recoverable read errors | <1 per 10^14 bits |
| Warranty | 3 years |
| Recording technology | **CMR** (conventional) |
| TLER | Yes |

**Real-world experience:** The WD20EFRX specifically uses **CMR** — this is important context given WD's later scandal. Starting around 2019–2020, WD quietly switched some WD Red SKUs (particularly 2–6 TB) to SMR (shingled magnetic recording) without disclosure. SMR drives caused severe RAID rebuild failures, array drops under ZFS/TrueNAS, and data loss. WD faced a class-action lawsuit settled for $2.7 million. The **WD20EFRX is the older CMR variant** and does not have the SMR write-performance degradation problem. However, Backblaze reported an 8.2% annualised failure rate for this model in their 2016 data — notably high, though this reflects data-centre duty cycles and may not map directly to home NAS use. The very high Start/Stop count on the units in this pool (76,000+) is consistent with this model being used in duty cycles it was not optimally designed for.

---

### Seagate Barracuda — ST2000DM008

**Target market:** Desktop PC, home server, entry-level direct-attached storage (DAS). Not NAS-rated. Seagate's mainstream desktop drive line, introduced 2017.

| Spec | Value |
|---|---|
| RPM | 7200 |
| Cache | 256 MB |
| Interface | SATA 6 Gb/s |
| Max sustained transfer | ~190 MB/s |
| MTBF | Not published (desktop class) |
| Workload rating | 55 TB/year |
| Warranty | 2 years |
| TLER / ERC | No |
| Advanced Format | 512e |

**Real-world experience:** Broadly regarded as a capable, fast desktop drive. The 7200 RPM spindle and 256 MB cache give it noticeably better sequential performance than the NAS-oriented 5900 RPM drives in this pool. The lack of TLER is the key liability in a RAID context — under a long read error, the drive will retry for up to 2 minutes before reporting the error, which causes most RAID controllers to drop it from the array. The 55 TB/year workload limit is also well below the 24/7 NAS-rated drives. Fine for low-duty-cycle or desktop use; workable in RAID if TLER is not strictly required by the controller. No major widespread failure pattern specific to this firmware revision.

---

### Seagate IronWolf — ST2000VN004

**Target market:** NAS enclosures, 1–8 bay, home through small business. The IronWolf line replaced the original Seagate NAS HDD line and is Seagate's current mainstream NAS product.

| Spec | Value |
|---|---|
| RPM | 5900 |
| Cache | 64 MB |
| Interface | SATA 6 Gb/s |
| Max sustained transfer | 180 MB/s |
| MTBF | 1,000,000 hours |
| Workload rating | 180 TB/year |
| Power-on hours rated | 8,760 hr/yr (24/7) |
| Load/unload cycles rated | 600,000 |
| Non-recoverable read errors | <1 per 10^14 bits |
| Warranty | 3 years |
| TLER / ERC | Yes (SCT ERC, confirmed on Z52BBV0P) |
| Recording technology | CMR |

**Real-world experience:** Consistently well reviewed as a reliable NAS drive. Designed specifically for RAID environments with rotational vibration (RV) sensors and TLER configured out of the box. User reviews across large sample sets (4.6 stars from ~6,000 Amazon reviews) indicate high satisfaction with reliability in sustained NAS workloads. Seagate includes three years of complimentary Rescue Data Recovery Services with IronWolf drives (95% success rate claimed). No widespread failure patterns documented. Considered best-in-class for this use case in the sub-4 TB range.

---

### Seagate Barracuda Green — ST2000DL003

**Target market:** Desktop PC, consumer storage, power-conscious desktop builds. Discontinued — the Green line was folded into the standard Barracuda lineup. Introduced around 2010.

| Spec | Value |
|---|---|
| RPM | 5900 |
| Cache | 64 MB |
| Interface | SATA 6 Gb/s |
| Max sustained transfer | ~144 MB/s |
| AFR | 0.34% (Seagate rated) |
| Average seek (read) | 12 ms |
| Average latency | 4.16 ms |
| Warranty | 1 year (OEM) / 2 years (retail) |
| TLER / ERC | No |
| Advanced Format | Yes (512 logical / 4096 physical) |

**Real-world experience:** Mixed-to-poor reputation. The "Green" branding indicated aggressive power-saving behaviour, which caused two significant problems in practice: (1) the drives would spin down and park heads very frequently under desktop power management defaults (especially on Linux), causing extremely high Load_Cycle_Count values and premature head wear — exactly what is seen on the units in this pool; and (2) in RAID enclosures, the spin-down behaviour caused RAID controllers to interpret the drive as having disconnected, kicking it out of the array. Multiple user reports describe drives dropping from arrays under load. An additional concern: data recovery shops report that the burnt PCB failure mode is disproportionately common on these drives. Seagate discontinued the Green line entirely. These drives are not suited to NAS or RAID use.

---

### Seagate Barracuda 7200.14 — ST2000DM001

**Target market:** Desktop PC, home storage, budget workstation. The "7200.14" designation refers to the 14th generation of the Barracuda line. Introduced 2011.

| Spec | Value |
|---|---|
| RPM | 7200 |
| Cache | 64 MB |
| Interface | SATA 6 Gb/s |
| Max sustained transfer | ~156 MB/s |
| MTBF | Not published (desktop class) |
| Workload rating | 55 TB/year |
| Average seek (read) | <8.5 ms |
| Warranty | 2 years |
| TLER / ERC | Firmware-dependent — HP33 supports SCT ERC; CC27 does not |
| Advanced Format | Yes (512e) |

**Real-world experience:** One of the most widely deployed desktop drives of its era, and consequently one of the most studied. Backblaze ran large populations of ST2000DM001 drives in their storage pods and documented notably elevated failure rates. The 3 TB variant (ST3000DM001) was particularly notorious; the 2 TB model fared better but still showed meaningful failure rates at the 3–4 year mark in high-duty environments. The model has two major real-world quirks: (1) the "Rosewood" firmware revision (CC27) found on some units disables SCT ERC/TLER entirely, making those drives unsuitable for RAID without modification; (2) the default APM settings in many desktop systems caused extremely high Load_Cycle_Count accumulation over time, accelerating head wear — both Z1E46C17 and ZFL0TF34 in this pool show this pattern. Performance is strong for desktop use. Not recommended as a primary NAS drive without TLER verification.

---

## Linux Commands & Workarounds

Known issues with the drives in this pool each have a corresponding Linux remedy. The table below maps the problem to the command, followed by persistence instructions.

### Issue 1 — Excessive head parking / high Load_Cycle_Count (ST2000DL003, ST2000DM001, ST2000DM008, WD20EFRX)

**Why it happens:** Linux's default APM (Advanced Power Management) setting aggressively parks drive heads after a few seconds of inactivity. This was the primary killer of Barracuda Green drives and has left ZFL0TF34 (283k LCC) and Z1E46C17 (186k LCC) in this pool in a degraded state.

**Check the current APM level:**
```bash
sudo hdparm -B /dev/sdX
```
Value 1–127 = power-saving with spin-down permitted. Value 128–254 = performance mode, no spin-down. Value 255 = APM fully disabled.

**Fix — disable aggressive parking:**
```bash
sudo hdparm -B 254 /dev/sdX    # max performance, no spin-down; use on all non-NAS drives
sudo hdparm -B 255 /dev/sdX    # disable APM entirely (not all drives support this)
```

**Fix for WD drives with the IntelliPark idle3 timer** (WD20EFRX — the 8-second internal firmware timer is separate from OS APM):
```bash
# Install idle3-tools (Debian/Ubuntu)
sudo apt install idle3-tools

# Check current idle3 value (0 = disabled)
sudo idle3ctl -g /dev/sdX

# Disable the idle3 timer completely
sudo idle3ctl -d /dev/sdX

# Or set a longer timer (value 129–255 = multiples of 30s; e.g. 130 ≈ 60s)
sudo idle3ctl -s 130 /dev/sdX
```
> **Important:** idle3 changes require a full power-off (not just reboot) to take effect — the drive must lose power for the new firmware setting to activate.

**Make APM setting persist across reboots** — create a udev rule:
```bash
# /etc/udev/rules.d/69-hdparm.rules
# Replace MODEL with the drive's model string from: udevadm info /dev/sdX | grep ID_MODEL
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000DL003*", RUN+="/usr/bin/hdparm -B 254 %N"
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000DM001*", RUN+="/usr/bin/hdparm -B 254 %N"
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000DM008*", RUN+="/usr/bin/hdparm -B 254 %N"
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="WDC_WD20EFRX*", RUN+="/usr/bin/hdparm -B 254 %N"
```

Alternatively, for `/etc/hdparm.conf` (applied by the hdparm init script on Debian-based systems):
```
/dev/disk/by-id/ata-ST2000DL003-... {
    apm = 254
}
```

---

### Issue 2 — No TLER / drives dropped from RAID on long read errors (ST2000DL003, ST2000DM008, ST2000DM001 CC27 firmware)

**Why it happens:** Without Time-Limited Error Recovery (TLER / SCT ERC), a drive retrying a bad sector can stall for up to 2 minutes. Most RAID controllers (including Linux md) treat this as a timeout and kick the drive from the array, causing a degraded rebuild even if the drive itself is fine.

**Check if SCT ERC is supported and what it is set to:**
```bash
sudo smartctl -l scterc /dev/sdX
```
Output will show Read and Write timeout values in deciseconds, or `Disabled` / `Not supported`.

**Enable TLER / set ERC timeout (7 seconds = 70 deciseconds — standard RAID value):**
```bash
sudo smartctl -l scterc,70,70 /dev/sdX
```
This sets both read and write error recovery timeout to 7.0 seconds.

> **Note:** The ST2000DL003 (Barracuda Green) and ST2000DM001 with CC27 firmware **do not support SCT ERC** — the command will fail. There is no software workaround for these; the only mitigation is ensuring the md RAID timeout (below) is generous enough to absorb the drive's natural retry time.

**Try the persistent flag** (supported on some drives, survives power cycle):
```bash
sudo smartctl -l scterc,70,70,p /dev/sdX
```

**Set the Linux md RAID device timeout to be more tolerant** (reduces the chance of md dropping a slow drive):
```bash
# Check current timeout (in seconds) — typically 30s default
cat /sys/block/sdX/device/timeout

# Increase to 180s to give non-TLER drives more time before md gives up
echo 180 | sudo tee /sys/block/sdX/device/timeout
```

**Make the md timeout persist** — add to a udev rule:
```bash
# /etc/udev/rules.d/69-raid-timeout.rules
ACTION=="add", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000DL003*", ATTR{../timeout}="180"
ACTION=="add", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000DM001*", ATTR{../timeout}="180"
```

**Make SCT ERC persist across reboots** — using a udev rule or the mdraid-safe-timeouts project:
```bash
# Manual udev approach — triggers when a drive appears
# /etc/udev/rules.d/69-scterc.rules
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000VN000*", RUN+="/usr/sbin/smartctl -l scterc,70,70 %N"
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000VN004*", RUN+="/usr/sbin/smartctl -l scterc,70,70 %N"
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="ST2000DM001*", RUN+="/usr/sbin/smartctl -l scterc,70,70 %N"
ACTION=="add|change", SUBSYSTEM=="block", ENV{ID_MODEL}=="WDC_WD20EFRX*", RUN+="/usr/sbin/smartctl -l scterc,70,70 %N"
```

The [mdraid-safe-timeouts](https://github.com/jonathanunderwood/mdraid-safe-timeouts) project provides a more robust version of this that triggers on md array assembly rather than individual drive appearance.

---

### Issue 3 — UDMA CRC errors / interface errors (Z52BBV0P, 5YD5PQE7)

**Why it happens:** CRC errors are an interface-layer problem — bad cable, loose connector, or marginal port — not a drive media fault. The drive itself is fine but the data path is unreliable.

**Check the current CRC error count:**
```bash
sudo smartctl -A /dev/sdX | grep UDMA_CRC
```

**There is no software fix** — CRC errors require a physical intervention:
1. Replace the SATA data cable (cheap cables are a common culprit)
2. Try a different SATA port on the motherboard or controller card
3. Re-seat both ends of the cable firmly
4. After swapping, monitor whether the count grows:

```bash
# Run twice, a day apart, and compare the raw value
sudo smartctl -A /dev/sdX | grep UDMA_CRC
```

If the count stops growing after the cable swap, the drive is fine. If it keeps climbing, suspect the controller or enclosure backplane.

---

### Issue 4 — Verify SCT ERC is configured after assembling the array

It is easy to forget to set ERC after an array is assembled or a drive is replaced. A one-liner to check all data drives at once:

```bash
for dev in /dev/sd{a..l}; do
    echo -n "$dev: "
    sudo smartctl -l scterc "$dev" 2>/dev/null | grep -E "Read|Write|supported" || echo "not supported / error"
done
```

And to set 7s ERC on all drives that support it in one pass:
```bash
for dev in /dev/sd{a..l}; do
    sudo smartctl -l scterc,70,70 "$dev" 2>/dev/null
done
```

---

### Quick reference — which fix applies to which drive

| Serial | Model | APM fix | idle3 fix | SCT ERC settable | SCT ERC native |
|---|---|---|---|---|---|
| W1H31BLJ | ST2000VN000 | Not needed (NAS drive) | No | Yes | Yes |
| WD-WCC4M1RYFSH9 | WD20EFRX | `hdparm -B 254` | `idle3ctl -d` | Yes | Yes |
| WD-WCC4M7YANCD5 | WD20EFRX | `hdparm -B 254` | `idle3ctl -d` | Yes | Yes |
| W520VJFQ | ST2000VN000 | Not needed (NAS drive) | No | Yes | Yes |
| WFL3ZBBC | ST2000DM008 | `hdparm -B 254` | No | Yes | No |
| Z52BBV0P | ST2000VN004 | Not needed (NAS drive) | No | Yes | Yes — already set |
| ZFL0TF34 | ST2000DM008 | `hdparm -B 254` | No | Yes | No |
| 5YD5PQE7 | ST2000DL003 | `hdparm -B 254` | No | **Not supported** | No — increase md timeout instead |
| 5YD5VWL1 | ST2000DL003 | `hdparm -B 254` | No | **Not supported** | No — increase md timeout instead |
| Z1E46C17 | ST2000DM001 (HP33) | `hdparm -B 254` | No | Yes (HP33 supports it) | No |
| Z1E7BC0E | ST2000DM001 (CC27) | `hdparm -B 254` | No | **Not supported (CC27)** | No — increase md timeout instead |
| Z1E9K96R | ST2000DM001 (CC27) | `hdparm -B 254` | No | **Not supported (CC27)** | No — increase md timeout instead |

