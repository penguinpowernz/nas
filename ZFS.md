# ZFS research

Research for my planned NAS setup.

- 12x 2TB drives
- 16GB RAM
- z97 CPU

Some of the drives are of questionable quality. Some have bad blocks, some have SMART health problems, and most are over 10 years old.

I had previously planned 2xRAID1 mirrors, each with 4x drives, combined into one 4TB
partition using LVM for the data.  Plus a 4 drive RAID1 to contain the ESP, OS and
restic snapshots parititons.  It has been bought to my attention however that ZFS
might be a far better solution, but it probably would require a different layout.

I need to know linux CLI commands for setting things up, potential layouts.

I am familiar with mdadm and LVM.

Should prefer best practice.

## Goals

- high reliability using numerous drives for redundancy
- prefer redundancy over space, would like ~4TB to store data on
- snapshots (had planned to use restic on a separate mirror, snapshottting hourly)
- server will not be powered on all the time
- no requirement for extreme performance
- I want it to last a decade or longer
- OS should have disk redundancy so I can always boot even if drives fail

## Questions

### Q1: How does using restic snapshots compare to ZFS snapshots

**I had planned to use restic to snapshot to a separate restic repo mirror and snapshot on the hour. How would I setup ZFS snapshots and how would I access them.  How is pruning of snapshots done in ZFS? How can I access files from the snapshots?  Can the snapshots restore the entire disk? How does the disk usage of snapshots compare to restic which has compression and block level dedup?**

#### ZFS snapshots vs restic

| Feature | ZFS snapshots | restic |
|---|---|---|
| Where backup lives | Same pool (same disks) | Separate repo (different disks/remote) |
| Off-site/off-pool protection | No — drive failure destroys data AND snapshots | Yes — repo can be on remote storage |
| Compression | Yes (zstd/lz4, per dataset) | Yes (built-in) |
| Deduplication | Block-level dedup available but **avoid it** (huge RAM cost, write perf killer); compression + COW gives some natural savings | Content-defined chunking — generally better dedup ratios than ZFS |
| Snapshot overhead | Near-zero at creation; grows only as data diverges from snapshot (COW) | Full chunk scan on each backup run; initial backup is slow |
| Speed | Instantaneous (`zfs snapshot` is metadata-only) | Minutes for large datasets |
| Granularity | Filesystem/dataset level | File-level |
| Full disk restore | Restores dataset contents, not the pool itself; `zfs send` can replicate an entire pool to another pool | Can restore all files; does not restore pool structure |
| Cross-machine restore | Via `zfs send | zfs recv` to another ZFS pool | To any machine with restic installed |

**Verdict for your use case:** ZFS snapshots are fast and free but they live on the same pool — if the pool is lost (catastrophic multi-drive failure, controller death) the snapshots are gone with it. Restic to a separate mirror is still valuable for off-pool protection. You could use ZFS snapshots for hourly/daily convenience restores and restic (or `zfs send`) for true off-pool backup.

#### Setting up automated ZFS snapshots

The standard tool is `zfs-auto-snapshot` or a `systemd` timer. Example with `zfs-auto-snapshot`:

```bash
apt install zfs-auto-snapshot
# It installs cron jobs for frequent/hourly/daily/weekly/monthly automatically.
# Snapshots appear as:  zfs-auto-snap_hourly-YYYY-MM-DD-HHmm
```

Or manually with systemd:
```bash
# /etc/systemd/system/zfs-snapshot-hourly.service
[Service]
Type=oneshot
ExecStart=/usr/sbin/zfs snapshot -r tank/data@%Y-%m-%d_%H%M

# /etc/systemd/system/zfs-snapshot-hourly.timer
[Timer]
OnCalendar=hourly
Persistent=true
```

#### Accessing files from snapshots

ZFS automatically exposes snapshots in a hidden `.zfs/snapshot/` directory inside each dataset mount:

```bash
ls /mnt/data/.zfs/snapshot/
# zfs-auto-snap_hourly-2026-05-31-1400   zfs-auto-snap_daily-2026-05-30-0000 ...

# Copy a file back:
cp /mnt/data/.zfs/snapshot/zfs-auto-snap_hourly-2026-05-31-1400/documents/file.txt ~/recovered.txt

# Or roll back the entire dataset to a snapshot:
zfs rollback tank/data@zfs-auto-snap_hourly-2026-05-31-1400
```

