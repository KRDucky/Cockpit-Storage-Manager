# Runtime branch loss: behavior, fail-closed design and test plan

Status: M0 closeout design answering owner conditions for Q4. Decision record: ADR-0004.
Everything marked **expected** is a hypothesis derived from source reading and must be
demonstrated by the tests in §5 before degraded mode is enabled (M6).

## 1. Fail-closed definition
After any branch of a managed pool stops being the verified expected filesystem:
1. **No byte is written to an underlying (bare) directory**, by mergerfs or anyone else. (L1)
2. **No SnapRAID sync, scrub or fix** runs for any array that includes the branch. (L2/L3)
3. **The pool is never reported healthy.** (L3)
4. **Re-enabling the branch for writes is never automatic.** (L2)
If 1 cannot be guaranteed for a pool, that pool does not run degraded: it uses the ADR-0004 §4
fallback and stops.

## 2. What mergerfs does when a branch filesystem vanishes after startup
Sources: 2.40.2 `src/policy_mfs.cpp`, `src/fs_info.cpp` (create filtering), 2.40.2/2.41.0/2.42.0
`src/fs_wait_for_mount.cpp` (startup only), 2.40.2 README (runtime interface).

mergerfs stores branch **paths**, not filesystem identities, and re-resolves each operation
against those paths. There are two different "vanish" cases:

| Case | What the branch path now is | statvfs on it | Create policy result (no guard) |
|---|---|---|---|
| A. Clean unmount (admin `umount`, systemd stop, lazy unmount) | the bare directory on the parent (usually root) filesystem | succeeds, RW, root fs free space | **branch eligible → writes land on root fs** |
| B. Surprise removal / dead device, still mounted | the dead filesystem | fails (`EIO`) | branch skipped (`ENOENT` recorded, next branch tried) |
| C. Different filesystem mounted at the path | another disk | succeeds | **branch eligible → writes land on the wrong disk** |

Case B often becomes case A: systemd is expected to stop (unmount) a mount unit whose backing
device disappears; whether that succeeds depends on open files.

Markers (`.mergerfs.branch_mounts_here`, ≥2.41) are consulted only during the startup wait
and therefore do not protect cases A–C on any version.

## 3. Do branch markers prevent unsafe writes? — per version
| mergerfs | Startup with branch missing | Runtime case A | Runtime case C |
|---|---|---|---|
| 2.40.2 (all targets) | waits if `branches-mount-timeout` set (st_dev test only), then mounts anyway; markers unsupported | **not prevented by mergerfs** | **not prevented** |
| 2.41.x / 2.42.x (Tier 2, upstream) | markers + `-fail=true` can refuse to mount | **not prevented by mergerfs** | **not prevented** |
| Any, with ADR-0004 L1 | pool refuses via `verify-pool` | prevented: bare dir is immutable ⇒ `EPERM` | detected by L2 timer / preflight; window until detection (see §4.4) |

Conclusion: protection must come from L1 (filesystem attribute) and L2 (identity checks),
not from mergerfs.

## 4. Expected behavior per operation in degraded mode (L1 + L2 active)
"Lost files" = files whose only copy is on the vanished branch. Between the loss and L2
applying `=NC` there is a short window (seconds, measured in tests) labelled "pre-NC".

| Operation (local, SMB or NFS client) | Pre-NC | After NC |
|---|---|---|
| Existing open handle to a lost file | case B: `EIO` on read/write. Case A cannot occur with open handles (unmount fails `EBUSY`) except lazy unmount, where the handle continues against the detached fs until closed. | same |
| Create new file / mkdir | policy may select the bare dir → `EPERM` from L1 (client sees permission/access error); other branches unaffected for policies that skip on error | create policies exclude the branch → succeeds on healthy branches (subject to their free space) |
| Rename / move of a lost file | `ENOENT` (file not visible) or `EIO` | same |
| Rename into a directory that only existed on the lost branch | parent not found → `ENOENT`; path clone onto other branches may be attempted per policy | same |
| Copy *from* a lost file | read fails (`EIO`/`ENOENT`) | same |
| Write to an existing file on healthy branches | unaffected | unaffected |
| Unlink / delete of a lost file | `ENOENT` (not visible) | same |
| Directory listing | lost files disappear from listings | same |
| SMB clients (Samba) | errors surface as access-denied or I/O errors; **lost files look deleted** | same |
| NFS clients | stale file handles for lost inodes; errors as above | same |
| Applications (media managers, sync tools) | may interpret missing files as deletions and act on it (re-download, purge DB rows, propagate deletes) | same |

