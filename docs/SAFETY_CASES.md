# Safety cases

Each case: hazard → required behavior → control (ADR/design ref) → verifying test → status.
Status: `designed` (M0), `implemented`, `verified` (test passes in CI or disposable VM).
Source requirements: `docs/SAFETY_AND_TESTING.md`. New cases from M0 research marked **(new)**.

| ID | Hazard | Required behavior | Control | Verifying test (milestone) | Status |
|---|---|---|---|---|---|
| SC-01 | Data disk missing, mountpoint dir exists | Sync refuses; timer suspended; critical banner | ADR-0004 verifier, ADR-0005 guard 4a | I: loop disk detached, sync attempted (M4) | designed |
| SC-02 | Pool mounted with a branch absent; writes land on root fs **(new detail)** | Managed pool does not mount; imported pool flagged degraded; shares blocked | ADR-0004 `ExecStartPre`, `.mergerfs.branch_mounts_here` | I: boot with branch absent (M5); detection on imported pool (M2) | designed |
| SC-03 | Wrong disk mounted at a branch / data path | Refuse pool mount and sync | ADR-0004 identity check; SnapRAID `-U` never passed | I: swap UUIDs between paths (M5) | designed |
| SC-04 | `/dev/sdX` reorder | Correct target by stable_id | stable_id everywhere; kname display-only | I: re-attach loops in different order (M1) | designed |
| SC-05 | Duplicate filesystem UUID | Inspect-only; writes blocked `E_AMBIGUOUS_IDENTITY` | inventory finding | U fixture (M1); I cloned loop fs (M2) | designed |
| SC-06 | Mass deletion propagated to parity | Sync suspended pending human review | ADR-0005 thresholds + approved diff hash | I: 1,000 deletions (M4) | designed |
| SC-07 | Diff approved, then data changes before sync **(new)** | Refuse sync | ADR-0005 4d TOCTOU re-check | I (M4) | designed |
| SC-08 | Sync after disk loss undermines recovery | No automatic sync after any anomaly | ADR-0005 rule 5 | I (M4, M8) | designed |
| SC-09 | Recovery followed by automatic sync | Never; `check` plan offered, sync separately approved | ADR-0005 rule 6 | I (M8) | designed |
| SC-10 | External scheduler (cron, snapraid-runner, SnapRAID Daemon) syncs during recovery **(new)** | Detect and block recovery until disabled; re-check before irreversible phases | critique C3 | I fixture with cron entry (M8) | designed |
| SC-11 | Stale plan executed | Refuse `E_PLAN_STALE` | plan-engine §4 | U + I (M3) | designed |
| SC-12 | Concurrent conflicting jobs | Second fails `E_LOCKED` | plan-engine §5 | U property test; I (M3) | designed |
| SC-13 | Config edited concurrently by another admin | Abort, show drift, no overwrite | fingerprinted writes (ADR-0006) | I (M5) | designed |
| SC-14 | Partial failure (mkfs, config validation, systemd reload, mount) | Report exact partial state; originals preserved | job `partial` state; backups | I fault injection (M5, M7) | designed |
| SC-15 | Wrong device erased | Typed stable_id confirmation, server-checked; excluded classes never selectable | ADR-0003 irreversible tier | I wrong-ID attempt; root/boot/swap/loop/LUKS/md/ZFS (M7) | designed |
| SC-16 | Removal treated as evacuation | Config changed only after verified copy or confirmed-empty | milestones M7 | I fault injection between copy and verify (M7) | designed |
| SC-17 | Share exports bare directory of unmounted pool | Share service `Requires=` pool; preflight checks live mount | ADR-0004 §4, ADR-0006 | I partial mount + share (M6) | designed |
| SC-18 | Over-permissive share defaults | No guest, no world-writable, no `*` clients without separate confirmation | ADR-0006 | S + I (M6) | designed |
| SC-19 | Takeover of 45Drives / external config | Never modified; read-only + delegation | ADR-0006 | I fixture (M2, M6) | designed |
| SC-20 | Non-root mutation attempt | Denied by backend regardless of UI | ADR-0003, invariant I2 | S (M1) | designed |
| SC-21 | Injection via labels/paths/names (newline, colon, `-` prefix, metachar) | Rejected or safely encoded; never reaches a shell | threat model T2–T4 | U/S fuzz (M1, M6) | designed |
| SC-22 | Parity too small (by available bytes, not label) **(new detail)** | Block with explanation and options | critique H1 | U size math (M5) | designed |
| SC-23 | Too few content copies / same physical device **(new detail)** | Warn below parity levels + 1 on distinct devices | critique H2 | U (M2) | designed |
| SC-24 | Unsafe cancel | Cancel only at phases marked safe; backend enforces | plan-engine, ADR-0003 | U + I (M3) | designed |
| SC-25 | Interrupted job / power loss | `interrupted` → `partial`; timers suspended; no auto-resume of irreversible phases | plan-engine §3 | I power-loss injection (M5, M7) | designed |
| SC-26 | SnapRAID 14.0 include/exclude regression **(new)** | Refuse managed sync on exactly 14.0 with include/exclude rules | ADR-0005 rule 7 | U version gate (M4) | designed |
| SC-27 | Nested mounts under a data path silently unprotected **(new)** | Finding shown on Parity page | critique C1 | U fixture (M2) | designed |
| SC-28 | Misleading protection status | No "Protected" label; last-sync shown with evidence source | critique H3, wireframes | C (M2) | designed |