#### Pruning / destroying old snapshots

```bash
# List snapshots with space usage
zfs list -t snapshot -o name,used,refer

# Destroy a single snapshot
zfs destroy tank/data@zfs-auto-snap_hourly-2026-05-30-1400

# Destroy a range (OpenZFS 2.0+)
zfs destroy tank/data@snap1%snap50

# zfs-auto-snapshot prunes automatically via the --keep=N flag
```

#### Disk usage compared to restic

- ZFS snapshots use **zero space at creation**; space accumulates only for blocks that have changed since the snapshot was taken (copy-on-write). A snapshot of a 2TB dataset with 1GB of daily churn costs ~1GB/day.
- restic uses content-defined chunking which typically gives **better dedup ratios** across snapshots, especially for datasets with lots of small files or partial file changes.
- With `zstd` compression enabled on the ZFS dataset (`zfs set compression=zstd tank/data`), ZFS handles compressible data well, but restic still wins on pure dedup efficiency for backup workloads.
- **Practical bottom line:** for hourly snapshots of a slowly-changing NAS, ZFS snapshot space overhead is very modest. Enable `compression=zstd` and you will be fine without ZFS dedup.

### Q2: Booting to ZFS

**I want the OS/ESP to be mirrored as well so I can always boot.  Is it possible to boot to ZFS?  Does it require anything special like UKI or UEFI? Is it worth it to boot to ZFS or would mdadm be preferable?**

#### Is it possible?

Yes. "Root on ZFS" is well-supported on Linux. The OpenZFS project maintains detailed guides for Ubuntu, Debian, and others. The short version: your root filesystem lives on a ZFS mirror or raidz vdev, and a bootloader (ZFSBootMenu or GRUB with ZFS support) loads the kernel from it.

#### What it requires

1. **EFI System Partition (ESP)** — ZFS cannot hold the ESP itself (firmware requires FAT32). You need at least one FAT32 ESP partition that the firmware can read.
2. **Bootloader** — Two options:
   - **ZFSBootMenu** (recommended): a small EFI application that lives on the ESP, understands ZFS natively, can boot any ZFS dataset, and supports boot environments (like OpenBSD). Much more flexible than GRUB for ZFS.
   - **GRUB with ZFS module**: works but is more fragile; GRUB's ZFS support lags behind OpenZFS releases and cannot handle all ZFS features.
3. **No special UKI requirement**, though UKI-style setups work fine alongside ZFSBootMenu.

#### Mirroring the ESP

The ESP cannot be on a ZFS vdev. The standard approach is **mdadm RAID-1 for the ESP** using metadata version `1.0` (metadata written at the *end* of the partition so firmware still sees valid FAT32):

```bash
mdadm --create /dev/md0 --level=1 --metadata=1.0 --raid-devices=2 /dev/sda1 /dev/sdb1
mkfs.vfat /dev/md0
# Mount /dev/md0 as /boot/efi
```

This is actually the recommended pattern from the ZFSBootMenu docs: mdadm for the ESP, ZFS mirror for everything else.

#### Recommendation: hybrid approach

For your use case (server not always on, decade lifespan, drive reliability concerns) the best layout is:

- **ESP**: mdadm RAID-1 across 2-4 drives, metadata=1.0 — simple, firmware-transparent
- **OS root + swap**: ZFS mirror (2 drives, or more) — gets ZFS checksumming and easy snapshots/rollback for OS
- **Data**: separate ZFS pool (remaining drives in raidz2/raidz3)

This is more reliable than GRUB-on-ZFS and simpler to recover from than a pure mdadm/LVM OS setup, because ZFSBootMenu lets you boot into and roll back prior boot environments if an update breaks the system.

#### Is it worth it over mdadm?

| | mdadm RAID-1 + ext4 OS | ZFS mirror OS |
|---|---|---|
| Complexity | Low — familiar to you | Moderate — new tooling |
| Self-healing | No | Yes (checksums + scrub) |
| Boot environment snapshots | No | Yes (via ZFSBootMenu) |
| Recovery familiarity | High | Lower until you learn it |
| Firmware compatibility | Excellent | Excellent (ESP still FAT32) |

