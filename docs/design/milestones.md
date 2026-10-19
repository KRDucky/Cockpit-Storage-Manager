# Milestones, acceptance criteria and key tests

Ordering principle: **read-only before write; simulate before act; reversible before
irreversible; single-object before multi-object; data-preserving before data-destroying.**
Each milestone needs explicit owner authorization. "Done" means all criteria demonstrated
with real test output (`docs/design/test-environment.md` §5).

Legend: **U** unit · **C** UI component · **G** golden fixture · **I** integration (disposable VM) ·
**F** fault injection · **S** security · **R** release. Requirement IDs: R1–R6
(`docs/design/requirements-traceability.md`).

---

## M0 — Discovery and architecture sign-off
Acceptance (final, see `docs/STATUS.md`): ADR-0001…0010 recorded with owner decisions;
R1–R6 coverage and traceability; runtime branch loss design; sync guard specification;
recovery simulation design with fixtures F01–F10; test environment; docs consistent; no code.

## M1 — Skeleton, transport and read-only inventory
Scope: starter-kit-derived build (esbuild, TypeScript, PatternFly 6, pinned Cockpit `pkg/lib`),
manifest (`requires` 337), `csm-helper rpc` with schema validation, `exec.py`,
`system.capabilities`, `inventory.get`, Overview (dependencies) + Disks page, "Open in Storage"
link, mock transport with fixture hosts, `.deb` (`arch: all`), CI tiers T0–T2, test-host interlock (T0 tests).
Acceptance:
1. Installs on Tier-1 targets; "Storage Manager" appears in Cockpit navigation; uninstall leaves nothing outside package paths.
2. **Mandatory Q1 test:** the built package (pkg/lib from Cockpit 369) loads and passes the C-suite on Cockpit 337, 360 and 367 shells; any incompatibility recorded with a decision.
3. Disks by stable_id with aliases; `/dev/sdX` secondary text only.
4. Missing tools are explicit states; non-admin users see root-only fields as "needs administrative access".
5. LUKS/LVM/md/ZFS/btrfs/loop/root/boot/swap classified inspect-only or excluded, with reasons.
6. Duplicate filesystem UUIDs raise a critical finding.
7. `cockpit.jump("/storage")` works when `cockpit-storaged` is installed and is hidden otherwise.
Key tests: U parsers incl. newline/space/Unicode/colon names; U tool failure/timeout/output cap; S invariants I1, I2, I5 (fuzz); C Disks table keyboard/sort/filter; I `/dev/sdX` reorder keeps stable_id mapping.

## M2 — Read-only import (MVP candidate) — R1, R2 (discovery), R5 (detection)
Scope: fstab parser; mergerfs discovery (fstab, units, `.mergerfs` runtime xattrs); SnapRAID
config model + `status` via tag adapter for 12.2/13.0/14.x; **multiple pools and arrays**;
derived pool↔array links; Samba/NFS discovery with provenance-based ownership; Pools, Parity,
Shares pages (read-only); first-run "Inspect existing setup"; diagnostics bundle with redaction;
VM image builder (T3).
Acceptance:
1. Lossless SnapRAID config round-trip on corpus; otherwise takeover disabled.
2. Pooling-only, parity-only, combined and multi-pool/multi-array fixture hosts render correctly; no page assumes another exists.
3. Pool page flags: branch not mounted, bare directory without L1 guard, wrong UUID, option drift.
4. Parity page shows last-sync evidence and confidence; never "Protected".
5. Ownership with evidence; 45Drives detection via settings file / registry comment, not package presence; uncertain ⇒ read-only.
6. **Zero host writes**, proven by before/after manifests of `/etc`, state dirs and all data filesystems in T3.
Key tests: G fixture hosts; U tag parser per version incl. `scan:copy` order; I partially mounted pool; I missing disk with empty mountpoint; S bundle redaction.

## M3 — Plan engine, jobs, locks, reconciliation; first safe operations
Scope: plan create/validate/hash, stable-ID binding, job runner via `systemd-run`, event journal,
locks, interrupted-job detection, `state.reconcile` (read-only report), Jobs console; operations
limited to `snapraid status`, `diff`, `scrub`, `check` (never `fix`, never `sync`).
Acceptance: stale plans refused; conflicting jobs `E_LOCKED`; jobs survive browser close;
killed job ⇒ `interrupted`; cancel only at safe phases; reconcile report lists every drift between
recorded and discovered state and changes nothing.
Key tests: U transition table; U lock-order property test; I scrub interrupted; S stale-plan replay, tampered hash.

