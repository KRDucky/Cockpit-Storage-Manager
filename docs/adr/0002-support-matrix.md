# ADR-0002: Supported platforms and component versions

- Status: **Proposed** (M0, awaiting review)
- Date: 2026-10-19
- Evidence gathered: 2026-10-19 (see "Sources"). Re-verify at the start of every milestone.

## Observed distro package versions

| Component | Debian 13 "trixie" (stable) | Ubuntu 26.04 LTS "resolute" | Ubuntu 26.10 "stonking" | Upstream latest |
|---|---|---|---|---|
| cockpit | 337-1+deb13u1 (trixie-backports: 368-1~bpo13+1) | 360-1 | 367-1 | 369 (tag) |
| mergerfs | 2.40.2-5 | 2.40.2-5build1 (universe) | 2.40.2-5build1 (universe) | 2.42.0 (2026-05-07) |
| snapraid | 12.2-1 | 13.0-1 (universe) | 14.9-1 (universe) | 14.10 released (2026-08); 15.0 tagged 2026-10-15 but HISTORY still says "Work in Progress" |
| python3 | 3.13 | verify in M1 | verify in M1 | — |

Not verified (marked uncertain, do **not** rely on): exact Python, systemd, Samba and
nfs-utils versions on Ubuntu 26.04/26.10; whether Ubuntu `-backports` carries newer
cockpit/snapraid for 26.04. The features this project needs from systemd
(`RequiresMountsFor=`, `BindsTo=`, `ConditionPathIsMountPoint=`, `OnFailure=`, timers with
`Persistent=` and `RandomizedDelaySec=`, `systemd-run` transient units) are all long
established and are not expected to vary across these targets, but M1 CI must assert them.

## Version-gated behavior that matters for safety

| Feature | Available from | Present on targets | Consequence |
|---|---|---|---|
| SnapRAID structured log tags (`--log ">&2"`, `snapraid_log(1)`) | 14.0 | 26.10 only | Need **two** SnapRAID output adapters: tag-based (≥14.0) and text-scrape (12.x/13.x). Text adapter is lower confidence; UI must say so. |
| SnapRAID include/exclude directory fix | 14.1 (14.0 regression) | — | Refuse managed sync on exactly 14.0 when include/exclude rules exist. |
| SnapRAID `relocated` file state in diff | 14.0 | 26.10 only | Diff parser must handle both vocabularies. |
| SnapRAID `.snapraidignore` files | 14.0 | 26.10 only | Exclusions may live outside `snapraid.conf`; config view must say "plus any `.snapraidignore` files" on ≥14.0. |
| SnapRAID `--gui-threshold-removes` abort | present in source, **undocumented** | unknown per version | May be used only as defense-in-depth, feature-detected, never as the primary guard. |
| SnapRAID `snapshot` option | 15.0 (unreleased) | none | Out of scope; detect and display only. |
| mergerfs `branches-mount-timeout` | absent in 2.33.5, present in 2.35.1 (source grep by tag) | all | Usable, but without `-fail` it only delays; it does not prevent a degraded mount. |
| mergerfs `branches-mount-timeout-fail` | 2.41.0 | **none** | Cannot be relied on. Mount-ordering safety must come from systemd units we generate (ADR-0004). |
| Cockpit manifest `conditions` `path-exists` | 286 | all | Usable but we will **not** hide the page when tools are missing (spec wants a dependency-aware overview). |
| Cockpit manifest `conditions` `any` | 348 | not Debian 13 (337) | Do not use. |

## Decision (proposed)

**Tier 1 (CI-tested, release-blocking):**

- Debian 13 with distro cockpit 337 and distro mergerfs 2.40.2, snapraid 12.2.
- Ubuntu 26.04 LTS with distro cockpit 360, mergerfs 2.40.2, snapraid 13.0.

**Tier 2 (CI-tested, non-blocking):**

- Ubuntu 26.10 (cockpit 367, snapraid 14.9) — exercises the structured-log adapter.
- Debian 13 + trixie-backports cockpit 368.
- Upstream mergerfs `.deb` 2.42.x (trapexit publishes per-distro `.deb`s) and upstream snapraid 14.10.

**Explicitly unsupported in v1:** Ubuntu 24.04 and earlier, Debian 12, RHEL/Fedora/Arch
(packaging later), SnapRAID < 12.0, mergerfs < 2.40, any 15.x pre-release. Hosts outside
the matrix get **read-only inspection only**, with a banner, never mutations.

**Minimum Cockpit:** manifest `"requires": {"cockpit": "337"}`. Rationale: Debian 13 floor.
The starter-kit pins `pkg/lib` from Cockpit 369 and uses PatternFly 6; whether that
`pkg/lib` behaves correctly on a 337 shell must be **verified in M1** (open question Q1).

**Dependency handling:** mergerfs and snapraid are `Recommends:`, not `Depends:`, so the
plugin can install on a host that only wants inspection or shares. Missing tools surface
as explicit "not installed" states, never as empty data.

## Sources

- mergerfs, snapraid, cockpit, starter-kit, cockpit-file-sharing: shallow clones of upstream
  `master`/`main` and release tags on 2026-10-19 (HISTORY, `mkdocs/docs/config/*.md`,
  `doc/snapraid.txt`, `doc/snapraid_log.txt`, `doc/modules/guide/pages/packages.adoc`).
- Debian: tracker.debian.org/pkg/src:cockpit; piuparts trixie logs for mergerfs 2.40.2-5;
  packages.debian.org/source/trixie/snapraid (12.2-1).
- Ubuntu: packages.ubuntu.com/cockpit, /source/resolute/mergerfs, /resolute/snapraid,
  /stonking/snapraid; ubuntuupdates.org/mergerfs.
