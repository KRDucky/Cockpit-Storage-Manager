# Product Requirements / UX Specification

> **Authority (ADR-0009):** this repository file is authoritative. The section "Amendments — M0
> closeout" at the end is normative and takes precedence over earlier text where they differ.

## Audience
Home lab enthusiasts and Linux NAS administrators. Beginners need a safe guided flow; experts need inspection of exact config and commands.

## Navigation
1. **Overview**: total usable data capacity (not sum of parity), branch capacity, parity disk capacity, last *successful* sync, parity change status, scrub coverage/date, health warnings, active tasks, missing/unmounted disks, backups/config status.
2. **Disks**: physical disks / partitions / filesystems, persistent IDs, model/serial (redact if exported), size, rotational flag, filesystem, mount, data/parity/spare roles, SMART health and temps when available, SMART source, free/used, per-disk usage. Actions only when safe.
3. **Pools**: mergerfs pool creation/edit/import; branches; mount path; policy presets (explain e.g. `category.create=mfs`, `epmfs`); minfreespace; advanced options with version-aware validation; per-branch inclusion/read-only; mount persistence; live vs configured state; verify mount source.
4. **Parity**: SnapRAID config/status/diff/sync/scrub/check; parity level/layout validation; data disk mapping; content file locations; include/exclude rules; discover imported config and preserve comments where possible; show active/unsynced changes and destructive-sync hazards.
5. **Operations**: guided new disk (existing-data adoption versus blank initialization), add to pool, add to SnapRAID, remove after data evacuation, replace healthy disk, replace failed disk, recover from parity, parity disk replacement, content-file restore, disk balancing/move (optional later). Separate each workflow and its effects. Show risk before execution.
6. **Shares**: directories on union pool versus underlying disk, SMB/NFS creation/edit/delete share definitions; default share path must be a verified active mounted pool, user/group access, read-only/guest options, ACLs, network/client restrictions, conflict detection, service validation, and no deletion of share data when removing share definition.
7. **Schedules**: systemd service+timer unit management for sync and scrub, configurable recurrence, overlap exclusion, timeouts, logs and notifications; auto-suspend on ambiguous/missing disks, mount inconsistency, suspicious deletions or SnapRAID errors; manual sync approval for risky changes.
8. **Settings & Diagnostics**: dependency/version checks, mount scheme, read-only import, config backups, export sanitized diagnostic bundle, systemd journal, logs and action history.

## Mandatory workflows
### First-run
Detect host tools and existing config; offer **Inspect existing setup** or **Set up new array**. Never take ownership of existing config without explicit opt-in and backup. Support partially configured hosts.
### Create new pool
Choose independent formatted/mounted data partitions → choose branches → policy preset and min-free → preview mount/config/service changes → apply → verify mount and persistence. SnapRAID setup is distinct, opt-in, requires data and parity mapping and content copies.
### Add disk
Identify physical drive and partitions; offer **Adopt existing filesystem (non-destructive)** or **Initialize blank device (erases selected target)**. New-device wizard defaults to non-destructive. Check mounted, signatures, active services, device mapping and filesystem. If initializing, use typed high-friction confirmation referencing unique stable identifier; preview partition scheme and filesystem. Verify after creation, then add to pool, optionally SnapRAID config; require separate sync approval.
### Remove disk
Offer safe removal with migration/evacuation or remove a known-empty branch. Never remove while data remain by default. Preflight free space on destinations, file integrity, open files, mount dependencies, available parity, last-sync state and data movement plan; produce checkpoint/log. Refuse unexplained missing disk. Update configuration only after verified evacuation; do not treat removal as deleting files.
### Replace disk
Different workflows for **healthy disk to new disk** (copy/verify) versus **failed/missing data disk** (SnapRAID recovery according to documented procedure), **parity disk** and **content file relocation**. Freeze automated sync/scrub if required; explain recovery limits, deletion exposure and parity sufficiency. Verify roles, sizes and identities; do not perform a parity sync until recovery and verification succeeded, with separate explicit authorization.
### Share
Verify path on intended mounted union pool and security model. Do not allow accidental export of unmounted underlying empty directory or accidentally shadow a data path. Validate `testparm` for Samba and export syntax for NFS, safely reload and verify. Handle externally managed shares as read-only or supported imports.

## Important UX details
- Show visual disk diagram with parity separate from data branches; indicate `Parity is snapshot-based` at relevant times, not as a generic green `Protected` label.
- Tables sort by model/role/health/capacity/free; filter; expandable command/config preview; responsive layout, accessible errors and progress.
- Disabled action buttons explain *why*; always provide recovery guidance, not just raw exit status.
- Job console with cancellable operations only where cancellation is known safe; retain logs; display resumability versus rollback properties.
- Advanced mode exposes documented options but does not bypass irreversible-operation safety gates.
- Make shares independent of mergerfs existence: SMB/NFS share management works for normal filesystem mountpoints, with clear warnings if outside managed pools.

