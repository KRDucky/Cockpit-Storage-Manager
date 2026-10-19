# Safety cases

Each case: hazard → required behavior → control → verifying test (milestone) → status.
Status: `designed` (accepted design, no code), `implemented`, `verified` (test passes in CI or a
disposable VM, with artifacts). Source requirements: `docs/SAFETY_AND_TESTING.md`, owner
decisions Q3–Q6, requirements R1–R6. Updated at M0 closeout (2026-10-19).

| ID | Hazard | Required behavior | Control | Verifying test (milestone) | Status |
|---|---|---|---|---|---|
| SC-01 | Data disk missing, mountpoint dir exists | Sync refused; schedule suspended; critical banner | ADR-0005 D4.1–D4.2; ADR-0004 L0 | I loop disk detached, scheduled sync (M5) | designed |
| SC-02 | Branch missing/unmounted; writes land on the root filesystem | No write reaches a bare directory | ADR-0004 L1 immutable underlay + L2 `=NC`; fail-closed fallback | RBL-01, 04, 07, 08, 10 (M6) | designed |
| SC-03 | Wrong disk mounted at a branch / data path | Pool refuses at start; at runtime detected and branch set NC; sync refused | ADR-0004 L0, L2 timer; ADR-0005 D4.3 | RBL-09; I swapped UUIDs (M6) | designed |
| SC-04 | `/dev/sdX` reorder | Correct target by stable identifier | plan-engine §1 rule 5 | I re-attach in different order (M1, M9) | designed |
| SC-05 | Duplicate / ambiguous filesystem UUID | Inspect-only; writes blocked; readiness BLOCKED | inventory finding; readiness C1 | U fixture (M1); G F10 (M4); I cloned fs (M2) | designed |
| SC-06 | Mass deletion propagated to parity | Sync suspended pending review of the exact diff | ADR-0005 D5–D7 | I 1,000 deletions (M5) | designed |
| SC-07 | Approved diff changes before sync | Sync refused on **any** file-level change | ADR-0005 D6, D8 | I single-file rename after approval (M5) | designed |
| SC-08 | Sync after disk loss undermines recovery | No sync while any anomaly exists, regardless of counts | ADR-0005 D4 | I (M5) | designed |
| SC-09 | Recovery followed by automatic sync | Never; `check` offered; sync is a separate exceptional approval | ADR-0005 D9, D12; invariant I6 | S static check over plan builders (M9) | designed |
| SC-10 | External scheduler / SnapRAID Daemon syncs during recovery | Readiness BLOCKED until resolved; re-checked before irreversible phases | readiness C10; ADR-0005 D13 | G F08 (M4); I cron fixture (M9) | designed |
| SC-11 | Stale plan executed | Refused `E_PLAN_STALE` | plan-engine §1, §4 | U + I (M3) | designed |
| SC-12 | Concurrent conflicting jobs | Second fails `E_LOCKED` | plan-engine §5 | U property test; I (M3) | designed |
| SC-13 | Config edited concurrently by another admin or tool | Abort, show drift, never overwrite | fingerprinted writes (ADR-0006 §2); reconcile | I (M6) | designed |
| SC-14 | Partial failure (config validation, reload, mount) | Exact partial state reported; originals preserved | job `partial`; backups; plan-engine §6 | F (M6, M8) | designed |
| SC-15 | Wrong device erased | Typed stable-ID confirmation (safety control), excluded device classes never selectable, Cockpit Storage handoff by default | ADR-0003, ADR-0007 D4 | I wrong ID; excluded classes (M8) | designed |
| SC-16 | Removal treated as evacuation | Config changes only after verified copy or confirmed-empty member | M8 scope | F between copy and verify (M8) | designed |
| SC-17 | Share exports a bare directory of an unmounted pool | Shares ordered after the pool; pool mountpoint underlay immutable; preflight verifies live mount | ADR-0004 L1 §5; ADR-0006 §4 | RBL-13; I partial mount + share (M7) | designed |
| SC-18 | Over-permissive share defaults | No guest, world-writable or wildcard clients without separate confirmation | ADR-0006 §4 | S + I (M7) | designed |
| SC-19 | Taking over 45Drives or other external configuration | Never modified without a designed migration; read-only + handoff | ADR-0006 §1, §3 | G attribution fixtures (M2); I (M7) | designed |
| SC-20 | Non-root mutation attempt | Denied by the backend regardless of UI | ADR-0003; invariant I2 | S (M1) | designed |
| SC-21 | Injection via labels/paths/names | Rejected or safely encoded; never reaches a shell | threat model T2–T4 | U/S fuzz (M1, M7) | designed |
| SC-22 | Parity too small (available bytes, not label) | Blocked with explanation | readiness C8; critique H1 | U size math (M4, M6); G F05 | designed |
| SC-23 | Too few or invalid content copies | Warn below levels + 1; readiness BLOCKED with zero valid copies | critique H2; readiness C4 | U (M2); G F04a/b (M4) | designed |
| SC-24 | Unsafe cancel | Cancel only at phases marked safe; backend enforces | plan-engine §3 | U + I (M3) | designed |
| SC-25 | Interrupted job / power loss | `interrupted` → `partial`; schedules suspended; no auto-resume of irreversible phases | plan-engine §3 | F power loss (M6, M8) | designed |
| SC-26 | SnapRAID 14.0 include/exclude regression | Refuse managed sync on 14.0 with include/exclude rules | ADR-0005 D4.9 | U version gate (M5) | designed |
| SC-27 | Nested mounts or excluded files silently unprotected | Reported on Parity page and in every readiness report | critique C1; readiness C11 | U (M2); G (M4) | designed |
| SC-28 | Misleading protection or recovery wording | No "Protected", "rebuild", "resilver"; readiness never presented as recovery or integrity proof | wireframes copy rules; ADR-0010 | C copy tests (M2, M4) | designed |
| SC-29 | Adoption modifies user data or filesystem roots (R2) | No writes to data filesystems; existing mounts/markers/units kept | ADR-0004 §1; ADR-0006 §2 | I manifest before/after incl. ownership (M6) | designed |
| SC-30 | Pool assumed to need parity, or array assumed pooled (R1) | Each mode works alone; a filesystem is a data member of at most one array | ADR-0008 | G mode fixtures (M2) | designed |
| SC-31 | Cockpit Storage "fix" creates a conflicting mount definition (R4) | Adopted mounts are normal fstab entries; duplicates across fstab/units/UDisks2 blocked | ADR-0007 D3, D5 | I no mismount warning after adoption (M6) | designed |
| SC-32 | Recovery started without feasibility analysis, or on stale analysis (R6) | Plan creation needs a report; `job.start` re-evaluates and needs READY | ADR-0010; plan-engine rule 6 | U + I freshness (M9) | designed |
| SC-33 | Readiness engine mutates state (R6) | Engine cannot reach mutating tools | ADR-0010 item 4; invariant I7 | S static test; I manifest proof (M4) | designed |
| SC-34 | Recovery overwrites surviving or recoverable data | Replacement must be empty and not a member of any pool/array | readiness C12 | G + I (M4, M9) | designed |
| SC-35 | Failure count exceeds parity, presented as recoverable | BLOCKED; never claim recoverability from parity count alone | readiness C5 | G F03 (M4) | designed |
| SC-36 | Rejected sync retried automatically | Never; new plan and approval required | ADR-0005 D9 | U scheduler/job-runner tests (M5) | designed |
| SC-37 | Thresholds loosened to bypass safety | Changes only via audited plan; no "off"; anomalies not adjustable | ADR-0005 D11 | U + C (M5) | designed |
