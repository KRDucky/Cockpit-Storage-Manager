# UI wireframes (text)

Status: M0 proposal. PatternFly 6 components named in brackets. All pages are keyboard
navigable; every disabled control has a tooltip/inline reason; status is never conveyed by
colour alone (icon + text).

## Global frame

```
┌ Cockpit nav ─┐┌──────────────────────────────────────────────────────────────────────┐
│ …            ││ Pool Manager                                  [Admin access: ON  🔓] │
│ Storage      ││ [Tabs] Overview | Disks | Pools | Parity | Operations | Shares |      │
│ ▶ Pool Mgr   ││        Schedules | Settings & Diagnostics                            │
│ …            ││ ─────────────────────────────────────────────────────────────────────│
└──────────────┘│ [Alert banner, sticky, only when critical]                           │
                │  ⛔ Data disk "d2" expected at /mnt/disk2 is NOT mounted.             │
                │     Scheduled sync is suspended. The directory exists but is empty —  │
                │     that is the root filesystem, not your disk.  [View details]       │
                └──────────────────────────────────────────────────────────────────────┘
```

## First run

```
┌ Welcome ─────────────────────────────────────────────────────────────────────┐
│ Detected on this host                                                         │
│   mergerfs 2.40.2 ✔   snapraid 13.0 ✔   smartctl ✔   Samba ✔   NFS server ✖  │
│   Existing: 1 mergerfs mount (/srv/pool, from /etc/fstab)                     │
│             1 SnapRAID config (/etc/snapraid.conf, 4 data, 1 parity)          │
│             3 Samba shares (2 managed by "File Sharing" plugin)               │
│                                                                               │
│ (●) Inspect existing setup   — read-only; nothing on this host is changed     │
│ ( ) Set up new array         — guided; you will review every change first     │
│                                                    [Continue]                 │
│ ℹ This plugin never takes over existing configuration unless you choose       │
│   "Import" later, which first makes a backup and shows the exact diff.        │
└───────────────────────────────────────────────────────────────────────────────┘
```

## Overview

```
┌ Capacity ─────────────────────┐ ┌ Parity ──────────────────────────────────────┐
│ Usable data   38.2 / 48.0 TB  │ │ Parity is snapshot-based.                     │
│ (4 data branches; parity not  │ │ Last successful sync: 2026-10-18 03:12 (23 h) │
│  counted)                     │ │ Changes since then: 1,204 added, 12 removed   │
│ Parity disks  1 × 16 TB       │ │   → not protected until next sync             │
└───────────────────────────────┘ │ Scrub coverage: 61% · oldest block 41 days    │
┌ Health ───────────────────────┐ │ [Review changes]  [Sync…]                     │
│ ⚠ d3  SMART: 8 reallocated     │ └──────────────────────────────────────────────┘
│ ✔ others OK · temps 31–38 °C  │ ┌ Active tasks ────────────────────────────────┐
└───────────────────────────────┘ │ ▸ scrub (array: main)  42% · ETA 1 h 10 m    │
┌ Config & backups ─────────────┐ └──────────────────────────────────────────────┘
│ snapraid.conf  external (read │
│ only) · last backup: none     │
└───────────────────────────────┘
```

## Disks  [Toolbar + Table, expandable rows]

```
Filter: [role ▾] [health ▾] [search…]                      Group by: (●) physical ( ) role
┌──┬──────────────────────────────┬────────┬───────┬──────┬─────────┬────────┬──────────┐
│  │ Disk (stable ID)             │ Role   │ Size  │ Free │ Health  │ Temp   │ Mount    │
├──┼──────────────────────────────┼────────┼───────┼──────┼─────────┼────────┼──────────┤
│▸ │ ata-WDC_WD120…_8CK2        sdb│ data d1│ 12 TB │ 3.1T │ ✔ OK    │ 34 °C  │/mnt/disk1│
│▾ │ ata-ST16000…_ZL2A          sdc│ parity │ 16 TB │ 0.9T │ ✔ OK    │ 36 °C  │/mnt/par1 │
│  │   Partition part1 · ext4 · UUID 5f1e… · label PARITY1                                │
│  │   Mount source verified ✔ (UUID matches) · origin: /etc/fstab (external)            │
│  │   SMART source: smartctl 7.4, direct ATA                                             │
│  │   Actions: [Add to pool] disabled — "Parity disks cannot be pool branches."          │
│▸ │ usb-JMicron_…_0000         sdf│ —      │ 4 TB  │ —    │ ? n/a   │ —      │ —        │
│  │   ⚠ inspect-only: USB bridge reports no unique serial                               │
│▸ │ nvme-Samsung_990…          nvme0n1│ system│ 1 TB│ —  │ ✔       │ 41 °C  │ /        │
│  │   ⛔ excluded: contains root filesystem                                              │
└──┴──────────────────────────────┴────────┴───────┴──────┴─────────┴────────┴──────────┘
```

