# Milestones, acceptance criteria and key tests

Ordering principle: **read-only before write; reversible before irreversible; single-object
before multi-object; data-preserving before data-destroying.** Each milestone needs explicit
owner authorization. "Done" means all criteria demonstrated with real test output.

Legend: **U** unit · **C** UI component · **I** integration (disposable VM, loop devices) · **S** security · **R** release.

---

## M0 — Discovery and architecture sign-off (this milestone)
Acceptance:
- ADR-0001…0006, component/API outline, plan engine + locks, threat model, milestones, UI wireframes, unsafe-assumption critique committed for review.
- docs/STATUS.md, DECISIONS.md, SAFETY_CASES.md exist.
- No code, no host mutation.
Tests: none (documentation only); markdown/mermaid render check.

## M1 — Skeleton, transport and read-only inventory
Scope: repo skeleton (starter-kit-derived build: esbuild, TS, PF6, pinned Cockpit `pkg/lib`),
manifest, `cpm-helper rpc` with schema validation, `exec.py`, `system.capabilities`,
`inventory.get`, Overview (dependency status only) + Disks page, mock transport + 4 fixture hosts,
`.deb` build, CI on Debian 13 / Ubuntu 26.04 / 26.10 containers.
Acceptance:
1. Package installs on Tier-1 targets; "Pool Manager" appears in Cockpit nav; uninstall leaves no files outside package-owned paths.
2. Disks page lists devices by stable_id with aliases, transport, size, rotational, partitions, fs, mounts; `/dev/sdX` shown only as secondary text.
3. Missing mergerfs/snapraid/smartctl shown as explicit "not installed", page still works.
4. Non-admin user: page works, root-only fields marked "needs administrative access".
5. LUKS/LVM/md/ZFS/btrfs/loop/root/boot/swap classified `inspect-only` or `excluded` with reasons.
6. Duplicate fs UUID across two devices raises a critical finding.
7. Mock mode renders all four fixture hosts without a backend.
8. Open question Q1 (Cockpit 337 shell vs 369 `pkg/lib`) answered with evidence.
Key tests: U lsblk/findmnt/mountinfo parsers on fixtures incl. newline/space/Unicode/colon names; U absent disk, malformed JSON from tool, tool exit≠0, tool timeout, output cap; S invariant I1 (static check), I2, I5 (fuzz 10k requests); C Disks table sort/filter/keyboard; I enumeration-order change (loop devices re-attached in different order) keeps stable_id mapping.

## M2 — Read-only import of existing setups (MVP candidate)
Scope: fstab parser, mergerfs pool discovery (fstab, units, live via `.mergerfs` xattrs), SnapRAID config lossless parser + status (text + tags adapters), Samba/NFS share discovery + ownership attribution (incl. 45Drives), Pools/Parity/Shares pages read-only, diagnostics bundle with redaction, first-run "Inspect existing setup".
Acceptance:
1. Every Tier-1 fixture's `snapraid.conf` round-trips byte-identically; non-lossless ⇒ `roundtrip_lossless=false`, takeover disabled.
2. Pool page shows configured vs live branches and flags: branch not mounted, branch on root fs, missing `.mergerfs.branch_mounts_here`, option drift.
3. Parity page shows last *successful* sync with evidence source, never "Protected".
4. Shares show ownership (`managed/external/foreign_tool/unknown`); 45Drives shares read-only.
5. Partially configured hosts (mergerfs only; snapraid only; shares only) render correctly.
6. Zero writes to the host (verified by an I test comparing file hashes of /etc and state dirs before/after a full UI session).
Key tests: U parsers with property tests; U snapraid text vs tags fixtures for 12.2/13.0/14.9; I pool partially mounted ⇒ degraded + share-exposure warning; I data disk missing with empty mountpoint ⇒ critical finding; S diagnostic bundle contains no serials/hostnames by default.

