# Recovery Simulation / Replacement Readiness

Status: M0 closeout design (R3, R6). Decision record: ADR-0010. Implementation: M4 (read-only
foundation), consumed by M8/M9 destructive workflows.

## 1. Purpose and non-claims
The feasibility engine answers, **without changing anything**: "If I start this replacement or
recovery now, what is known, what is missing, what could go wrong, and what exactly would happen?"

It does **not**:
- perform or rehearse a SnapRAID recovery (static tier);
- validate parity or prove file integrity (even the optional `check` tier only reports what
  `check` reports);
- guarantee success. `READY` means "all verified prerequisites satisfied".

Forbidden from the engine (static test enforced): `mkfs*`, `wipefs`, `sgdisk`/`parted`/`sfdisk`,
`snapraid fix|sync|touch|scrub`, `mount`, `umount`, `chattr`, `setfattr`, mergerfs runtime writes,
any file write outside the engine's own report directory.

## 2. Scenario types (R3)
| Scenario | Applies to mode | Core mechanism of the eventual plan |
|---|---|---|
| S1 Healthy disk replacement (capacity upgrade) | any | copy → verify → reconfigure → (parity modes: exceptional sync) |
| S2 Planned retirement / evacuation | any | move files to other members → verify → remove member |
| S3 Failed SnapRAID data disk | parity, combined | upstream §4.4: freeze writers → reconfigure → `fix -d` → `check -a -d` → decide |
| S4 Failed parity disk | parity, combined | reconfigure parity path → `fix -d parity` → `check` → decide |
| S5 Failed content-file storage | parity, combined | restore a valid copy to a new location → verify → update config |
| S6 Disk in a mergerfs-only pool | pooling | evacuate if readable; otherwise report loss — **no parity recovery exists** |
| S7 Multiple simultaneous failures | any | combination of the above, capacity-checked as a whole |

## 3. Evidence sources (all read-only)
Inventory (`lsblk`, `findmnt`, `blkid`, `/proc/self/mountinfo`), SMART (`smartctl -a`, optional),
SnapRAID config model, `snapraid status`, `snapraid diff`, `snapraid list` (all with
`--log ">&2"`; exit codes per manual §15), content-file metadata (stat only), our state store
(job history, recorded identities), schedulers (systemd timers/units, cron tables, SnapRAID
Daemon units), running processes (`/proc`), mergerfs runtime config (`getxattr` on `.mergerfs`).
Each finding records its **evidence source and confidence** (`high` = direct tool/system result,
`medium` = derived, `low` = heuristic).

## 4. Checks (R6 items 1–12)
Each check yields `pass | warn | block | unknown`. `unknown` on a check marked *required* makes the
verdict `INDETERMINATE`.

| # | Check | Block when | Unknown when | Required |
|---|---|---|---|---|
| C1 | Expected disk and filesystem identity | target or replacement stable_id/UUID duplicated or ambiguous; a "surviving" disk does not match its recorded UUID | recorded identity absent (imported array never inventoried) | yes |
| C2 | Roles of missing, failed and surviving disks | a disk's role conflicts between pool, array and inventory; replacement disk already has a role | role cannot be derived | yes |
| C3 | Parity levels and accessible parity files | (S3) a needed parity level has any of its files inaccessible | parity file presence/size unreadable | yes (S3/S4/S7) |
| C4 | Surviving content-file copies | zero valid copies | copies disagree in size/mtime and no tie-break evidence | yes (parity modes) |
| C5 | Failure pattern vs parity capacity | failed data disks > accessible parity levels | — | yes (S3/S7) |
| C6 | Last-successful-sync evidence | — | `status` unavailable, or reports unsynced blocks (interrupted sync) with no later completed sync | yes (parity modes) |
| C7 | Known changes since sync (surviving disks) | — (informational; thresholds of ADR-0005 do not apply) | `diff` cannot run | no |
| C8 | Replacement capacity and layout | replacement filesystem available bytes < used bytes of failed data disk (S1/S3), or < largest data disk (S4); unsupported fs type | size/used figures missing | yes (when a replacement is named) |
| C9 | Required mountpoints available | target path missing, not a mountpoint, or not the replacement's UUID | — | yes |
| C10 | Conflicting activity | a SnapRAID process or csm job holds the array; external scheduler for the array not disabled; SnapRAID Daemon manages the array | process/scheduler scan incomplete | yes |
| C11 | Data not covered by SnapRAID | — (always reported) | — | no |
| C12 | Overwrite risk | replacement destination is non-empty (beyond `lost+found`), is a member of any pool/array, or contains a SnapRAID content/parity file | emptiness not checkable | yes |