**For a decade-long NAS where drives are already suspect**, ZFS's self-healing checksums on the OS partition are genuinely valuable. The learning curve is real but the docs are good. Recommended: ZFS mirror for OS, mdadm RAID-1 for ESP.

### Q3: How to layout the drives

**How to layout the drives so that I can have 4TB to store data on, with OS also being redundant, 3-drive failure tolerance, hot spares, and a separate off-pool restic backup.**

#### Drive inventory recap

- 12× 2TB drives, many old/suspect
- Want: ~4TB usable data, redundant OS, 3-drive failure tolerance, hot spares
- Separate off-pool restic backup
- Prefer redundancy over space

#### Final layout

```
Boot:         3× USB sticks               ← FAT32 ESP + ZFSBootMenu EFI binary
Main pool:    6× HDD raidz3 vdev
              4× HDD hot spares
Backup pool:  2× HDD ZFS mirror           ← restic repo only
```

**Main pool (tank) — 6-drive raidz3 + 4 hot spares:**
- raidz3 = 3 parity + 3 data drives
- Usable: 3 × 2TB = **~5.5TB** — comfortably above the 4TB target
- Tolerates any 3 simultaneous drive failures
- 4 hot spares: when a vdev drive faults, ZFS automatically resilveres onto a spare
- All datasets share this pool: OS root, NAS data

