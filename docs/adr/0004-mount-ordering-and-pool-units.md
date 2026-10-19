# ADR-0004: Mount ordering, branch identity and pool unit layout

- Status: **Proposed** (M0, awaiting review). Applies to *managed* pools (M3+). Imported pools are read-only and only *evaluated* against these rules.
- Date: 2026-10-19

## Context (verified upstream behavior)

- mergerfs deliberately does **not** require branches to be mounted: "It doesn't require
  branch paths to be mounted or to match a specific filesystem at startup"
  (`mkdocs/docs/config/branches-mount-timeout.md`). A missing literal path is retained as a
  branch. With a disk absent, writes can land on the root filesystem underneath the
  empty mountpoint directory, and later be **shadowed** when the real disk mounts over it.
- `branches-mount-timeout` defaults to `0`. `branches-mount-timeout-fail` (fail the mount
  if branches never appear) exists only from **2.41.0**; every target distro ships 2.40.2.
- mergerfs's "mounted" heuristic is: xattr `user.mergerfs.branch` or file
  `.mergerfs.branch` ⇒ mounted; `.mergerfs.branch_mounts_here` ⇒ **not** mounted; else
  `st_dev` differs from the pool mountpoint ⇒ mounted. The last test only proves "some
  filesystem", not "the right filesystem".
- Upstream's documented systemd pattern is a `Type=simple` service running `mergerfs -f`
  with `RequiresMountsFor=<branches>`. `RequiresMountsFor=` adds dependencies only on mount
  units that exist; a branch path that is a plain directory satisfies it trivially.

## Decision (proposed)

For each managed pool `P` with branches `B1..Bn`:

1. **Branch filesystems** are mounted by plugin-generated native mount units
   `/etc/systemd/system/<escaped-path>.mount` with `What=/dev/disk/by-uuid/<UUID>`
   (UUID chosen at adoption, after duplicate-UUID check), explicit `Type=`, and
   `Options=` including `nofail` so a dead disk does not block boot. Not fstab, so we never
   edit a shared file for managed branches. Preflight refuses if fstab or another unit
   already claims the same path.
2. **Branch identity marker:** at adoption, write `<branch>/.cpm-branch.json`
   (`{"pool_id", "branch_id", "fs_uuid", "created"}`) on the branch filesystem, and
   create `.mergerfs.branch_mounts_here` in the **underlying** directory *before* mounting,
   so that mergerfs itself treats a bare directory as not-mounted.
3. **Pool unit** `cpm-pool-<pool_id>.service`:
   - `Requires=` + `After=` every branch mount unit (not just `RequiresMountsFor=`).
   - `ExecStartPre=/usr/libexec/cockpit-pool-manager/cpm-helper verify-pool <pool_id>` —
     verifies, for every branch: is a mountpoint (`findmnt --mountpoint`), source resolves
     to the expected UUID, `.cpm-branch.json` matches, not read-only unless configured so.
     Any failure ⇒ non-zero exit ⇒ pool does **not** mount. This works on mergerfs 2.40.2.
   - `ExecStart=mergerfs -f -o <validated options> <branches> <mountpoint>`.
   - When mergerfs ≥ 2.41 is detected, additionally set `branches-mount-timeout=N` and
     `branches-mount-timeout-fail=true` as defense in depth.
4. **Consumers** (Samba, NFS, SnapRAID jobs) for paths on the pool get drop-ins
   `Requires=cpm-pool-<id>.service` + `After=`; share preflight also checks the live mount.
5. **Runtime disappearance:** pool unit uses `BindsTo=` on branch mount units? — **Open
   question Q4.** `BindsTo=` unmounts the pool when a branch vanishes (fail closed, but
   disruptive to open files and shares). Alternative: leave pool up, mark degraded, block
   all jobs and auto-suspend timers. Default proposal: *do not* `BindsTo`; block jobs and
   raise a critical banner, because an abrupt unmount under Samba clients can itself
   cause data loss for in-flight writes. Needs owner decision.

## Consequences
- Pool cannot come up on a bare directory, regardless of mergerfs version.
- Unit names derive from paths via `systemd-escape --path`; must be computed by the
  helper (allowlisted `systemd-escape`), never by the UI.
- Requires integration tests: branch absent at boot, wrong disk mounted at branch path,
  disk removed at runtime, marker missing, duplicate UUID.
