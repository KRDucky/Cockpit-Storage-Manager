# Mandatory safety case and test matrix

## Golden rules
1. No mutation without preflight, persistent-ID match, human-readable plan, explicit authorization and verification.
2. Data stays untouched until a genuinely necessary step; no format, partition, fsck -y, force mount, parity fix or mass deletion as a side effect of discovery.
3. Treat missing mount as unsafe, not an empty disk. Block sync if branch mount identity differs from expected or any participating disk is missing.
4. Never assume recovery can restore changes since last successful sync. Parity sync after disk disappearance can undermine recovery.
5. Disk removal requires verified data relocation or an explicitly confirmed empty disk; membership change alone is not data evacuation.
6. Show separate consequence and confirmation for formatting, replacing parity, reconfiguring SnapRAID, forcing recovery and changing access permissions.
7. A plan is stale when underlying devices, UUIDs, mounts or config fingerprints change. Refuse to execute stale plans.

## Critical test scenarios
- Existing data disk adopted without formatting and remains readable after reboot.
- Blank test device initialized only after exact typed confirmation; wrong ID blocked.
- `/dev/sdX` enumeration order changes while stable ID remains; correct target selected.
- Same filesystem UUID cloned onto two disks; ambiguous identity blocks writes.
- SnapRAID data disk missing and its expected mountpoint exists empty; scheduled sync refuses to run.
- Pool partially mounted; dashboard shows degraded/unsafe and shares don't silently expose bare mountpoint.
- 1,000 unintended deletions from mistaken mount; diff-based guard suspends sync until reviewed.
- Sync interrupted, scrub error, check failure, recovery interrupted, disk parity undersized, content copies unavailable.
- Config file concurrently edited by another admin; detect fingerprint change and abort/merge deliberately.
- Failed filesystem creation, failed config validation, failed systemd reload, failed mount: report exact partial state; preserve original files.
- Permissions/ACL/SMB/NFS errors; no world-writable default and no automatic guest access.
- Existing 45Drives plugin present; do not claim ownership of its configs.
- Non-root viewer and unauthorized editor: read-only works, writes denied both UI and backend.
- Device path symlinks, traversal, malicious labels, newline names, Unicode, spaces, colon in mergerfs paths, shell metacharacters, huge command output and timeouts.
- Loop devices, encrypted volumes, mdraid/ZFS roots, mounted boot or swap devices are not accidentally selected for erase.
- Runtime branch disappearance/reappearance; UI refresh and safety interlocks re-evaluate.
- Parity disk failure and content file loss: preserve viable recovery paths and avoid premature sync.
- Mixed ext4/XFS branches; permissions and xattrs; test branch policy behavior on full disks.
- power loss simulation during planned config update and between data-copy/verification steps.

## Automated suites
- Unit: typed contract validation, fstab parser, SnapRAID parser/round-trip, size math, role compatibility, error states, path constraints.
- Component UI: accessible wizard steps, confirmation requirements, mock job progress, blocked actions and explanatory messages.
- Integration (privileged disposable VM or isolated test host): sparse-file-backed loop devices, ext4/XFS, actual mergerfs and SnapRAID binaries, mounts, systemd units; reboot tests and fault injection. Never run privileged destructive test suites automatically outside a configured disposable environment.
- Security: authz bypass attempts; malformed JSON and command-injection attempts; temp symlink races; unsafe env vars; unexpected stdout/stderr; concurrency.
- Release: Debian/Ubuntu install/uninstall preserves user configs; upgrade compatibility; source build; package metadata; SBOM and checksums.

## Release sign-off
No production release if disk replacement or removal workflow has not passed fault injection and an independent safety review. Mark incomplete workflows explicitly `Preview / not implemented`, not as functioning controls.