**Backup pool (backup) — 2-drive ZFS mirror:**
- Separate failure domain from the main pool
- Holds only the restic repo — restic backs up `tank/data` here on a schedule
- ZFS checksumming protects the restic repo itself from silent corruption
- Usable: 2TB (one drive's worth)

**Boot (3× USB sticks):**
- Each USB holds a FAT32 ESP with the ZFSBootMenu EFI binary
- UEFI tries each in order — any one USB is enough to boot
- All 12 HDDs are whole-disk ZFS (no partitioning needed)
- Update: when ZFSBootMenu is upgraded, copy the new EFI binary to all 3 USBs

#### Datasets

```
tank/ROOT/ubuntu    ← OS root (ZFSBootMenu boots this)
tank/data           ← NAS data

backup/restic       ← restic repo (backs up tank/data)
```

Set quotas to prevent any one dataset filling the pool:

```bash
zfs set quota=50G   tank/ROOT
zfs set quota=4T    tank/data
# backup pool is 2TB total — restic will fill it naturally up to that limit
```

#### Failure sequence with hot spares

```
Normal:          raidz3 tolerates 3 failures
1 drive fails  → spare resilveres in (hours); 2 failure tolerance during resilver
2nd fails      → 2nd spare resilveres in; 1 failure tolerance during resilver
3rd fails      → 3rd spare resilveres in; 0 tolerance — any further failure = data loss
4th fails      → pool lost
```

Note: resilvering reads every block on every remaining drive — the most stressful moment for old hardware. raidz3 gives maximum runway through this window.

#### Setup commands

```bash
# Main pool (6 raidz3 drives + 4 spares, whole disk):
zpool create -o ashift=12 \
  -O compression=zstd -O atime=off \
  tank raidz3 \
  /dev/disk/by-id/ata-DRIVE_A \
  /dev/disk/by-id/ata-DRIVE_B \
  /dev/disk/by-id/ata-DRIVE_C \
  /dev/disk/by-id/ata-DRIVE_D \
  /dev/disk/by-id/ata-DRIVE_E \
  /dev/disk/by-id/ata-DRIVE_F \
  spare \
  /dev/disk/by-id/ata-DRIVE_G \
  /dev/disk/by-id/ata-DRIVE_H \
  /dev/disk/by-id/ata-DRIVE_I \
  /dev/disk/by-id/ata-DRIVE_J

# Datasets:
zfs create -o mountpoint=/   tank/ROOT
zfs create                   tank/ROOT/ubuntu
zfs create -o mountpoint=/home tank/home
zfs create -o mountpoint=/data tank/data

# Backup pool (2-drive ZFS mirror, whole disk):
zpool create -o ashift=12 \
  -O compression=zstd -O atime=off \
  backup mirror \
  /dev/disk/by-id/ata-DRIVE_K \
  /dev/disk/by-id/ata-DRIVE_L

zfs create backup/restic
```

Always reference drives by `/dev/disk/by-id/...` — never by `/dev/sdX` which can change on reboot.

#### Scrubbing

Schedule monthly scrubs on both pools to detect and repair bit rot:

```bash
# /etc/cron.d/zfs-scrub
0 2 1 * * root zpool scrub tank && zpool scrub backup
```

### Q4: What are the analogues between ZFS and mdadm/LVM

**I want a table that translates mdadm/LVM concepts to ZFS ones.  Do add a note if there is no analogues in ZFS for mdadm/LVM concepts, and include ZFS concepts that have no analog in mdadm/LVM**

#### mdadm/LVM → ZFS translation table

| mdadm / LVM concept | ZFS equivalent | Notes |
|---|---|---|
| `/dev/md0` (RAID array) | **vdev** | A vdev is ZFS's internal redundancy unit. It can be a mirror, raidz1/2/3, or a single disk. You never interact with vdevs directly via device nodes. |
| RAID-1 mirror | `mirror` vdev | `zpool create tank mirror sda sdb` |
| RAID-5 (single parity) | `raidz1` vdev | ZFS uses variable-width stripes; no fixed stripe width like mdadm |
| RAID-6 (double parity) | `raidz2` vdev | |
| *(no mdadm equivalent)* | `raidz3` vdev | Triple parity; no analog in mdadm |
| `mdadm --add` (hot spare) | `zpool add tank spare sdX` | Works similarly; ZFS can use it automatically on failure |
| Physical Volume (PV) | **vdev** member disk | Individual disks are added to vdevs, not directly to pools |
| Volume Group (VG) | **zpool** | A pool aggregates one or more vdevs and manages free space dynamically. There is no need to pre-declare size. |
| Logical Volume (LV) | **dataset** (or **zvol**) | Datasets are filesystems (mounted automatically). Zvols are block devices (for VMs, swap, iSCSI). |
| `lvcreate -L 500G` (fixed-size LV) | *(no direct equivalent)* | Datasets share pool space by default. You can set `quota` and `reservation` properties to constrain them, but space is not pre-allocated unless you set `volsize` on a zvol. |
| `lvresize` / `resize2fs` | *(not needed)* | Datasets have no fixed size — they grow as data is added, up to pool capacity or quota. |
| `mkfs.ext4` / `mkfs.xfs` | *(not needed)* | ZFS datasets are filesystems. There is no separate `mkfs` step. |
| `mount` / `/etc/fstab` | `zfs set mountpoint=` | ZFS mounts datasets automatically on import. `/etc/fstab` entries are not needed (and are discouraged). |
| `mdadm --detail` | `zpool status` | Shows pool health, vdev layout, and any errors |
| `mdadm --manage` | `zpool` subcommands | `zpool attach/detach/replace/remove` |
| `pvdisplay / vgdisplay / lvdisplay` | `zpool list` + `zfs list` | `zpool list` for pool-level info; `zfs list` for dataset-level |
| `vgextend` (add PV to VG) | `zpool add tank mirror sdX sdY` | You can add new vdevs to expand a pool. You **cannot** add a single disk to an existing raidz vdev (only whole new vdevs). |
| LVM snapshots (`lvcreate -s`) | `zfs snapshot` | ZFS snapshots are instantaneous and COW. LVM snapshots require pre-allocated snapshot space. |
| LVM thin provisioning | *(no equivalent)* | ZFS datasets share pool space dynamically — thin provisioning is essentially the default. |
| `mdadm --grow` (reshape raidz width) | *(limited)* | OpenZFS 2.2+ added RAIDZ expansion (add one disk at a time to a raidz vdev), but it is slow and new. Generally you expand by adding a new vdev instead. |
| `dmcrypt` / LUKS (encryption layer) | `zfs set encryption=aes-256-gcm` | ZFS has native per-dataset encryption. No separate dm-crypt layer needed. |
| `mdadm --monitor` (event daemon) | `zed` (ZFS Event Daemon) | Watches for pool events and sends email/notifications on failure. Ships with `zfsutils-linux`. |

#### ZFS-only concepts with no mdadm/LVM analog

| ZFS concept | What it does |
|---|---|
| **Scrub** (`zpool scrub`) | Reads every block and verifies checksums; repairs silent corruption using parity/mirror data. mdadm has `--check` but without checksums it cannot detect which copy is corrupt. |
| **Checksum** (per-block) | Every block has a cryptographic checksum stored separately. Detects bit rot that RAID alone cannot. |
| **Copy-on-Write (COW)** | ZFS never overwrites data in place. Writes always go to new blocks. This makes snapshots free and avoids the "write hole" problem of mdadm RAID-5/6. |
| **Boot environments** | Entire OS snapshots that ZFSBootMenu can boot into. Lets you safely update and roll back if something breaks. |
| **`zfs send` / `zfs recv`** | Stream a dataset (or incremental snapshot diff) to another pool or remote host — a built-in replication mechanism. No mdadm/LVM equivalent. |
| **`zfs inherit`** | Properties (compression, encryption, quota) cascade from parent datasets to children and can be overridden per-child. |
| **Deduplication** (block-level) | Avoid in practice — RAM cost is ~5GB per 1TB of deduplicated data, and write performance degrades severely. |
| **ARC / L2ARC** (adaptive cache) | ZFS manages its own RAM cache (ARC) and optionally an SSD cache tier (L2ARC). The kernel page cache is bypassed. This is why ZFS wants as much RAM as possible. |
| **SLOG / ZIL** | Separate Intent Log — an optional SSD used to accelerate synchronous writes. Analogous in purpose to a RAID controller's battery-backed write cache. |

### Q5: Is 16GB RAM enough for ZFS?

**The system has 16GB RAM. Is that sufficient for ZFS, given the pool size and workload?**

#### The "1GB per TB" rule — and why it's mostly a myth

The old rule of thumb was 1GB RAM per 1TB of raw storage. By that measure, 12× 2TB = 24TB would demand 24GB. In practice this rule comes from enterprise ZFS deduplication tables (DDTs), which are enormous. **If you don't use ZFS deduplication (and you shouldn't — see Q1), the rule doesn't apply.**