Additional scenario-specific checks: S1/S2 source readability (SMART + read test is **not**
performed; only SMART status and recent kernel I/O errors from the journal are read); S2/S6
destination free space across remaining members after `minfreespace`; S6 marks every file
whose only copy is on a failed pooling-only disk as **unrecoverable by this tool**.

## 5. Verdict
```
if any required check == block            -> BLOCKED
elif any required check == unknown        -> INDETERMINATE
else                                      -> READY        (warnings may still exist)
```
No user input can turn `BLOCKED` or `INDETERMINATE` into `READY`; only new evidence can
(rerun after fixing the condition, or the optional verification tier in §7).

## 6. Report contract
```ts
type Verdict = "READY" | "BLOCKED" | "INDETERMINATE";
interface ReadinessReport {
  report_id: string; created_at: string; engine_version: string;
  scenario: "S1"|"S2"|"S3"|"S4"|"S5"|"S6"|"S7";
  subjects: { pool_ids: string[]; array_ids: string[]; failed: StableId[]; replacement?: StableId };
  host_snapshot: { id: string; fingerprint: string };
  tier: "static" | "static+check";
  verdict: Verdict;
  checks: { id: `C${number}`; result: "pass"|"warn"|"block"|"unknown";
            evidence: { source: string; confidence: "high"|"medium"|"low"; detail: string }[] }[];
  blocking_reasons: Reason[];
  warnings: Reason[];
  missing_information: Reason[];
  assumptions: string[];                 // e.g. "content copy /var/snapraid.content is current"
  unrecoverable_categories: { category: string; scope: string; estimate?: string }[];
  last_sync: { completed: boolean|null; at?: string; confidence: "high"|"medium"|"low"|"none" };
  proposed_operations: { order: number; op: string; mutates: boolean; irreversible: boolean;
                         description: string }[];            // execution order, not executed
  replacement_readiness?: { stable_id: StableId; verdict: "suitable"|"unsuitable"|"unknown"; reasons: Reason[] };
  recommended_actions: string[];
  disclaimer: string;                    // fixed text: not a recovery, not proof of integrity
}
interface Reason { code: string; message: string; subject?: string; check: `C${number}` }
```
API: `readiness.evaluate({scenario, subjects, tier})` → `ReadinessReport` (admin; static tier
may run partially as viewer with `unknown` for root-only facts). `readiness.get(report_id)`.
`plan.create` for S1–S7 requires `readiness_report_id`; `job.start` re-evaluates (ADR-0010).

Unrecoverable categories reported, as applicable: files added or changed on a failed disk
after the last sync; files deleted from surviving disks after the last sync (they can break
recovery of other blocks — manual §3 "Limitations", behavior without the `snapshot` option); excluded files; `.snapraidignore`d files;
nested mounts under data paths; hidden files when `nohidden` is set; all data on failed
pooling-only disks; blocks beyond parity capacity.

## 7. Optional non-destructive verification tier
Started explicitly, long-running, cancellable at any time (it writes nothing to the array):
`snapraid -d <name> check` (simulated recovery of that disk) or `check -a` for hash-only
verification of surviving disks. Results are attached as **new evidence** (`confidence: high`)
and the verdict is recomputed. UI wording: "Verification simulated recovery and reported N
unrecoverable files" — never "recovery succeeded".

