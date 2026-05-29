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

| Mirror | Count | Drives | Purpose |
|---|---|---|---|
| /dev/md0 | 3-4 | W1H31BLJ, WD-WCC4M7YANCD5, <TBD>, <TBD> | LVM PV1 (Data) |
| /dev/md1 | 3-4 | W520VJFQ, WD-WCC4M1RYFSH9, <TBD>, <TBD> | LVM PV2 (Data) |
| /dev/md3 | 4   | <TBD>, <TBD>, <TBD>, <TBD> | OS and restic snapshots |

| PVs | VGs | LVs | Size |
|---|---|---|---|
| /dev/md0 | raid10 | data | 4T |
| /dev/md1 | | | |
| /dev/md3 | system | root | 80G |
| | | swap | 20G |
| | | bkp | 1.8T |
| | | boot | 1G |

> Note: I didn't want to do real RAID10 cos you can't grow or shrink the mirrors

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

| Size | Model | Serial | SH | SE | BB | SC | Hours | Status |
|---|---|---|---|---|---|---|---|---|
| 2TB | Seagate NAS ST2000VN000-1H3164 | W1H31BLJ | :white_check_mark: | :white_check_mark: | 0 | — | ~6.4 yrs | OK |
| 2TB | :warning: WD Red WD20EFRX-68EUZN0 | WD-WCC4M1RYFSH9 | :white_check_mark: | :white_check_mark: | 16 | — | ~6.4 yrs | CAUTION |
| 2TB | WD Red WD20EFRX-68EUZN0 | WD-WCC4M7YANCD5 | :white_check_mark: | :white_check_mark: | 0 | — | ~6.4 yrs | OK |
| 2TB | Seagate NAS ST2000VN000-1HJ164 | W520VJFQ | :white_check_mark: | :white_check_mark: | 0 | — | ~6.4 yrs | OK |
| 2TB | Seagate Barracuda ST2000DM008-2FR102 | WFL3ZBBC | :white_check_mark: | :white_check_mark: | — | :white_check_mark: | ~0.3 yrs | PENDING |
| 2TB | Seagate IronWolf ST2000VN004-2E4164 | Z52BBV0P | :white_check_mark: | :white_check_mark: | — | :white_check_mark: | ~0.06 yrs | PENDING |
| 2TB | Seagate Barracuda ST2000DM008-2FR102 | ZFL0TF34 | :white_check_mark: | :white_check_mark: | — | :white_check_mark: | ~4.2 yrs | PENDING |
| 2TB | :warning: Seagate Barracuda Green ST2000DL003-9VT166 | 5YD5PQE7 | :white_check_mark: | :white_check_mark: | — | — | ~0.7 yrs | PENDING |
| 2TB | :warning: Seagate Barracuda Green ST2000DL003-9VT166 | 5YD5VWL1 | :white_check_mark: | :white_check_mark: | — | — | ~5.7 yrs | PENDING |
| 2TB | :warning: Seagate Barracuda ST2000DM001-1CH164 | Z1E9K96R | :white_check_mark: | :white_check_mark: | — | — | ~0.6 yrs | PENDING |
| 2TB | | | | | | | | | |
| 2TB | | | | | | | | | |

I have about 9x 1TB drives but probably won't use them.

## Drive Status Reports

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

