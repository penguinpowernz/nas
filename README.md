# NAS research

The purpose is to research how to setup/run my network attached storage system.

- Preferably 4TB usable space
- Recover from >2 disk fails 
- NAS will not be powered on all the time
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

- StarTech 4x PCIe card with 6 SATA Ports using ASM1611

## Disk pool

These are the drives I have available:

| Size | Model | Serial | SMART Health | Bad Blocks | Power-On Hours | Temp (°C) | Status |
|---|---|---|---|---|---|---|---|
| 2TB | Seagate NAS ST2000VN000-1H3164 | W1H31BLJ | PASSED | 0 | 56,370 (~6.4 yrs) | 28 | OK |
| 2TB | WD Red WD20EFRX-68EUZN0 | WD-WCC4M1RYFSH9 | PASSED | 16 | 56,393 (~6.4 yrs) | 27 | CAUTION |
| 2TB | WD Red WD20EFRX-68EUZN0 | WD-WCC4M7YANCD5 | PASSED | 0 | 56,393 (~6.4 yrs) | 27 | OK |
| 2TB | Seagate NAS ST2000VN000-1HJ164 | W520VJFQ | PASSED | 0 | 56,368 (~6.4 yrs) | 28 | OK |
| 2TB | | | | | | | |
| 2TB | | | | | | | |
| 2TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |
| 1TB | | | | | | | |

## Drive Status Reports

### W1H31BLJ — Seagate NAS HDD ST2000VN000-1H3164 (sdb)

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

