# ADR-0002: Supported platforms and component versions

- Status: **Accepted** (M0 closeout, 2026-10-19). Version table must be re-verified at the start of every milestone.
- Date: 2026-10-19
- Supersedes: the M0-discovery draft of this ADR, which wrongly stated that SnapRAID structured log tags exist only from 14.0 (see "Corrections").

## Observed distro package versions (verified 2026-10-19)

| Component | Debian 13 "trixie" | Ubuntu 26.04 LTS "resolute" | Ubuntu 26.10 "stonking" | Upstream |
|---|---|---|---|---|
| cockpit | 337-1+deb13u1 (backports 368-1~bpo13+1) | 360-1 | 367-1 | 369 |
| mergerfs | 2.40.2-5 | 2.40.2-5build1 (universe) | 2.40.2-5build1 (universe) | 2.42.0 (2026-05) |
| snapraid | 12.2-1 | 13.0-1 (universe) | 14.9-1 (universe) | 14.10 (2026-08); 15.0 tagged but HISTORY says "Work in Progress" |
| python3 | 3.13.5 | 3.14.3 | 3.14.3 | — |
| systemd | 257.13 | 259.5 | 261.3 | — |
| samba | 4.22.10 (backports 4.24.7) | 4.23.6 | 4.24.7 | — |
| nfs-utils (`nfs-kernel-server`) | 2.8.3 | 2.8.5 | 2.9.1 | — |
| udisks2 | 2.10.1 | 2.10.91 | 2.11.2 | — |

Still unverified (tracked, not relied on): `e2fsprogs`/`xfsprogs` versions and
`chattr +i` behavior per root-filesystem type (needed by ADR-0004; verified by M1/M6
integration tests, not by package lookup); Ubuntu 26.10 ships Rust coreutils (reported by
release coverage) — we use util-linux (`lsblk`, `findmnt`, `blkid`) and e2fsprogs
(`chattr`, `lsattr`), not coreutils output, but M1 CI must assert this.

## Version-gated behavior that matters for safety

| Feature | Evidence | Present on targets | Consequence |
|---|---|---|---|
| SnapRAID `--log ">&1"/">&2"` and `log_tag` scan tags (`scan:add/remove/update/move/copy/restore`) | present in v12.2 source (`cmdline/snapraid.c`, `cmdline/scan.c`); formally documented in `snapraid_log(1)` from 14.0 | **all** | One tag-based adapter for all supported versions, with per-version captured fixtures. Text scraping is not used for decisions. |
| `scan:relocate` tag / `relocated` diff state | 14.0 | 26.10 only | Parser handles both vocabularies. |
| `scan:copy` argument order | source in **all** versions emits `source_disk:source_path:dest_disk:dest_path`; the 14.x manpage documents the reverse | all | Parser follows source and fixtures, not the manpage; fixture test locks the order. |
| `.snapraidignore` files, `**` globs, `extra` option | 14.0 | 26.10 only | Config view says "plus any `.snapraidignore` files" on ≥14.0. |
| include/exclude directory regression | 14.0 only, fixed 14.1 | none | Refuse managed sync on exactly 14.0 with include/exclude rules. |
| `--gui-threshold-removes` abort | present in source, undocumented | unknown per version | Defense in depth only, feature-detected, never relied on. |
| SnapRAID `snapshot` option | 15.0 pre-release | none | Inspection-only. |
| mergerfs `branches-mount-timeout` | absent 2.33.5, present 2.35.1 | all | Startup delay only; in 2.40.2 it then mounts anyway. |
| mergerfs `branches-mount-timeout-fail`, `.mergerfs.branch*` markers | 2.41.0 (`src/fs_wait_for_mount.cpp`) | **none** | Markers are read only during startup wait; never at runtime. Not relied on (ADR-0004). |
| mergerfs runtime branch modification (`user.mergerfs.branches`, `=NC/=RO`) | documented in 2.40.2 README | all | Used by the runtime branch guard (ADR-0004). |
| Cockpit manifest `conditions` `path-exists` / `any` | 286 / 348 | `any` not on Debian 13 | Do not use `any`; do not hide the page when tools are missing. |
| Cockpit Storage (`storage` package) | requires cockpit ≥266 and UDisks2 D-Bus config present (`pkg/storaged/manifest.json`) | all (if `cockpit-storaged` installed) | Integration per ADR-0007; optional dependency. |

## Decision

**Tier 1 (CI-tested, release-blocking):** Debian 13 (distro cockpit 337, mergerfs 2.40.2,
snapraid 12.2) and Ubuntu 26.04 LTS (cockpit 360, mergerfs 2.40.2, snapraid 13.0).

**Tier 2 (CI-tested, non-blocking):** Ubuntu 26.10; Debian 13 + trixie-backports cockpit;
upstream mergerfs 2.42.x `.deb`; upstream snapraid 14.10.

**Unsupported in v1:** Ubuntu ≤ 24.04, Debian ≤ 12, non-Debian families, SnapRAID < 12.0,
mergerfs < 2.40, any 15.x pre-release. Hosts outside the matrix get **read-only inspection only**.

**Minimum Cockpit:** `"requires": {"cockpit": "337"}`. Whether `pkg/lib` pinned from Cockpit
369 works on a 337 shell is a **mandatory M1 test** (Q1), not an assumption.

**Dependencies:** `mergerfs`, `snapraid`, `smartmontools`, `cockpit-storaged` are
`Recommends:`. Missing tools are explicit "not installed" states, never empty data.

## Corrections to the M0-discovery draft
- "Structured tags only from 14.0" — wrong. Tags and `--log ">&2"` exist in 12.2; 14.0 documented and expanded them. The low-confidence text adapter is dropped.
- "Markers make mergerfs treat a bare directory as not mounted" — true only for ≥2.41 and only during the startup wait.

## Sources
Upstream clones of mergerfs, snapraid (tags v12.2, v13.0, v14.9, HEAD), cockpit, starter-kit,
cockpit-file-sharing on 2026-10-19; packages.ubuntu.com (cockpit, python3, systemd, samba,
nfs-kernel-server, udisks2, mergerfs, snapraid); tracker.debian.org (cockpit, systemd, samba,
nfs-utils, udisks2, python3-defaults); piuparts trixie mergerfs 2.40.2-5;
packages.debian.org/source/trixie/snapraid.