## M3 — Plan engine, jobs, locks; first safe operations
Scope: plan create/validate/hash, job runner via `systemd-run`, events journal, lock manager, interrupted-job detection, Jobs console; operations limited to `snapraid status`, `diff` (read-only), `scrub`, `check` (no `-a`/fix) on imported arrays.
Acceptance:
1. Plan bound to snapshot; changing a fs UUID/mount/config fingerprint makes it `stale`; `job.start` refuses.
2. Two conflicting jobs: second fails `E_LOCKED` naming the holder.
3. Browser closed mid-scrub: job continues; reopened UI shows progress.
4. Killing the job process ⇒ `interrupted` on next helper start; timers (none yet) would be suspended (unit-tested).
5. Cancel only offered on phases marked `safe`; backend enforces.
Key tests: U exhaustive transition table; U lock ordering property test (no deadlock across random plan sets); I scrub interrupted; I check failure reported with recovery guidance; S stale plan replay, tampered plan_hash.

## M4 — Sync guard, manual sync and schedules
Scope: ADR-0005 guard, diff review/approve UI, manual sync, systemd timers for sync/scrub, auto-suspend, notifications hook (journald + optional email via host MTA only if configured).
Acceptance:
1. 1,000 deletions after mis-mount ⇒ sync suspended; UI shows diff summary; approval required.
2. Data disk missing / UUID changed / content copies below minimum ⇒ timer suspended with reason.
3. Approved diff changes before sync starts ⇒ refuse (TOCTOU test).
4. No code path passes `-E -U -Z -D -F -R -W`.
5. SnapRAID Daemon detected ⇒ our scheduling disabled for that array.
Key tests: I mass-deletion; I missing disk with empty mountpoint; I sync interrupted then next sync; U thresholds; S grep/static test for forbidden flags.

## M5 — Managed pools and managed SnapRAID config (non-destructive)
Scope: adopt existing formatted filesystems (no mkfs), generate branch mount units + pool unit per ADR-0004, edit pool options/branches, create managed snapraid config (opt-in, separate wizard, content copies across failure domains, parity size validation), import/takeover with backup.
Acceptance:
1. Adopted disk readable after reboot with data unchanged (hash-verified).
2. Branch absent at boot ⇒ pool unit fails `verify-pool`; nothing written under the bare dir.
3. Wrong disk mounted at branch path ⇒ pool refuses.
4. Parity smaller than largest data disk ⇒ blocked with explanation.
5. Every config write: backup exists, atomic, re-read verified; failed systemd reload leaves original files intact and reports partial state.
Key tests: I reboot persistence; I runtime branch disconnect/reconnect; I concurrent edit by "another admin" (fingerprint drift); I power-loss injection during config update.

## M6 — SMB/NFS shares
Scope: ADR-0006; create/edit/delete share definitions on verified mounted pools or confirmed plain mountpoints; validation (`testparm`, `exportfs`), reload, read-back verification; 45Drives coexistence.
Acceptance: no guest or wide-open client ranges without separate confirmation; removing a share never touches data; unmounted pool ⇒ share service blocked from exporting bare dir; foreign_tool shares never modified.
Key tests: I partial mount + share; S injection in share names/paths/comments; I 45Drives installed fixture.

## M7 — Disk lifecycle: initialize, add, remove/evacuate
Scope: blank-device initialization (typed confirmation with stable_id, preview partition/fs), add to pool/array (sync separately approved), evacuation with verified copy, remove known-empty branch.
Acceptance: wrong typed ID blocked; loop/LUKS/md/ZFS/root/boot/swap never selectable for erase; evacuation verifies checksums before config change; removal never deletes files.
Key tests: I fault injection between copy and verify; I failed mkfs; I destination full mid-evacuation.
**Release gate:** fault-injection suite + independent safety review before shipping.

## M8 — Replace and recovery
Scope: healthy disk replace (copy/verify), failed data disk recovery via documented `snapraid fix -d`, parity disk replacement, content file relocation/restore; timers frozen throughout; post-recovery `check`; sync only by separate approval.
Acceptance: recovery never triggers sync; limits explained (changes since last sync); interrupted recovery reported `partial` with next steps.
Key tests: I failed disk recovery end-to-end on loop devices; I recovery interrupted; I parity disk failure + content file loss.
**Release gate:** as M7.

## M9 — Packaging and release hardening
Scope: signed `.deb`s, reproducible build docs, SBOM, checksums, upgrade/uninstall tests, user docs.
Acceptance: install/upgrade/uninstall preserve user config and state; release notes mark any `Preview / not implemented` workflow.