## Pools

```
Pool /srv/pool    ownership: external (/etc/fstab line 14)    state: ⚠ degraded
Configured branches                Live branches              Identity
  /mnt/disk1   RW                   /mnt/disk1 ✔               UUID 1a2b… ✔
  /mnt/disk2   RW                   /mnt/disk2 ✖ not mounted   ⛔ bare dir on root fs
  /mnt/disk3   RW                   /mnt/disk3 ✔               UUID 9c8d… ✔
Policy: category.create = pfrd (default)  [ℹ what this means]
  "Picks a branch at random, weighted by free space."
minfreespace: 4G (default)
▸ Advanced options (12)        ▸ Show source config line        [Import as managed…]
```

## Parity

```
Array "main"  config /etc/snapraid.conf (external · lossless round-trip ✔)
SnapRAID 13.0 · output parser: text (best-effort) ℹ
Data   d1 → /mnt/disk1 ✔   d2 → /mnt/disk2 ⛔ not mounted   d3 → /mnt/disk3 ✔
Parity 1-parity → /mnt/par1/snapraid.parity  ✔ ≥ largest data disk
Content /var/snapraid.content (system disk) ✔ · /mnt/disk1/.snapraid.content ✔
        ✔ 2 copies on 2 different disks (minimum for 1-parity is parity levels + 1 = 2)
Rules  exclude *.unrecoverable · exclude /tmp/ … (6)  ▸ view
Commands
  [Status] [Diff] [Scrub…] [Check…] [Sync…] disabled:
      "Sync blocked: data disk d2 not mounted. Syncing now would record its files
       as deleted and destroy your ability to recover them."
  Fix / recovery lives in Operations → Recover a failed disk. There is no "repair all".
```

## Operations (wizard catalogue)

```
Choose a task                                         Risk
  ○ Add a disk that already has data (adopt)          Low  · no formatting
  ○ Initialize a blank disk                           ⛔ High · erases the selected disk
  ○ Add a branch to a pool                            Low
  ○ Add a disk to SnapRAID                            Medium · requires a sync afterward
  ○ Remove a disk (evacuate data first)               Medium · long-running copy
  ○ Replace a healthy disk                            Medium
  ○ Recover a failed data disk                        High · uses parity; recent changes may be lost
  ○ Replace a parity disk                             High · array unprotected until resync
  ○ Restore/relocate content files                    Medium
  (items not yet implemented show "Preview / not implemented" and cannot be started)
```

### Wizard pattern (every operation)
```
1 Select target ─ 2 Preflight ─ 3 Plan preview ─ 4 Confirm ─ 5 Run ─ 6 Verify
Step 3 shows:  targets by stable ID · exact config diffs · units to create · commands (argv)
               · irreversible effects (verbatim list) · rollback: full | partial | none
               · "This plan expires in 15 min or if any disk/mount changes."
Step 4 (irreversible only):
  Type the disk ID to confirm erasing it:
  ata-ST8000VN004-2M2101_WSD1ABCD
  [______________________________]   [Erase disk] (enabled only on exact match)
Step 5: job console — phases with ✔/⟳/✖, live log (redacted), Cancel shown only during
        phases marked safe; otherwise "This step cannot be interrupted safely."
```

## Shares

```
Shares                                              [+ New share] 
┌──────────┬────────┬───────────────────┬──────────────┬─────────────┬───────────────┐
│ Name     │ Proto  │ Path              │ Backing      │ Access      │ Managed by    │
├──────────┼────────┼───────────────────┼──────────────┼─────────────┼───────────────┤
│ media    │ SMB    │ /srv/pool/media   │ pool ✔       │ @media rw   │ this plugin   │
│ backups  │ SMB    │ /mnt/disk3/bk     │ ⚠ single disk│ alice rw    │ File Sharing ↗│
│ iso      │ NFS    │ /srv/pool/iso     │ pool ✔       │ 10.0.0.0/24 │ external      │
└──────────┴────────┴───────────────────┴──────────────┴─────────────┴───────────────┘
Removing a share removes only its definition — files are never deleted.
```

## Schedules

```
sync  (array main)   daily 03:00 ± 10 min   ⏸ SUSPENDED since 2026-10-18 03:00
      Reason: diff found 1,000 removed files on d2 (threshold 50).  [Review diff…]
scrub (array main)   weekly Sun 04:00, 8% per run, older than 10 days   ✔ next in 2 d
Overlap: sync/scrub never run concurrently (shared array lock). Timeout: 12 h.
```

## Settings & Diagnostics
```
Dependencies & versions · Support tier: Tier 1 (Debian 13)
Mount scheme for new pools: systemd units (recommended) ℹ
Action history (filter by actor/object) · Job logs · journal for cpm-* units
[Create diagnostic bundle…]  ☑ redact serials ☑ redact hostnames ☑ redact usernames/paths
```