## MVP acceptance criteria
- Installable Cockpit navigation entry and dependency-aware read-only overview.
- Disk inventory with reliable persistent identities and mount mapping.
- Existing mergerfs/SnapRAID config read-only inspection without modifications.
- Mock/demo mode to show pool, shares, progress and failure scenarios.
- Automated tests for absent disks, malformed config, command failure, no root, and partial mounts.

## Later release gates
Managed config/pool creation; scheduling; disk lifecycle; SMB/NFS; signed packaging and reproducible release documentation. Avoid shipping advanced destructive workflows until fault-injection tests pass.

---

## Amendments — M0 closeout (2026-10-19)
Sources: owner review of M0 (Q2–Q7) and "M0 Additional Owner Requirements" (R1–R6). Traceability:
`docs/design/requirements-traceability.md`.

### A0. Naming (Q2)
Product, Debian package, Cockpit package directory and repository: `cockpit-storage-manager`.
Internal prefix `csm-` (`csm-helper`, `csm-pool-<id>.service`, `csm-job-<id>.service`),
state `/var/lib/cockpit-storage-manager`, executables `/usr/libexec/cockpit-storage-manager`.

### A1. Independent pooling and parity (R1, ADR-0008)
Three deployment modes are first-class: mergerfs pooling without SnapRAID; SnapRAID without
mergerfs; both combined. The UI never assumes a pool needs parity or that every SnapRAID data
disk belongs to a pool. Pools and Parity are independent pages listing **all** pools and **all**
arrays on the host; links between them are shown as navigation, not prerequisites. No
single-array (Unraid-style) model.

### A2. Non-destructive adoption (R2)
Existing ext4 and XFS filesystems are adoptable without formatting, partitioning or wiping. The
adoption wizard discovers filesystems and mountpoints, filesystem UUIDs and physical disk
identities; detects duplicate UUIDs and ambiguous identities; identifies existing data, mounts,
shares and SnapRAID configurations; preserves contents and ownership; shows every proposed change
before applying; detects conflicts with fstab, systemd mount units and external tools; and falls
back to inspection-only when ownership or safety cannot be established. Adoption never relocates
or deletes files (migration is a separate, explicit, verifiable operation) and never silently
overwrites filesystem roots, existing marker files, mount configurations or service definitions.
Other filesystem types are inspection-only in v1.

### A3. Guided replacement and recovery (R3)
Multi-stage wizards distinguish: healthy replacement (capacity upgrade); planned retirement /
evacuation; failed SnapRAID data disk; failed parity disk; failed content-file storage;
replacement in mergerfs-only pools (evacuation, never parity recovery); multiple simultaneous
failures. SnapRAID recovery is never presented as an automatic RAID rebuild. Every recovery:
verifies identity and role; determines last-sync state with evidence; checks parity levels and
known failed disks; validates content copies; detects competing SnapRAID jobs; suspends managed
synchronization; identifies external schedulers and requires their resolution; shows scope and
limitations; requires explicit approval before modifying disks or data; keeps an auditable
recovery journal. **Never sync automatically after `snapraid fix`.**

### A4. Cockpit Storage integration (R4, ADR-0007)
Partitioning, formatting, LUKS/LVM/MD and generic mounts stay in Cockpit Storage; this plugin
links to it (`cockpit.jump("/storage")`) and returns to adoption afterwards. Persistent mounts
for adopted filesystems are created through UDisks2 so both UIs agree. Only documented Cockpit
interfaces are used.

### A5. 45Drives File Sharing coexistence (R5, ADR-0006)
Share management is enabled per protocol only after ownership detection. Ownership comes from
configuration provenance, never from package presence alone; uncertain ownership is read-only.
Shares managed by File Sharing are shown as externally managed with a handoff. Overlapping
exports, duplicate share names, path conflicts and reload failures are detected and handled.

### A6. Recovery Simulation / Replacement Readiness (R6, ADR-0010)
A priority, read-only feasibility workflow precedes every replacement or recovery and yields
`READY` / `BLOCKED` / `INDETERMINATE` with blocking reasons, warnings, missing information,
assumptions, unrecoverable categories, ordered proposed operations, replacement readiness and
recommended actions. It never claims recoverability from parity count alone, is never presented
as a recovery or integrity proof, separates static preflight from optional `snapraid check`
verification, never runs mutating commands, and is re-evaluated before the corresponding plan
executes. It ships (M4) before any destructive replacement operation.

### A7. Corrections to earlier text
- "indicate `Parity is snapshot-based`" → use "Parity reflects the last sync (point in time)";
  "snapshot" is reserved for SnapRAID 15's filesystem-snapshot feature.
- Mandatory workflow "Add disk → Initialize blank device" → default path is a handoff to
  Cockpit Storage (A4).
- Runtime branch loss (Q4): the pool may stay mounted in a clearly degraded state only where
  the ADR-0004 runtime guard is verified; otherwise it fails closed. SnapRAID operations are
  suspended and a critical warning is raised in both cases.
- Sync guard (Q6): see ADR-0005 — anomalies suspend regardless of counts; thresholds
  (removed > 50, removed > 1% of any disk, updated > 500) require review of the exact diff;
  rejected syncs are never retried automatically.
