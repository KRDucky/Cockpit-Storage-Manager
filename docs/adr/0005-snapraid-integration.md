# ADR-0005: SnapRAID integration, output parsing and sync guard

- Status: **Accepted** (owner decision Q6, 2026-10-19). Thresholds are **provisional defaults**; review after M5 field testing.
- Date: 2026-10-19

## Context (verified in upstream manual and source)
- Exit codes: `0` OK; `1` errors; `2` `diff` found a sync is needed; `3` degraded device command (manual §15).
- `diff` "doesn't check the file data, but only the file timestamp, size, and inode" (manual §5.6).
- `--log ">&1"`/`">&2"` and `scan:*` tags exist in **12.2, 13.0 and 14.x** (source); documented in `snapraid_log(1)` from 14.0. In tag paths a newline is written as `\n`, carriage return as `\r`, colon as `\d` and backslash as `\\` — identically in all three versions (`esc_tag` in `cmdline/support.c`).
- `scan:copy` / `scan:relocate` emit `source_disk:source_path:dest_disk:dest_path` in source (the 14.x manpage lists the opposite order).
- Built-in stops: `-E` all files on a disk missing, `-U` UUID changed, `-Z` file truncated to zero, `-D` for fix. `--gui-threshold-removes/-updates` exist but are undocumented.
- `fix` "unconditionally reverts the file state to the last sync"; `check` "only simulates a recovery and no changes are written" (§5.9, §5.10).
- Recovery procedure §4.4: disable remote connections and scheduled sync/scrub first; after syncing, `fix` can no longer be retried.

## Decisions

### D1. Parsing
One **tag adapter** for all supported versions (12.2+), driven by per-version fixtures captured
in disposable VMs. Exit codes and our own identity checks drive safety decisions; human text
output is never parsed for decisions. Unknown tags are preserved and surfaced, not dropped.

### D2. Config model
Lossless read-only model (`render(parse(x)) == x`). Takeover refused unless lossless.
Multiple arrays per host are supported, each identified by `array_id` = realpath of its config
file (R1).

### D3. Forbidden flags
`-E -U -Z -D -F -R -W` are never passed from scheduled or guarded-sync paths. They are reachable
only from dedicated recovery plans with their own consequence text and confirmation.

### D4. Anomaly suspension (count-independent)
A sync — scheduled or manual — is **refused** and the array's schedule **suspended** when any of
these holds, regardless of change counts, array size or thresholds:
1. any `data`, `parity` or `content` path fails mount-identity verification (ADR-0004 L0), or a filesystem UUID is duplicated or ambiguous;
2. a data or parity disk is missing, or a branch guard (ADR-0004 L2) has fired since the last successful sync;
3. a data path's filesystem UUID differs from the one recorded at the last successful sync (unexpected mount change);
4. a previous SnapRAID job for the array is `interrupted` or `partial`; a recovery (`fix`) has not been followed by a passing `check`; or a parity/content relocation plan is incomplete;
5. `diff` exits `1` or emits any error tag;
6. fewer accessible content-file copies than parity levels + 1, or copies disagree;
7. the config file fingerprint changed since the last successful sync without an applied config plan;
8. another SnapRAID process holds the array, or an external scheduler is detected for the array (critique C3);
9. a version gate applies (exactly 14.0 with include/exclude rules).
These cannot be disabled by threshold settings.

### D5. Count thresholds (provisional defaults)
Evaluated only when D4 passes. Exceeding **any** ⇒ suspend and require review:
- removed files > **50** (array total);
- removed files > **1%** of files tracked on **any single disk** at the last sync
  (a disk tracking zero files counts any removal as exceeding);
- updated files > **500**.
`moved`/`copied`/`relocated`/`restore` entries are reported but not thresholded; `restore`
(inode changed, everything else equal) is shown prominently because it can indicate a disk was
replaced from backup.

### D6. Diff fingerprint
The fingerprint binds an approval to the exact file-level change set **and** its context:

```
fingerprint = sha256( canonical_json({
  "snapraid_version":  "<x.y>",
  "config_sha256":     "<sha256 of config file bytes>",
  "disks":   [ {name, path, fs_uuid, device_stable_id} ... ],      // data + parity, sorted by name
  "content": [ {path, size, mtime_ns} ... ],                       // sorted by path
  "changes": [ [op, disk, path, src_disk|null, src_path|null, detail|null] ... ]
}))
```
- `changes` contains every non-`equal` `scan:*` record from `snapraid diff --log ">&2"`, sorted
  by `(disk, path, op, src_disk, src_path)`. `op ∈ {add, remove, update, move, copy, relocate, restore}`.
  Paths are the **unescaped** tag paths stored as UTF-8 with surrogate escapes for invalid bytes.
  `detail` holds the size/timestamp before→after values when the `update` tag provides them.
- The full change list is stored (root 0600) in the plan directory for review and audit; the UI
  pages through it and shows per-disk and per-op summaries.

**Limitations of these fields (stated in the UI):**
1. Content changes that preserve size, timestamp and inode are invisible to `diff` and therefore to the fingerprint.
2. Excluded files (`exclude`, `.snapraidignore` on ≥14.0) and anything under a nested mount are invisible (SnapRAID does not cross filesystem boundaries).
3. The fingerprint captures the state at the moment `diff` scanned each directory; it is not an atomic snapshot of the array.
4. Timestamp granularity depends on the filesystem; zero sub-second timestamps weaken copy/relocate matching (manual §5.6).
5. A change that is reverted before the final diff (e.g., a file deleted and restored with identical metadata) leaves no trace.

### D7. Scheduled vs exceptional sync
- **Scheduled sync** runs unattended only when D4 passes and D5 thresholds are not exceeded.
- Otherwise the schedule is **suspended** with a recorded reason. The only way forward is an
  **exceptional sync**: an administrator reviews the diff and approves its exact fingerprint,
  producing a one-shot plan. Approval expires (default 60 min) and is invalidated by **any**
  change to the fingerprint — not just by count changes.
- Approving an exceptional sync does not resume the schedule; resuming is a separate action.

### D8. Revalidation and execution
Under the array lock, immediately before `sync`: re-run every D4 check, re-run `diff`,
recompute the fingerprint; mismatch ⇒ **reject**. Where the binary accepts it,
`--gui-threshold-removes=<approved removals>` is also passed so SnapRAID itself aborts if more
removals appear during its own scan (defense in depth only).

### D9. Never retry
A rejected or failed sync is never retried automatically — not by the scheduler, not by the job
runner. Each attempt needs a fresh plan; exceptional syncs need fresh approval.

### D10. Residual concurrency gap and quiescence option
Between the final `diff` and the end of `sync`'s own scan, writers can still add, change or
delete files; SnapRAID syncs what it sees. For scheduled syncs this is accepted (bounded by D5
thresholds on the next run). For **exceptional** syncs the plan offers an opt-in
**exclusive-writer (quiesce) mode**:
- set every branch of the managed pools covering the array's data disks to `=RO` through the
  mergerfs runtime interface, and optionally stop the share services that export them, for the
  duration of final diff + sync; restore afterwards (restoration is a verified phase);
- limitations: does not stop writers that bypass mergerfs (direct branch paths, other mounts),
  does not apply to imported/external pools, and interrupts clients. Shown before approval.
Evaluation of stronger options (filesystem freeze, SnapRAID 15 snapshots) is deferred.

### D11. Threshold adjustment
Thresholds are per-array settings changed only through an audited **settings plan**
(old → new values, actor, reason, timestamp). Values may be lowered freely, including to 0
("always review"). Raising above the defaults shows a consequence warning. There is no "off"
switch, and D4 anomaly checks are not adjustable.

### D12. Recovery never syncs
`fix` plans are always followed by an offered `check -a -d <disk>` plan, never by `sync`.
Sync after recovery is an exceptional sync (D7) with the reminder that `fix` cannot be retried
after syncing (manual §4.4).

### D13. Coexistence
If the SnapRAID Daemon or another scheduler manages an array, the plugin does not schedule
that array and treats it as read-only for sync/scrub (D4.8).
