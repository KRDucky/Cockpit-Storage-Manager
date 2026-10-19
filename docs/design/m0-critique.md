# M0 critique: unsafe or unverified assumptions

Each item: the assumption (where it appears), what upstream evidence says, and the
proposed resolution. Severity reflects potential for data loss.

## Critical

### C1. "A missing disk is detectable because its mountpoint is empty"
*Where:* SAFETY_AND_TESTING golden rule 3; common mergerfs + SnapRAID practice.
*Evidence:* mergerfs keeps unmounted branches by design and its default
`branches-mount-timeout=0` mounts immediately. With create policies like `pfrd`/`mfs`, new
files can be written **into the bare directory on the root filesystem**, so the "empty"
mountpoint is no longer empty. Consequences:
- User writes made during the outage land on the root filesystem (possibly filling it) and
  are **shadowed and invisible** once the real disk remounts over the directory.
- SnapRAID's own `-E/--force-empty` stop does catch a *fully* absent data disk (all original
  files missing), and `-U` catches a different filesystem at the path. It does **not** catch
  large-but-not-total deletions (a mistaken `mv`/`rm` through the pool, an application
  bug) — those are synced as ordinary deletions.
- Related: SnapRAID does not cross filesystem/mount/subvolume boundaries inside a data disk
  (manual §16.1, warning only). Anything mounted *under* a data path is silently
  unprotected; discovery must surface that as a finding.
*Resolution:* ADR-0004 (identity verifier + `.mergerfs.branch_mounts_here` markers),
ADR-0005 (identity check precedes every sync). Discovery flags any non-empty directory
underneath a branch mountpoint (requires a bind-mount peek of the underlying dir —
implementation detail for M2: `findmnt` + `stat` comparisons; never unmount to look).

### C2. "SnapRAID will refuse to sync dangerous changes"
*Evidence:* Built-in stops cover only extremes (`-E` all files gone, `-U` UUID changed,
`-Z` zero-size truncation). Count-based abort (`--gui-threshold-removes`) is undocumented.
*Resolution:* Our own diff-based guard with thresholds and approved-diff TOCTOU re-check
(ADR-0005). Never pass `-E/-U/-Z/-D/-F/-R/-W` from automation.

### C3. "Freezing our timers freezes all syncs"
*Where:* PRODUCT_SPEC replace/recovery workflow ("Freeze automated sync/scrub").
*Evidence:* Many hosts already run sync via cron scripts or snapraid-runner, and an
official SnapRAID Daemon now exists. We cannot stop those.
*Resolution:* Discovery detects cron entries/timers/units invoking `snapraid` and the
SnapRAID Daemon; recovery wizards show a blocking checklist item "Other schedulers found:
… — disable them before continuing" and re-check before each irreversible phase.

### C4. "Per-action polkit permissions protect against misuse"
*Where:* ARCHITECTURE.md "Privilege and authorization".
*Evidence:* Cockpit admin mode gives every page root channels; the bridge's polkit agent
exists only during escalation (`src/cockpit/superuser.py`).
*Resolution:* ADR-0003. Honest tiering; server-side safety gates; delegated-operator mode deferred.

## High

### H1. "Parity disk ≥ largest data disk" checked on advertised sizes
*Evidence:* Manual §7: the **total space available** for parity files must be at least as
large as the largest data disk. Filesystem metadata and ext4's reserved blocks reduce
available space; same-model parity and data disks commonly fail this late in the array's life.
*Resolution:* Compare parity filesystem *available* bytes (sum across split parity files)
against the largest data **filesystem size**, show margin, and explain fixes (bigger disk,
reserved-blocks setting, split parity files on different disks per manual).

### H2. "Content files: multiple copies on separate failure domains"
*Evidence:* Example config: at least **one copy per parity level plus one**, each on a
different disk. Copies on data disks are explicitly allowed and auto-excluded from sync.
*Resolution:* Validate count ≥ parity levels + 1 on distinct physical devices (stable_id,
not mountpoint). Warn (not block) when a copy is on the boot disk only or on a USB device.

### H3. "Last successful sync" is directly available
*Evidence:* `status` reports per-block sync/scrub age graphs; a plain "last sync finished
OK at T" for syncs run *outside* the plugin is not something we can assume exists across
12.2/13.0/14.x without verifying output fixtures.
*Resolution:* "Last successful sync" is shown only with an evidence source: our own job
record, or SnapRAID log tags (≥14.0), or "unknown — last diff shows N pending changes".
Never fabricate from content-file mtime (scrub also rewrites it).

### H4. Distro mergerfs lacks safety options
*Evidence:* All targets ship mergerfs 2.40.2; `branches-mount-timeout-fail` is 2.41.0+.
*Resolution:* Safety does not depend on mergerfs version (ADR-0004); newer options used
only as defense in depth when feature-detected.

### H5. Text parsing of SnapRAID output is reliable
*Evidence:* Structured tags only from 14.0; Tier-1 targets ship 12.2 and 13.0.
*Resolution:* Dual adapter, per-version fixtures, "best-effort" labelling; safety decisions
use exit codes and our own identity checks rather than parsed prose where possible.

## Medium

### M1. Repo/project naming mismatch
Repo is `cockpit-storage-manager`; project instructions name the product
`cockpit-pool-manager`. Package name, install paths (`/usr/share/cockpit/<name>`), unit
prefixes and polkit IDs must be fixed **before** M1 because renaming later is a migration.
Proposal: product/package `cockpit-pool-manager`, unit prefix `cpm-`. **Owner decision Q2.**

### M2. CLAUDE.md referenced docs that did not exist in the repo
The four spec docs lived only in the claude.ai Project. This branch copies them verbatim
into `docs/` so the repository is self-contained. Future spec edits must be made in one
place (proposal: the repo is the source of truth).

### M3. "Parity is snapshot-based" wording
SnapRAID 15.0 introduces a `snapshot` option meaning *filesystem* snapshots. Using
"snapshot-based" for ordinary parity will confuse users of 15.x.
*Resolution:* UI copy "Parity reflects the last sync (point in time)"; reserve "snapshot"
for the 15.x feature.

### M4. Open-files preflight on removal
Detecting open files on a branch must cover access through the mergerfs mount as well as
direct branch access; `fuser`/`lsof` on the FUSE mount do not attribute to branches.
*Resolution:* M7 design item; require shares/services stopped for the pool during the final
cut-over phase rather than trusting open-file detection.

### M5. Licensing
Repo is GPL-3.0. Cockpit `pkg/lib` is LGPL-2.1-or-later (compatible to include);
PatternFly is MIT; mergerfs (ISC) and SnapRAID (GPL-3.0-or-later) are executed, not linked.
45Drives File Sharing (GPL-3.0) is research-only. SBOM must list bundled npm packages (M9).

### M6. Cockpit `pkg/lib` vs host Cockpit version
The starter-kit pins `pkg/lib` from Cockpit 369 while Debian 13 runs 337. Compatibility of
that JS on a 337 shell is unverified. **Open question Q1**, answered in M1 by running the
built package on each Tier-1 target.
