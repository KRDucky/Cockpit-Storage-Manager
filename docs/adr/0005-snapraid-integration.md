# ADR-0005: SnapRAID integration, output parsing and sync guard

- Status: **Proposed** (M0, awaiting review)
- Date: 2026-10-19

## Context (verified in upstream manual and HISTORY)

- Exit codes: `0` OK; `1` errors; `2` `diff` found a sync is needed; `3` device command
  completed with a degraded filesystem (`doc/snapraid.txt` §15).
- From 14.0, `--log ">&1"`/`">&2"` emits documented `NAME:VALUE:…` tags (`snapraid_log(1)`);
  paths escape `:`→`\d`, newline→`\n`, CR→`\r`, `\`→`\\`. Some tags only appear with `--gui`.
  Targets ship 12.2 (Debian 13), 13.0 (Ubuntu 26.04), 14.9 (Ubuntu 26.10).
- Built-in sync safety stops: `-E/--force-empty` (all files on a disk missing),
  `-U/--force-uuid` (disk UUID changed), `-Z/--force-zero` (files truncated to 0),
  `-D/--force-device` for fix. These protect only the **extreme** cases.
- `--gui-threshold-removes N` / `--gui-threshold-updates N` abort sync above a count, but are
  marked **undocumented** in `cmdline/snapraid.c`, intended for SnapRAID Daemon.
- 14.0 regression with include/exclude directories, fixed in 14.1.
- An official **SnapRAID Daemon** (`amadvance/snapraid-daemon`) now exists (needs ≥14.0).

## Decisions (proposed)

1. **Two adapters, one contract.** `snapraid_tags` (≥14.0, preferred) and `snapraid_text`
   (12.x–13.x). Every parsed result carries `parser: "tags"|"text"` and the UI labels
   text-derived numbers "best-effort". Both are fixture-tested against captured output
   from each supported version, generated in disposable VMs.
2. **Config: read-only lossless model.** Parser keeps every line (directives, comments,
   blanks, unknown directives) with spans. M2 test: `render(parse(x)) == x` byte-for-byte
   on a corpus including upstream `snapraid.conf.example`. If any line is not understood,
   ownership/takeover is refused (inspection still works).
3. **Never pass `-E`, `-U`, `-Z`, `-D`, `-F`, `-R`, `-W`** from any automated path.
   They are only reachable from a dedicated, separately confirmed recovery plan.
4. **Our own sync guard runs before every sync (manual or scheduled):**
   a. Mount identity check of every `data`/`parity`/`content` path (same as ADR-0004 verifier).
   b. `snapraid diff` (with `--log` where supported); exit 1 ⇒ suspend; exit 0 ⇒ no-op;
      exit 2 ⇒ evaluate counts.
   c. Thresholds (configurable, defaults: removed > 50 files **or** > 1% of tracked files
      on any disk; updated > 500 ⇒ suspend and require human review in UI).
   d. Approved diff is hashed; the sync job re-runs `diff` immediately before `sync` and
      refuses if counts moved beyond a small tolerance (TOCTOU between review and sync).
   e. Where `--gui-threshold-removes` is detected as accepted by the binary, pass it as
      well (defense in depth only; never relied on).
5. **No automatic sync after any anomaly**: missing disk, UUID change, content-file count
   below configured minimum, previous job interrupted, last `fix`/recovery not verified.
   Timers enter `suspended` state, recorded with a reason, cleared only by an admin action.
6. **`fix` is never one-click.** Each recovery plan scopes `-d <disk>` and/or `-f <filter>`,
   shows the "changes since last sync are not recoverable" warning, and is followed by a
   `check` plan — never by `sync` automatically.
7. **Version gates:** refuse managed sync on exactly 14.0 when include/exclude rules exist;
   treat 15.x pre-releases as inspection-only.
8. **SnapRAID Daemon coexistence:** if its unit/config is detected, the plugin will not
   schedule its own sync/scrub for that array (two schedulers on one array is a hazard);
   show status read-only. Upstream `snapraid` uses a lock file, but we do not rely on it
   for scheduling policy.

## Consequences
- More fixture work (per-version captured output), but parsing confidence is explicit.
- Default thresholds will occasionally block legitimate bulk deletes; that is the intended
  trade-off — the UI must make "review and approve this diff" fast.