## M4 — Recovery Simulation / Replacement Readiness (read-only foundation) — R3, R6
Scope: feasibility engine (ADR-0010, `docs/design/recovery-simulation.md`) for scenarios S1–S7,
static tier and optional `check` tier, `readiness.evaluate/get` API, Recovery Readiness UI,
fixtures F01–F10.
Acceptance:
1. All golden fixtures produce the expected verdict, checks and reason codes.
2. Static test: no mutating executable reachable from the engine.
3. T3: arrays on loop devices with a failed disk — evaluation changes nothing (manifest proof).
4. UI never labels the result a recovery, parity validation or integrity proof.
Key tests: U verdict function; G F01–F10; I static + `check` tiers; S forbidden-command test.

## M5 — Sync guard, manual sync and schedules — ADR-0005
Scope: anomaly suspension (D4), thresholds (D5), diff fingerprint + review UI (D6), scheduled vs
exceptional sync (D7), revalidation (D8), no-retry (D9), optional quiesce mode (D10), audited
threshold settings (D11), systemd timers.
Acceptance: 1,000 deletions after a mis-mount ⇒ suspended; any D4 anomaly suspends regardless of
counts (incl. empty and tiny arrays); approval invalidated by **any** file-level change (rename of
one file); no automatic retry; forbidden flags absent; SnapRAID Daemon/external scheduler ⇒ our
scheduling disabled.
Key tests: I mass deletion; I missing disk; I TOCTOU single-file rename after approval; U fingerprint canonicalization; S forbidden-flag static test.

## M6 — Managed pools and non-destructive adoption — R1, R2, ADR-0004, ADR-0007
Scope: adoption wizard for ext4/XFS (no formatting, no data relocation), UDisks2 fstab spike and
implementation, pool units with `verify-pool`, L1 guard setup, L2 branch guard + check timer,
fail-closed fallback, managed SnapRAID config creation (opt-in, separate wizard), import/takeover
with backup.
Acceptance: adopted disk readable after reboot, data and ownership unchanged (manifest);
existing fstab lines/units/marker files never overwritten; conflicts with fstab, units and
external tools detected; all runtime-branch-loss tests RBL-01…14 pass on Tier-1 targets before
degraded mode is enabled, else fail-closed is the default.
Key tests: I/F RBL-01…14; I reboot persistence; I concurrent admin edit; F power loss during config update.

## M7 — SMB/NFS shares — R5, ADR-0006
Acceptance: foreign/unknown shares never modified; handoff to File Sharing; duplicate names,
overlapping exports and path conflicts detected; failed reload restores and reports state;
no guest/world-writable/wildcard defaults; removing a share never touches data; bare pool
directory never exported.
Key tests: I 45Drives fixture with relocated exports path; I reload failure; S injection in names/paths/comments.

## M8 — Disk lifecycle: initialize, add, retire/evacuate, mergerfs-only replacement — R2, R3 (S2, S6)
Scope: blank-disk initialization via Cockpit Storage handoff (ADR-0007 D4), add to pool/array
(sync separately approved), evacuation with verified copy, remove known-empty member,
mergerfs-only replacement; every plan gated by a fresh readiness report.
Acceptance: evacuation verifies checksums before any config change; removal never deletes files;
excluded device classes never selectable; readiness `READY` required at `job.start`.
**Release gate:** fault-injection suite + independent safety review.

## M9 — Replacement and recovery — R3 (S1, S3, S4, S5, S7), ADR-0010
Scope: healthy replacement (copy/verify), failed data disk (upstream §4.4 procedure), parity
replacement, content-file restore/relocation, multiple failures within parity capacity;
schedules frozen; external schedulers resolved; recovery journal; `check` after `fix`; sync only
as a separately approved exceptional sync.
Acceptance: no path from any recovery plan to `sync`; readiness re-evaluated at `job.start`;
interrupted recovery ⇒ `partial` with next steps; journal complete and auditable.
**Release gate:** as M8.

## M10 — Packaging and release hardening
Signed `.deb`s, reproducible build docs, SBOM, checksums, upgrade/uninstall preserving config and
state, user documentation; any incomplete workflow marked `Preview / not implemented`.