#### What ZFS actually uses RAM for

| Use | Typical size | Notes |
|---|---|---|
| **ARC** (read cache) | Up to ~50% of RAM by default | ZFS caps itself; on 16GB it will use ~7-8GB for ARC, leaving the rest for the OS |
| **Pool metadata** | Hundreds of MB for a ~14TB pool | Scales with number of files, not raw pool size |
| **ZIL / write buffer** | Small (tens of MB) | Only relevant under heavy synchronous write load |
| **Kernel + OS** | ~1-2GB | |

#### Verdict: 16GB is fine for this workload

For a home NAS with:
- No ZFS deduplication
- `compression=zstd` enabled
- No VMs or databases running on the NAS
- A ~10-14TB raidz2 pool of bulk media/backup files

16GB is **comfortably sufficient**. The ARC will have ~7-8GB to cache hot data, which is plenty for a NAS serving one or a few users. 8GB would even work, just with a smaller cache.

#### When 16GB becomes a concern

- **ZFS deduplication**: DDT needs ~5GB RAM per 1TB of deduplicated data. A 14TB deduplicated pool would need ~70GB RAM. This is the main reason to avoid dedup.
- **Many small files**: millions of tiny files (e.g. a mail server) increase metadata pressure. A media NAS with large files is fine.
- **L2ARC**: if you add an SSD as an L2ARC cache, ZFS uses ~100MB of ARC RAM per 10GB of L2ARC to store the L2ARC index. A 500GB SSD L2ARC would consume ~5GB of RAM for its index — eating into your ARC. Don't add L2ARC unless you have ≥32GB RAM.
- **Running VMs or containers** on the same machine alongside ZFS.

#### ARC tuning (optional)

ZFS will self-tune, but you can set a hard cap if needed:

```bash
# /etc/modprobe.d/zfs.conf
# Cap ARC at 8GB (leave more for applications):
options zfs zfs_arc_max=8589934592

# Check current ARC usage at runtime:
arc_summary   # or: cat /proc/spl/kstat/zfs/arcstats | grep -E '^(size|c_max)'
```

For a dedicated NAS with no other heavy workloads, leave ARC uncapped and let ZFS manage it.
