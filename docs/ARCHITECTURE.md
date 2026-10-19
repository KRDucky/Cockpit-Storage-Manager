# Architecture and technical contracts

## High-level shape
Cockpit package (React + TypeScript, PatternFly) → Cockpit bridge authenticated transport → small privileged helper/service API (Python or Rust; decision via ADR) → discovery/state parsers + validated declarative plan engine → well-defined Linux executables/systemd/managed snippets. Avoid a persistent daemon unless justified.

UI must not call arbitrary shell commands or construct its own privileged command strings. Use structured, versioned API actions with schemas and deterministic errors. Backend must never trust frontend-only checks.

## Data model (implement as typed contracts)
- Device: physical ID, path aliases, logical partitions, physical topology, SMART status, capacity, removable/hotplug, holders and aliases.
- Filesystem: UUID/type/label/size, stable mount identity, mount options, superblock/signatures, filesystem metadata.
- Mount: path, origin, unit/fstab owner, live status, dependency chain, read-only status.
- Pool: pool ID, branches, union mount, mergerfs option set, live/config consistency, managed/external ownership.
- SnapRAID array: mapping of stable logical data names ↔ actual mounts, parity level/files, content-file placement, exclude rules, sync and scrub state, ownership and errors.
- Share: protocol, name/export target, ownership/ACLs, client access, managed/external status, backend service state.
- Job: ID, actor, plan hash, phases, progress, cancellation policy, timestamps, redacted log, result and rollback status.
- Plan: immutable target identifiers, expected preconditions, actions, config diffs, irreversible effects, postconditions, rollback strategy.

## Discovery
Use established system facilities: `lsblk --json` (explicit fields), `blkid`, `findmnt --json`, `/proc/self/mountinfo`, `systemctl`, `udevadm`, `smartctl` if installed. Correlate block device ancestry and physical disks; robustly handle virtual devices, USB/SATA/NVMe naming, LUKS/LVM, mdraid, ZFS and btrfs. Default to *exclude/unsupported* for complex layers until safe handling is formally specified.

Parse `/etc/fstab` conservatively and distinguish hand-managed entries. Recognize pre-existing mergerfs mounts and equivalent systemd mount units without blindly changing them. Parse SnapRAID config using actual installed syntax and version; preserve unknown directives/comments on read-only import; refuse takeover if roundtrip isn't lossless.

## Managed configuration strategy
Prefer per-feature dedicated managed configs or drop-ins only where genuinely supported. Never assume `/etc/fstab` includes directories or that SnapRAID supports config inclusion without verified docs. If modifying shared files is unavoidable: lock, fingerprint, backup, produce minimal patch, validate, write temp on same filesystem, fsync file+parent, atomic rename, re-read, rollback on validation failure. Protect against parallel operators and races.

Persist installed application state without becoming the source of truth for disk state. A JSON/SQLite inventory may store IDs and plans, but always rediscover and verify physical state before any write.

## Privilege and authorization
Cockpit authentication + narrowly scoped polkit authorization for backend operations. Distinct permissions for status inspection, configuration changes, share updates, manual jobs and irreversible disk initialization. Use allowlisted paths and executables with argument arrays (no shell=True), sanitize environment and working directories, bound output, defend against symlink traversal/TOCTOU/path injection/config injection. Restrict writable config files and tool paths. Do not expose raw root shell.

## Jobs, locking, and recovery
One lock per conflicting resource (device, pool, SnapRAID array, mount/shares config); prevent overlapping sync/scrub/recovery/evacuation/config edits. Jobs run with durable state/logging and explicit preflight revalidation *immediately* before each irreversible phase. Treat partial completion as a first-class state with manual repair instructions. No vague `rollback succeeded` if on-disk changes aren't reversible.

## Mount ordering
Ensure physical filesystems are mounted and positively verified BEFORE mergerfs mount, and mergerfs mount verified BEFORE share services and any consumers. Validate systemd dependencies and mountpoint emptiness/identity; prevent accidental writes into bare directories when expected disk is absent. Tests must simulate branch absent at startup and after runtime disconnect.

## SnapRAID semantics
- Parity covers the state of the latest successful sync, not continuous writes. Multiple parity levels have hard constraints; verify installed SnapRAID version/manual.
- Parity backing capacity must accommodate largest protected data disk (or exact file-based constraints according to official documentation); validate usable capacities not just advertised disk labels.
- Data names ↔ mount paths must remain stable. Avoid accidental mass-deletion sync, especially after mount failures.
- Multiple content files should be stored across independent physical failure domains; warn where this cannot be achieved.
- Keep sync, scrub, check, fix distinct. `fix` is a recovery/mutation command capable of reverting intentional changes. Never expose a one-click `repair everything` button.
- Never auto-replace a failed disk or imply SnapRAID rebuild is equivalent to RAID resilver.

## Share-service integration
Use Samba-supported configuration approach only after determining compatibility with existing instance (registry-backed or other supported method). For NFS, manage a dedicated exports.d file where supported. Avoid conflicting with the 45Drives Cockpit File Sharing plugin or another administrator: detect ownership and offer read-only/delegation rather than duplicate edits. Validate permissions and exports, not merely share definitions.

## Suggested tree
```
package.json
src/ui/                    # Cockpit React + TypeScript
src/api/                   # typed frontend contracts
backend/                   # privileged helper, adapters, validators
backend/adapters/          # lsblk, findmnt, systemd, snapraid, mergerfs, shares
backend/plans/             # no mutation outside plans
packaging/                 # deb/rpm/source packaging, polkit, units
mock/                      # repeatable synthetic hosts
integration/               # isolated VM and loop-device scenarios
docs/                      # ADRs, threat model, UI workflows, operations
```

## Review gates
API schema fixtures and fuzzing; test state transitions; fixture-based command parsing; disposable loopback integration; reboot persistence; privilege review; never run destructive tests against actual host disks.
