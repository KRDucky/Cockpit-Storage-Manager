# UI wireframes (text)

Status: Accepted at M0 closeout (2026-10-19). PatternFly 6 components implied. All pages are
keyboard navigable; every disabled control states why; status is never conveyed by colour
alone. Copy rules: never "Protected", never "rebuild"/"resilver"; parity "reflects the last
sync (point in time)"; typed confirmation is described as a safeguard, not as security.

## Global frame
```
┌ Cockpit nav ─┐┌──────────────────────────────────────────────────────────────────────┐
│ …            ││ Storage Manager                                [Admin access: ON 🔓] │
│ Storage ↗    ││ Overview | Disks | Pools | Parity | Operations | Shares | Schedules |│
│ ▶ Storage Mgr││ Settings & Diagnostics                                               │
└──────────────┘│ [Sticky critical banner — only when critical]                        │
                │ ⛔ Disk "d2" (pool media, array main) disappeared at 14:02.            │
                │    New files now go to other disks. SnapRAID sync/scrub suspended.   │
                │    Files on d2 are invisible, not deleted — clients and apps may     │
                │    react as if they were deleted.  [Details] [Recovery readiness…]   │
                └──────────────────────────────────────────────────────────────────────┘
```

## First run
```
Detected: mergerfs 2.40.2 ✔  snapraid 13.0 ✔  smartctl ✔  Samba ✔  NFS ✖  Cockpit Storage ✔
Existing: 2 mergerfs pools · 1 SnapRAID array · 5 Samba shares (3 managed by File Sharing)
(●) Inspect existing setup — read-only; nothing on this host changes
( ) Set up something new  — choose: [ ] a pool  [ ] a parity array  (either, both, neither)
ℹ Pools and parity are independent. You can use either one without the other.
```

## Overview (aggregates by object; no single-array assumption)
```
Pools                                         Parity arrays
 media  /srv/media  ⚠ degraded (1/4 missing)   main (/etc/snapraid.conf) ⏸ sync suspended
 scratch /srv/scratch ✔ (no parity)            Last completed sync: 2026-10-18 03:12 (high)
                                               Changes since: 1,204 added · 12 removed
                                               Parity reflects the last sync (point in time).
Health: ⚠ d3 SMART pre-fail (reallocated 8)    Active jobs: scrub main 42% · ETA 1 h 10 m
```

## Disks
```
Filter [role ▾][health ▾][search…]        [Open in Storage ↗] (shown only if Cockpit Storage installed)
Disk (stable ID)                     Roles                         Size  Free  Health Temp
▸ ata-WDC_WD120…_8CK2  (sdb)         branch of media · data d1@main 12 TB 3.1T  ✔     34 °C
▸ ata-ST16000…_ZL2A    (sdc)         parity 1 @main                 16 TB 0.9T  ✔     36 °C
▸ usb-JMicron_…        (sdf)         —  ⚠ inspect-only: no unique serial
▸ nvme-Samsung_990…    (nvme0n1)     system ⛔ excluded: root filesystem
Expanded row: partitions, fs UUID, mount source verified ✔, mount origin (fstab line 14,
external), SMART source, actions with reasons ("Partition/format in Cockpit Storage ↗").
```

## Pools (list of all pools; each independent)
```
media  /srv/media   ownership: managed   state: ⚠ degraded   guard: L1 ✔ (degraded allowed)
  Branch        Mode  Mounted  Identity           Linked to
  /mnt/disk1    RW    ✔        UUID 1a2b… ✔       data d1 @ main
  /mnt/disk2    NC*   ✖        —                  data d2 @ main
      * set to no-create automatically at 14:02 — [Re-enable after verification…]
  /mnt/disk3    RW    ✔        UUID 9c8d… ✔       data d3 @ main
  Policy category.create = pfrd ℹ   minfreespace 4G   ▸ Advanced   ▸ Source config
scratch  /srv/scratch  ownership: external (fstab line 22)  state: ✔  guard: ⚠ L1 missing
  ⚠ If a branch disappears, writes may land on the system disk. [Explain] [Plan guard setup…]
```

## Parity (list of all arrays; each independent)
```
main  /etc/snapraid.conf  external · lossless round-trip ✔ · SnapRAID 13.0 · log tags ✔
  Data    d1 /mnt/disk1 ✔   d2 /mnt/disk2 ⛔ missing   d3 /mnt/disk3 ✔
  Parity  1 level → /mnt/par1/snapraid.parity ✔ (available ≥ largest data disk: margin 0.6 TB)
  Content /var/snapraid.content ✔ · /mnt/disk1/snapraid.content ✔ (2 copies; minimum = levels + 1 = 2)
  Not covered: excluded patterns (6) · nested mount /mnt/disk3/vm ⚠ · ▸ list
  Schedule ⏸ suspended — d2 missing (anomaly; cannot be overridden by thresholds)
  [Status] [Diff] [Scrub…] [Check…] [Sync…] disabled: "A data disk is missing. Syncing now would
   record its files as deleted and destroy your ability to recover them."
  Recovery is in Operations → Recovery readiness. There is no "repair all".
```

