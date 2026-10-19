# Decision log

Index of architecture decisions. Full text in `docs/adr/`. Status values: Proposed → Accepted / Rejected / Superseded.

| ADR | Title | Status | Date | Summary |
|---|---|---|---|---|
| 0001 | Backend helper language | Proposed | 2026-10-19 | Python ≥3.11, stdlib-only runtime; JSON Schema contracts; single subprocess site; mypy --strict. Rust kept as an option for future components. |
| 0002 | Support matrix | Proposed | 2026-10-19 | Tier 1: Debian 13, Ubuntu 26.04. Tier 2: Ubuntu 26.10, trixie-backports, upstream mergerfs/snapraid. Outside matrix ⇒ read-only. Min Cockpit 337. |
| 0003 | Privilege & job execution | Proposed (**deviation, Q3**) | 2026-10-19 | One-shot helper via `cockpit.spawn`; viewer/admin/irreversible tiers; long jobs as `systemd-run` transient units; per-action polkit deferred because Cockpit admin mode is already root. |
| 0004 | Mount ordering & pool units | Proposed (Q4 open) | 2026-10-19 | Managed branches as UUID mount units; pool as `mergerfs -f` service with identity-verifying `ExecStartPre`; markers prevent bare-dir mounts; no reliance on mergerfs ≥2.41. |
| 0005 | SnapRAID integration & sync guard | Proposed (Q6 open) | 2026-10-19 | Tag adapter (≥14.0) + text adapter; lossless config model; own diff-threshold guard with TOCTOU re-check; forced flags never automated; coexist with SnapRAID Daemon. |
| 0006 | Config ownership & shares | Proposed | 2026-10-19 | managed/external/foreign_tool/unknown; never write fstab or /etc/exports; 45Drives shares read-only; atomic fingerprinted writes. |
