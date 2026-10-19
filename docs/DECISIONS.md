# Decision log

Authoritative index of decisions (ADR-0009). Full text in `docs/adr/`.
Status values: Proposed → Accepted / Accepted with conditions / Rejected / Superseded.

## Architecture decision records
| ADR | Title | Status | Date | Summary |
|---|---|---|---|---|
| 0001 | Backend helper language | **Accepted** (Q7) | 2026-10-19 | Python ≥3.11 stdlib-only; mypy --strict; Ruff; schema validation; one subprocess module; strict allowlist; no shell; typed transitions. Rust only for isolated components if measurements justify it. |
| 0002 | Support matrix | **Accepted** | 2026-10-19 | Tier 1 Debian 13, Ubuntu 26.04; Tier 2 Ubuntu 26.10, backports, upstream mergerfs/snapraid. Versions verified incl. Python/systemd/Samba/NFS/UDisks2. Min Cockpit 337; 337-vs-369 `pkg/lib` is a mandatory M1 test. |
| 0003 | Privilege & job execution | **Accepted — Option A** (Q3) | 2026-10-19 | Viewer / admin / typed-confirmation tiers; typed confirmation is a safety control, not a security boundary; no privileged persistent D-Bus service; delegated operators deferred to a separate proposal. Long jobs via `systemd-run`. |
| 0004 | Mount ordering, runtime branch loss, pool units | **Accepted with conditions** (Q4) | 2026-10-19 | Startup identity gate; immutable underlay dirs (L1); runtime `=NC` guard + SnapRAID suspension (L2); degraded mode only where L1 verified and RBL tests pass, else fail-closed. Markers not relied on. |
| 0005 | SnapRAID integration & sync guard | **Accepted** (Q6, provisional thresholds) | 2026-10-19 | Tag adapter for 12.2+; anomaly suspension independent of counts; thresholds 50 / 1% per disk / 500; full diff fingerprint; revalidate before sync; never auto-retry; optional quiesce mode; audited threshold changes; recovery never syncs. |
| 0006 | Config ownership & shares | **Accepted** | 2026-10-19 | Provenance-based ownership; foreign/unknown read-only; File Sharing evidence incl. `/etc/cockpit-file-sharing.conf.json`; conflict and reload-failure handling. |
| 0007 | Cockpit Storage integration | **Accepted** (D3 provisional) | 2026-10-19 | Boundaries; `cockpit.jump("/storage")`; fstab entries via UDisks2; blank-disk init handed off to Cockpit Storage; no internal APIs. |
| 0008 | Independent pools and arrays | **Accepted** | 2026-10-19 | Three deployment modes; many pools and arrays per host; derived links; one array per data filesystem. |
| 0009 | Documentation source of truth | **Accepted** (Q5) | 2026-10-19 | Repository `docs/` is authoritative; Project Knowledge is reference only. |
| 0010 | Recovery readiness gate | **Accepted** | 2026-10-19 | Read-only feasibility engine; READY / BLOCKED / INDETERMINATE; mandatory and re-evaluated before recovery; ships in M4 before destructive operations. |

## Owner decisions recorded 2026-10-19
| Q | Decision | Where applied |
|---|---|---|
| Q2 | Name `cockpit-storage-manager`, prefix `csm-`; no migration needed | all docs; PRODUCT_SPEC A0 |
| Q3 | ADR-0003 Option A | ADR-0003; ARCHITECTURE amendments; threat model T22 |
| Q4 | Degraded pool accepted conditionally; fail-closed otherwise | ADR-0004; `docs/design/runtime-branch-loss.md` |
| Q5 | Repository `docs/` authoritative | ADR-0009; CLAUDE.md |
| Q6 | Thresholds 50 / 1% / 500 as provisional defaults + nine additional requirements | ADR-0005 D4–D11 |
| Q7 | ADR-0001 accepted | ADR-0001 |

## Requirement changes
| Date | Change | Record |
|---|---|---|
| 2026-10-19 | R1–R6 added (owner "M0 Additional Owner Requirements") | PRODUCT_SPEC A1–A6; `requirements-traceability.md` |
| 2026-10-19 | Milestones re-sequenced: readiness simulator is M4, before all destructive work; old M4–M9 shift by one | `docs/design/milestones.md` |