## Sync review (exceptional sync)
```
Sync suspended for array main: 1,000 removed files on d2 (threshold: 50, or 1% of a disk).
Review every change ▸ by disk ▸ by type ▸ full list (paged, searchable)
Fingerprint 7f3a…c1 · computed 14:20 · expires 15:20
Limits: diff compares size, timestamp and inode only; excluded and nested-mount files are not listed.
☐ Quiesce writers during sync (set pool branches read-only, optionally stop shares) ℹ
[Approve this exact change set and sync]  — if any file changes before the sync starts,
                                            the sync is refused and a new review is needed.
```

## Operations
```
Readiness first — every replacement and recovery starts with a read-only evaluation.
  ○ Adopt existing disks (no formatting)                     Low
  ○ Initialize a blank disk → opens Cockpit Storage, then returns here to adopt   ⛔ erases
  ○ Retire / evacuate a disk                                 Medium · long copy
  ○ Replace a healthy disk (upgrade)                         Medium
  ○ Recover a failed data disk (SnapRAID)                    High
  ○ Replace a failed parity disk                             High
  ○ Restore / relocate content files                         Medium
  ○ Replace a disk in a pool without parity                  evacuation only
  ○ Multiple failed disks                                    High
  (not yet implemented items show "Preview / not implemented" and cannot start)
```

### Adoption wizard (R2)
```
1 Select filesystems ─ 2 Inspect ─ 3 Conflicts ─ 4 Plan preview ─ 5 Apply ─ 6 Verify
2 Inspect: ext4 · UUID 1a2b… · 3.1 TB used · owner/permissions sample · existing mount /mnt/disk1
           (fstab line 14 — will be kept unchanged) · appears in: share "media", array main d1
3 Conflicts: none | ⛔ same UUID on two devices | ⛔ path claimed by fstab and a unit |
             ⚠ owned by another tool → inspect-only
4 Plan: "No data, files, permissions or filesystem roots will be changed." Proposed changes listed
        individually (fstab entry via UDisks2 if none exists · L1 guard on empty underlying dir).
```

### Recovery readiness report (R6)
```
Scenario: Recover failed data disk d2 (array main) → replacement ata-ST18000…_ABC1
Verdict: ✔ READY — all verified prerequisites are satisfied. Recovery may still not restore everything.
         (or ⛔ BLOCKED / ❓ INDETERMINATE, with the reasons first)
Checks  C1 identity ✔ · C2 roles ✔ · C3 parity 1/1 accessible ✔ · C4 content 2 valid ✔ ·
        C5 1 failed ≤ 1 parity ✔ · C6 last sync completed 2026-10-18 03:12 (high) ✔ ·
        C7 changes on other disks: 1,204 added, 12 removed ⚠ · C8 replacement 18 TB ≥ 11.2 TB used ✔ ·
        C9 /mnt/disk2-new mounted, UUID matches ✔ · C10 no competing jobs; cron entry found ⛔→ resolved ✔ ·
        C11 not covered: 6 exclude rules, nested mount ⚠ · C12 replacement empty ✔
Will not be recoverable: files written to d2 after 2026-10-18 03:12 · excluded files · …
Assumptions: content copy /var/snapraid.content is the current one (matches job record)
Planned steps (not executed): 1 suspend schedules … 5 snapraid -d d2 fix … 6 check -a … 8 sync = separate approval
[Run optional verification (snapraid check, read-only, hours; stresses surviving disks)]
[Create recovery plan] (re-evaluated automatically before it starts)
This is an analysis, not a recovery, parity validation or proof of file integrity.
```

## Shares
```
Name     Proto Path              Backing         Access       Managed by          Evidence
media    SMB   /srv/media        pool ✔          @media rw    this plugin         marker + state
backups  SMB   /mnt/disk3/bk     ⚠ single branch alice rw     File Sharing ↗      registry + include comment
iso      NFS   /srv/media/iso    pool ✔          10.0.0.0/24  File Sharing ↗      /etc/cockpit-file-sharing.conf.json
old      SMB   /srv/old          ✖ not mounted   —            unknown (read-only) registry, no tool evidence
Conflicts: ⚠ "iso" and "media" export overlapping paths with different access models.
Removing a share removes only its definition — files are never deleted.
```

## Schedules
```
sync  main   daily 02:51 ± 10 min   ⏸ SUSPENDED 2026-10-18 — reason: d2 missing (anomaly)
scrub main   weekly                  ⏸ SUSPENDED (same array)
Thresholds (main): removed > 50 · removed > 1% of a disk · updated > 500   [Change… (audited)]
External schedulers detected: cron.d/snapraid-runner ⚠ — this plugin will not schedule array main
Rejected syncs are never retried automatically.
```

## Settings & Diagnostics
```
Dependencies & versions · Support tier · Reconciliation report [Run] (read-only)
Action history · Job logs · Recovery journals · journal for csm-* units
[Create diagnostic bundle…] ☑ redact serials ☑ hostnames ☑ usernames/paths
```