## 8. Proposed operations (examples, never executed by the engine)
S3, failed data disk `d2`, single parity, replacement `ata-NEW…`:
1. Suspend managed schedules for the array; require external schedulers disabled (C10).
2. Recommend stopping shares/services writing to the array (upstream §4.4).
3. Verify replacement empty and mounted at `/mnt/disk2-new` (C9, C12).
4. Back up and change `data d2 /mnt/disk2/` → `data d2 /mnt/disk2-new/` (mutating, reversible).
5. `snapraid -d d2 -l <job-dir>/fix.log fix` (mutating, irreversible on the replacement disk).
6. `snapraid -d d2 -a check` (read-only verification).
7. Review `.unrecoverable` files and `fix.log`.
8. Optional exceptional sync — **separate plan, separate approval**; reminder that `fix` cannot be retried after syncing.

## 9. Fixtures (mock hosts; each a JSON host snapshot + expected report)
| ID | Fixture | Scenario | Expected verdict | Key expected reasons |
|---|---|---|---|---|
| F01 | Healthy single-disk replacement: d1 SMART OK, replacement empty, larger, ext4, mounted | S1 | READY | warn: new filesystem UUID ⇒ one UUID change allowed with single parity; sync needed after copy |
| F02 | Failed data disk d2 missing, 2-parity, both parity files accessible, 3 content copies valid, status: no unsynced blocks | S3 | READY | warn: changes after last sync on d2 unrecoverable; C7 lists surviving-disk changes |
| F03 | Two data disks failed, single parity | S7 | BLOCKED | C5: failed data disks (2) > accessible parity levels (1); unrecoverable: whole disks beyond capacity (SnapRAID may still recover some blocks — out of v1 scope) |
| F04a | Content copies: one on failed disk, one corrupt (size mismatch), none valid | S3 | BLOCKED | C4: zero valid content copies |
| F04b | Content copies disagree (different mtimes, sizes) and no job record to choose | S3 | INDETERMINATE | C4 unknown: cannot determine current copy |
| F05 | Parity replacement disk smaller than largest data filesystem | S4 | BLOCKED | C8: available bytes < largest data disk (manual §7) |
| F06 | Imported array, no job history, `status` reports unsynced blocks (interrupted sync) | S3 | INDETERMINATE | C6 unknown: last sync incomplete; recommended: optional `check -d d2` tier |
| F06b | Same, but `status` reports no unsynced blocks; sync time unknown | S3 | READY | warn: last-sync time unknown (confidence low); loss window unbounded |
| F07 | Changes after last sync: 1,200 adds and 40 removals on surviving disks | S3 | READY | warn: removals on surviving disks may reduce recoverability of blocks sharing their parity positions |
| F08 | Competing activity: cron entry runs `snapraid sync` nightly; SnapRAID process running | S3 | BLOCKED | C10: external scheduler and running process |
| F09 | mergerfs-only pool, failing (SMART pre-fail) but readable disk, enough free space elsewhere | S6 | READY (evacuation) | warn: no parity; evacuation is the only path; data unrecoverable if disk dies first |
| F09b | mergerfs-only pool, disk already dead | S6 | BLOCKED | unrecoverable: all files only on that disk; recommend backup restore |
| F10 | Replacement disk carries the same filesystem UUID as the failed disk (clone) | S3 | BLOCKED | C1: duplicated/ambiguous identity |

Each fixture has a golden expected report; tests assert verdict, check results, reason codes
and that the engine invoked only allowlisted read-only commands (exec log in mock transport).

## 10. Tests
- Unit: verdict function over all `pass/warn/block/unknown` combinations; per-check functions
  over fixture inputs; report schema validation; forbidden-command static test.
- Golden: F01–F10 (+ variants) in mock mode.
- Integration (disposable VM, M4): build real arrays on loop devices, detach/dm-error a disk,
  run `readiness.evaluate` static and `check` tiers, and assert nothing changed: a manifest of
  every file on every data/parity/content filesystem (path, size, mtime, sha256; atime excluded)
  and the content files' bytes are identical before and after. Whole-image hashes are not used
  because mounting itself updates filesystem superblocks.
- Freshness: change a UUID/mount/config between evaluate and `job.start` ⇒ refused (M9).
