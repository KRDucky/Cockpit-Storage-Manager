# Project status

_Last updated: 2026-10-19 (M0)_

## Current milestone
**M0 — Discovery and architecture sign-off: complete, awaiting review.**
No code exists. No host has been modified. Next milestone (M1) is **not authorized**.

## Repository state at M0 start
- `main` @ `53df43c`: `LICENSE` (GPL-3.0), `CLAUDE.md` only.
- `CLAUDE.md` referenced `docs/PRODUCT_SPEC.md`, `docs/ARCHITECTURE.md`,
  `docs/SAFETY_AND_TESTING.md`, `docs/REFERENCES.md`, which were **missing** from the repo.
  Copied verbatim from the claude.ai Project on branch `m0/discovery`.

## M0 deliverables (branch `m0/discovery`)
| # | Deliverable | Location |
|---|---|---|
| 1 | Backend language ADR | `docs/adr/0001-backend-language.md` |
| — | Support matrix + version evidence | `docs/adr/0002-support-matrix.md` |
| — | Privilege & job execution | `docs/adr/0003-privilege-and-execution-model.md` |
| — | Mount ordering / pool units | `docs/adr/0004-mount-ordering-and-pool-units.md` |
| — | SnapRAID integration & sync guard | `docs/adr/0005-snapraid-integration.md` |
| — | Config ownership & shares | `docs/adr/0006-config-ownership-and-shares.md` |
| 2 | Component diagram + typed API outline | `docs/design/components-and-api.md` |
| 3 | Plan/job state machines + lock model | `docs/design/plan-engine.md` |
| 4 | Privilege/threat model | `docs/design/threat-model.md` |
| 5 | Per-milestone acceptance criteria & tests | `docs/design/milestones.md` |
| 6 | UI wireframes (text) | `docs/design/ui-wireframes.md` |
| — | Critique of unsafe assumptions | `docs/design/m0-critique.md` |
| 7 | Skeleton | **Not created** — proposed below, pending approval |

## Open questions for the owner
- **Q1** Does a `pkg/lib` pinned from Cockpit 369 (starter-kit) work on Debian 13's Cockpit 337? (Answer in M1 by testing.)
- **Q2** Product/package name: `cockpit-pool-manager` (project instructions) vs `cockpit-storage-manager` (repo). Proposal: package `cockpit-pool-manager`, prefix `cpm-`.
- **Q3** ADR-0003 deviation: accept tiering option A (admin mode = root, server-side safety gates, delegated polkit mode deferred)?
- **Q4** ADR-0004: on runtime branch loss, unmount pool (`BindsTo=`) or keep it up degraded with all jobs blocked? Proposal: keep up, block jobs.
- **Q5** Source of truth for spec docs: repo `docs/` (proposal) or the claude.ai Project?
- **Q6** Default sync-guard thresholds (removed > 50 files or > 1% per disk; updated > 500).
- **Q7** Python ≥3.11 stdlib-only backend (ADR-0001) — accept?

## Proposed M1 skeleton (not created)
```
package.json, build.js, tsconfig.json, Makefile        # from cockpit starter-kit pattern
src/ui/{index.html,index.tsx,app.tsx,pages/*}          # PF6, React 18
src/api/{client.ts,transport-cockpit.ts,transport-mock.ts,types.gen.ts}
api/schema/*.json                                      # normative contracts
backend/cpm_helper/{__main__.py,rpc.py,exec.py,errors.py,snapshot.py}
backend/cpm_helper/adapters/{lsblk.py,findmnt.py,mountinfo.py,udev.py,smart.py}
backend/tests/{unit,fixtures}/
mock/hosts/{healthy,degraded-missing-disk,partial-config,dup-uuid}.json
packaging/debian/                                       # arch: all
integration/README.md                                   # VM/loop harness (M2+)
.github/workflows/ci.yml                                # lint, mypy, unit, build, container matrix
```

## Environment notes
- M0 research workspace: Ubuntu 24.04 container, Python 3.13, Node 22, cargo present; no Cockpit, mergerfs, SnapRAID or Samba installed. No VM or loop-device capability verified yet — required for M2 integration tests.