Operator-facing consequences shown in the critical banner: "Files on disk X are currently
invisible, not deleted. Applications and clients may react as if they were deleted.
Consider stopping shares/services until the disk is restored."

### 4.4 Wrong-device remount (case C)
L1 does not apply (the path is a real mount). Detection: `csm-branch-guard` start action if
systemd pulls it in for externally performed mounts (unverified), the `csm-branch-check.timer`
backstop (default 60 s), and every job/UI preflight. Window of exposure = until detection.
On detection: branch set `=NC` (or `=RO`), SnapRAID suspended, critical banner.
Residual risk: writes during the detection window land on the wrong disk; they are not lost,
but they are misplaced. Documented, not hidden.

### 4.5 Concurrent writers
External writers are unaffected by job suspension (ADR-0004 context item 6). L1 covers direct
writers to branch paths in case A. Nothing in this design stops writers to *healthy* branches —
that is intended degraded operation.

## 5. Tests (integration, disposable VM — see `docs/design/test-environment.md`)
| ID | Scenario | Pass criteria |
|---|---|---|
| RBL-01 | Branch absent at boot | pool unit fails `verify-pool`; nothing written under bare dirs; critical finding |
| RBL-02 | Delayed mount (branch appears after 30 s) | pool starts within timeout; no write before verification |
| RBL-03 | Delayed beyond timeout | pool stays down; clear reason |
| RBL-04 | Clean unmount of a branch while idle | L2 sets NC; creates succeed on other branches; bare dir unchanged (hash of listing + `lsattr`) |
| RBL-05 | Unmount attempt with open files | `EBUSY`; pool stays healthy; no state change |
| RBL-06 | Lazy unmount with open handle | handle completes against detached fs; new creates follow RBL-04 |
| RBL-07 | Surprise removal (QEMU hot-unplug / dm-error swap) under write load | `EIO` to writers of that branch; no bytes on bare dir; SnapRAID suspended; banner |
| RBL-08 | Pre-NC window | measure window; every create in window → `EPERM` or other branch; none on bare dir |
| RBL-09 | Wrong-device remount at branch path | detected ≤ timer interval; NC applied; writes in window recorded |
| RBL-10 | Concurrent writers: SMB client, NFS client, local process, direct-to-branch writer | per §4 table; no bare-dir writes |
| RBL-11 | Branch returns (same UUID) | stays NC until admin re-enable after identity verification |
| RBL-12 | Fallback mode (L1 unsupported) | pool stops; mountpoint returns `ENOTCONN` or is guarded; shares do not expose bare pool dir |
| RBL-13 | Pool mountpoint bare dir | pool stopped → share path write attempts `EPERM` |
| RBL-14 | Each Tier-1 root fs type (ext4, XFS, btrfs) | `chattr +i` on mountpoint dir works and mounting over it works |

## 6. Unverified assumptions (each must pass a test above or the fallback becomes default)
1. `chattr +i` on a directory is supported by the parent filesystem and does not prevent mounting over it (RBL-14).
2. systemd stops a mount unit when its backing device disappears (RBL-07).
3. Whether a unit `WantedBy=`/bound to a mount unit is started for mounts performed outside systemd (RBL-09; timer backstop exists regardless).
4. mergerfs skips a dead-but-mounted branch without hanging (RBL-07; bounded by FUSE timeouts).
5. Setting `=NC` at runtime is immediate for in-flight create operations (RBL-08).
6. Killing mergerfs with a busy mount yields `ENOTCONN`, not exposure of the bare directory (RBL-12).

## 7. Fallback when safety cannot be established
Any of: unsupported parent filesystem for L1; L1 attribute missing at pool start; any test
in §5 failing on a supported target; mergerfs version outside the matrix ⇒ the pool runs in
**fail-closed mode** (ADR-0004 §4). The UI states the trade-off: "If any disk in this pool
disappears, the whole pool will stop to prevent writes to the wrong place."
