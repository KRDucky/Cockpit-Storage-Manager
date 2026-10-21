# Project status

_Last updated: 2026-10-19 (M0 closeout)_

## Current milestone
**M0 — Discovery and architecture: closeout complete, awaiting owner acceptance.**
Documentation only. No code exists, no packages were installed, no host storage was touched.
**M1 is not authorized.**

Branches: `m0/discovery` (original proposal) → `m0/closeout` (this closeout; PR to `main`, not merged).

## Proposed final M0 acceptance criteria
M0 is accepted when the owner confirms all of:
1. ADR-0001…0010 exist with owner decisions Q2–Q7 recorded and statuses set (`docs/DECISIONS.md`).
2. Repository contains no legacy product name or `cpm-` prefix except explicitly historical notes.
3. Requirements R1–R6 each have coverage status, design artifacts, milestone and automated
   tests (`docs/design/requirements-traceability.md`).
4. Runtime branch loss design answers all six Q4 conditions, with tests RBL-01…14 and a
   fail-closed fallback (`docs/design/runtime-branch-loss.md`).
5. Sync guard specifies anomaly suspension, thresholds, exact diff fingerprint fields and their
   limits, revalidation, no-retry, concurrency gap and quiesce option, audited threshold changes
   (ADR-0005).
6. Recovery simulation design defines checks, verdict rules, typed report contract, forbidden
   commands and fixtures F01–F10 (`docs/design/recovery-simulation.md`), scheduled as M4 before
   any destructive milestone.
7. Disposable VM / loopback test approach with an enforced interlock (`docs/design/test-environment.md`).
8. Plan engine states stable-identifier, readiness-gate, recovery-journal, reversibility and
   reconciliation rules (`docs/design/plan-engine.md`).
9. Version gaps closed or explicitly tracked (ADR-0002); Cockpit 337-vs-369 is a mandatory M1 test.
10. Consistency checks pass: all Mermaid diagrams parse, all referenced docs exist, no
    contradictions found in the final review (see PR description for the check log).

## Unresolved risks (carried into M1+)
| ID | Risk | Mitigation / owner of resolution |
|---|---|---|
| U1 | Cockpit `pkg/lib` from 369 may misbehave on 337 (Debian 13) | Mandatory M1 test; fallback: pin an older `pkg/lib` or raise the minimum |
| U2 | L1 immutable underlay may be unsupported or behave unexpectedly on some root filesystems | RBL-14 per target; fail-closed default where unverified |
| U3 | Detection of wrong-device remount depends on a timer; exposure window exists | RBL-09 measures it; interval configurable; documented residual risk |
| U4 | systemd behavior on device removal and for externally performed mounts unverified | RBL-07/09; timer backstop independent of it |
| U5 | Sync TOCTOU between final diff and sync's own scan | D8 revalidation, undocumented `--gui-threshold-removes` when available, optional quiesce mode |
| U6 | UDisks2 fstab configuration items may differ across 2.10.1 / 2.10.91 / 2.11.2 | M6 spike; fallback to our own atomic fstab append |
| U7 | SnapRAID tag formats may change between versions (e.g., manpage vs source order for `scan:copy`) | per-version captured fixtures; unknown tags surfaced, not ignored |
| U8 | No CI nested virtualization confirmed; T3/T4 may need an owner-provided disposable VM host | decide in M1 |
| U9 | Clients and applications may treat invisible files on a lost branch as deletions | critical banner guidance; outside the plugin's control |
| U10 | File Sharing could change its provenance markers in future versions | uncertain ownership ⇒ read-only; fixtures per File Sharing version |

## Proposed M1 skeleton (not created; requires M1 authorization)
```
package.json, build.js, tsconfig.json, Makefile        # cockpit starter-kit pattern
src/ui/{index.html,index.tsx,app.tsx,pages/*}          # PatternFly 6, React 18
src/api/{client.ts,transport-cockpit.ts,transport-mock.ts,types.gen.ts}
api/schema/*.json                                      # normative contracts
backend/csm_helper/{__main__.py,rpc.py,exec.py,errors.py,snapshot.py}
backend/csm_helper/adapters/{lsblk.py,findmnt.py,mountinfo.py,udev.py,smart.py}
backend/tests/{unit,fixtures}/
mock/hosts/{healthy,degraded-missing-disk,partial-config,dup-uuid,pool-only,parity-only,multi}.json
packaging/debian/                                      # arch: all; name cockpit-storage-manager
integration/{README.md,interlock/}                     # test-host interlock (T0-tested in M1)
.github/workflows/ci.yml                               # T0–T2
```

## Environment notes
M0 workspace: Ubuntu 24.04 container, Python 3.13, Node 22; no KVM, Cockpit, mergerfs,
SnapRAID or Samba installed. Research used shallow clones of upstream sources and distro
package pages (listed in ADR-0002).
