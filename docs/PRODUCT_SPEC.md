# Product Requirements / UX Specification

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
