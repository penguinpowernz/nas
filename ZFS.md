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

...ANSWER GOES HERE...

### Q2: Booting to ZFS

**I want the OS/ESP to be mirrored as well so I can always boot.  Is it possible to boot to ZFS?  Does it require anything special like UKI or UEFI? Is it worth it to boot to ZFS or would mdadm be preferable?**

...ANSWER GOES HERE...

### Q3: How to layout the drives

**How to layout the drives so that I can have 4TB to store data on, with OS/ESP also being redundant and with snapshots.**

...ANSWER GOES HERE...

### Q4: What are the analogues between ZFS and mdadm/LVM

**I want a table that translates mdadm/LVM concepts to ZFS ones.  Do add a note if there is no analogues in ZFS for mdadm/LVM concepts, and include ZFS concepts that have no analog in mdadm/LVM**

...ANSWER GOES HERE...
