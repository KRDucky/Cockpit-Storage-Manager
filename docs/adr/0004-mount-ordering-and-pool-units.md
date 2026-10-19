# ADR-0004: Mount ordering, branch identity, runtime branch loss and pool units

- Status: **Accepted with conditions** (owner decision Q4, 2026-10-19). Degraded operation is permitted **only** for pools where the runtime guard (§3, layer L1) has been verified; otherwise the fail-closed fallback (§4) applies. Implementation in M6; detailed behavior in `docs/design/runtime-branch-loss.md`.
- Date: 2026-10-19
- Amended by ADR-0007 (branch mount persistence uses fstab for Cockpit Storage interoperability).

## Context (verified in upstream source, 2026-10-19)
1. mergerfs deliberately does **not** require branches to be mounted; a missing path is kept as a branch.
2. Create policies (verified in 2.40.2 `src/policy_{mfs,pfrd,epmfs,ff,lfs}.cpp`) skip a branch only if: its mode is `RO`/`NC`, `statvfs` fails, the filesystem is read-only, or free space < `minfreespace`. **A bare directory on the root filesystem passes all four tests**, so after an unmount new files can land on the root filesystem and be shadowed when the disk returns.
3. `.mergerfs.branch` / `.mergerfs.branch_mounts_here` markers and `branches-mount-timeout-fail` exist only from **2.41.0** and are read **only** in `src/fs_wait_for_mount.cpp` during the startup wait. They have **no effect at runtime on any version**, and do not exist on 2.40.2 (the version on every target distro). The M0-discovery draft of this ADR overstated them; that claim is withdrawn.
4. 2.40.2 supports runtime branch changes through the `.mergerfs` pseudo-file (`user.mergerfs.branches`, with `=NC`/`=RO`/`=RW`).
5. Cockpit Storage flags filesystems mounted without an fstab entry (`pkg/storaged/filesystem/mismounting.jsx`, `mounted-no-config`) and offers a one-click fix that writes fstab. Native `.mount` units created by us would trigger that warning (see ADR-0007).
6. Disabling plugin jobs does **not** stop external writers: SMB/NFS clients, local services and users writing directly to branch paths.

## Decision

### 1. Branch mount persistence (non-destructive adoption, R2)
- If an adopted filesystem already has a persistent mount (fstab line or systemd unit), it is
  **kept as-is and recorded as an external dependency**; we verify it, never rewrite it.
- If it has none, the adoption plan proposes an fstab entry `UUID=<uuid> <path> <type>
  <options incl. nofail>` written through UDisks2's configuration interface (the mechanism
  Cockpit Storage uses) — ADR-0007. Native `.mount` units are not used.
- Adoption writes **nothing** to the data filesystem itself. Identity is the filesystem UUID
  plus device stable_id; no marker file is placed in the user's data root.

### 2. Startup gate (layer L0)
Pool unit `csm-pool-<pool_id>.service` (`Type=simple`, `mergerfs -f …`, the upstream-documented pattern):
- `Requires=`/`After=` the branch mount units (systemd generates them from fstab).
- `ExecStartPre=/usr/libexec/cockpit-storage-manager/csm-helper verify-pool <pool_id>` checks
  every branch: is a mountpoint; source resolves to the expected UUID; UUID not duplicated;
  L1 guard present on the underlying directory; mount is RW unless configured RO. It waits up
  to a configurable delayed-mount timeout (default 90 s), then fails. Failure ⇒ the pool does
  not mount.
- On mergerfs ≥2.41, `branches-mount-timeout` + `-fail=true` are added as defense in depth only.

### 3. Runtime guard (layers L1–L3) — required for degraded operation
- **L1 underlay guard (hard, zero-latency, version-independent):** the *underlying* directory
  of every branch mountpoint **and of the pool mountpoint** is root-owned, mode 0755, empty,
  and marked immutable (`chattr +i`). Any create/mkdir/rename-into on the bare directory fails
  with `EPERM` — for mergerfs and for direct writers alike. Set up once, while the filesystem is
  unmounted or via a private bind mount of the parent filesystem; shown in the plan; reversible
  (`chattr -i`). On ≥2.41 a `.mergerfs.branch_mounts_here` file is created first, so the
  startup wait also recognises it.
- **L2 runtime branch guard:** `csm-branch-guard@<branch>.service` (oneshot, bound to the
  branch mount unit). When the mount goes away, its stop action sets that branch to `=NC` via
  `user.mergerfs.branches`, so create policies route new files to healthy branches instead of
  failing on L1, suspends all SnapRAID operations for affected arrays, and records a critical
  event. A periodic `csm-branch-check.timer` (not a daemon) re-verifies identity of every live
  branch to catch **wrong-device remounts** that systemd dependencies may not signal. Restoring
  `RW` is **never automatic**: it requires identity verification and an administrator action.
- **L3 visibility:** critical banner; pool state `degraded`, never `healthy`; shares on the pool
  flagged; SnapRAID sync/scrub/fix blocked for affected arrays.

### 4. Fail-closed fallback (when L1 cannot be established or verified)
If the parent filesystem does not support the immutable attribute, L1 cannot be verified, or
the owner opts out of degraded mode: the pool unit additionally gets `BindsTo=` on every branch
mount unit. On branch loss systemd stops the pool. If the FUSE unmount fails because files are
open, stopping the mergerfs process leaves the mountpoint returning `ENOTCONN` (expected
behavior; verified in tests) — no writes anywhere, and no exposure of the bare pool directory
(also L1-guarded where possible). The operational trade-off — every client loses the whole pool,
and in-flight writes fail — is shown before the pool is created.

### 5. Consumers
Samba/NFS shares on a pool and SnapRAID jobs touching its branches are ordered after the pool
unit and re-verify the live mount in preflight; share definitions target the pool path only
when it is a verified mergerfs mount.

## Consequences
- The no-write-to-bare-mountpoint invariant rests on L1, not on mergerfs behavior or plugin jobs.
- Unverified assumptions (immutable attribute on each root filesystem type; systemd stopping a mount unit on device removal; whether `Wants=` on a mount unit fires for externally performed mounts; mergerfs behavior on a dead-but-mounted branch) are listed in `docs/design/runtime-branch-loss.md` §6 and **must be demonstrated by M6 integration tests before degraded mode ships**. If any fails, the fallback in §4 becomes the default.
