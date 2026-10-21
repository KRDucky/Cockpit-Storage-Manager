# Requirements coverage and traceability (R1–R6)

Status: M0 closeout, 2026-10-19. Source: owner document "M0 Additional Owner Requirements —
Storage Management and Recovery". Audit baseline: branch `m0/discovery` @ `45d1de0`.

## 1. Coverage audit (before → after closeout)
| Req | Before closeout | Gaps and contradictions found | After closeout |
|---|---|---|---|
| R1 Independent mergerfs / SnapRAID | **Partial** — separate Pools/Parity pages and data types existed | No explicit deployment modes; wireframes assumed one array ("Array main"); API keyed arrays by `config_path` only; no multi-array locking statement | **Covered** — ADR-0008; `array_id`; derived links; wireframes list multiple pools/arrays; mode-aware workflows |
| R2 Non-destructive adoption | **Partial** — "Adopt existing filesystem" in spec; inspect-only support levels | **Contradiction:** ADR-0004 draft wrote a `.csm-branch.json` marker into the user's filesystem root; ext4/XFS not named; conflict detection limited to fstab/units on the same path; no rule against overwriting existing mount definitions | **Covered** — ADR-0004 §1 (no writes to data filesystems; existing mounts kept), ADR-0007 D3/D5, wizard in wireframes, SC-29…SC-31 |
| R3 Guided replacement and recovery | **Partial** — healthy/failed/parity/content workflows named in spec | Missing: mergerfs-only replacement, multiple simultaneous failures, planned retirement as a distinct type, auditable recovery journal requirement, mandatory external-scheduler resolution | **Covered** — scenarios S1–S7, ADR-0010, M8/M9 scope, ADR-0005 D12 |
| R4 Cockpit Storage integration | **Missing** | **Contradiction:** native `.mount` units for branches would trigger Cockpit Storage "mismounted" warnings and its fstab "fix" | **Covered** — ADR-0007 (boundaries, `cockpit.jump`, UDisks2 fstab, init handoff); ADR-0004 amended |
| R5 45Drives coexistence | **Partial** — registry/`include = registry` facts, read-only display | **Contradiction:** detection listed the installed package path as an ownership signal; settings file that relocates the NFS exports path was not considered; overlapping exports / duplicate names / reload-failure handling incomplete | **Covered** — ADR-0006 §1, §3 (evidence table), §4 |
| R6 Recovery simulation / readiness | **Missing** — only generic preflight | — | **Covered** — `recovery-simulation.md`, ADR-0010, M4 placed before all destructive milestones, fixtures F01–F10 |

## 2. Traceability
| Req | Design artifacts | Milestone(s) | Automated tests (ID / suite) |
|---|---|---|---|
| R1 | ADR-0008; components-and-api §3 (`Pool`, `SnapraidArray`, links); plan-engine §5 (per-object locks); ui-wireframes (Pools, Parity) | M2 (discover/display), M6 (manage) | G fixture hosts: pooling-only, parity-only, combined, 2 pools + 2 arrays; U link derivation by fs UUID; U "data member of two arrays" finding; C pages render without the other subsystem; U lock independence between arrays |
| R2 | ADR-0004 §1, ADR-0006 §1–2, ADR-0007 D3/D5; ui-wireframes "Adopt" wizard | M2 (discovery, conflicts), M6 (adoption) | I adopt ext4 and XFS with data → reboot → manifest identical incl. owner/mode; I existing fstab line preserved byte-for-byte; I conflict: same UUID in fstab + unit ⇒ blocked; I duplicate UUID ⇒ inspect-only; S no write to data fs root (manifest); G inspect-only when ownership unknown |
| R3 | recovery-simulation §2, §8; ADR-0005 D12; ADR-0010; plan-engine §3 recovery journal | M8 (S2, S6), M9 (S1, S3, S4, S5, S7) | I/F each scenario on loop devices; S static test: no recovery plan contains `sync`; I external scheduler present ⇒ blocked; I interrupted `fix` ⇒ `partial` + journal; C wording never "rebuild"/"resilver" |
| R4 | ADR-0007; ui-wireframes Disks ("Open in Storage") | M1 (navigation), M6 (UDisks2 fstab), M8 (init handoff) | I `cockpit.jump("/storage")` on 337/360/367; I adopted mount shows no Cockpit Storage mismount warning; U conflict detection across UDisks2 config items/fstab/units; R link hidden without `cockpit-storaged` |
| R5 | ADR-0006 §3–4; ui-wireframes Shares | M2 (detection, display), M7 (management) | G 45Drives fixture with default and relocated exports path; G registry without File Sharing evidence ⇒ unknown; G package installed but no config ⇒ no attribution; I duplicate share name / overlapping exports ⇒ blocked; F reload failure ⇒ restore + exact state |
| R6 | recovery-simulation (whole); ADR-0010; components-and-api §4 (`readiness.*`) | **M4** (foundation, before M8/M9) | G F01–F10 (+F04b, F06b, F09b); U verdict function; S forbidden-command static test; I evaluation changes nothing (manifest); I freshness: change between evaluate and `job.start` ⇒ refused (M9) |

## 3. Owner closeout items → where addressed
| Item | Addressed in |
|---|---|
| ADR statuses + decisions recorded | `docs/DECISIONS.md`, ADR headers |
| Ubuntu/Debian Python/systemd/Samba/NFS gaps | ADR-0002 table (verified 2026-10-19); remaining: e2fsprogs/`chattr` behavior → RBL-14 |
| Cockpit 337 vs 369 compatibility | M1 acceptance item 2 (mandatory) |
| Disposable VM / loopback testing approach | `docs/design/test-environment.md` |
| Stable identifiers in plan engine | plan-engine §1 rule 5, §4 |
| Reversible operations and readable reconciliation | plan-engine §6 |
| Replacement/recovery never auto-syncs | ADR-0005 D9/D12, ADR-0010, SC-09 |
| External configs and third-party shares read-only | ADR-0006 §1, §4 |
