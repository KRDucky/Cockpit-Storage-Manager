# ADR-0006: Configuration ownership, shared files and share-service integration

- Status: **Proposed** (M0, awaiting review). Share implementation is M6; this records constraints now so earlier milestones do not paint us into a corner.
- Date: 2026-10-19

## Ownership model
Every discovered object has `ownership ∈ {managed, external, foreign_tool, unknown}`:

- `managed` — created by this plugin, recorded in `/var/lib/cockpit-pool-manager/state.json`
  **and** carrying an in-file marker (`# managed-by: cockpit-pool-manager id=<uuid>`).
  Both must agree; disagreement ⇒ `unknown`.
- `external` — hand-written (fstab line, hand-written unit, smb.conf section).
- `foreign_tool` — positively attributed to another tool (45Drives File Sharing,
  OpenMediaVault, SnapRAID Daemon, …).
- `unknown` — anything else. Read-only.

Only `managed` objects are writable. Taking over an `external` object is an explicit
"import" plan with backup, lossless round-trip proof and a diff preview.

## Shared files we may have to touch
| File | Policy |
|---|---|
| `/etc/fstab` | Never written for managed pools (ADR-0004 uses units). Read-only parser. |
| `/etc/snapraid.conf` | SnapRAID has no verified include mechanism. Managed arrays use a plugin-owned config path passed with `-c`; importing an existing `/etc/snapraid.conf` is an explicit takeover with lossless round-trip. |
| `/etc/samba/smb.conf` | See below. |
| `/etc/exports` | Never written. Managed NFS exports go to `/etc/exports.d/cockpit-pool-manager.exports` (`exportfs` reads `exports.d` per nfs-utils; verify per target in M6). |

When a shared file must be written: lock → fingerprint (sha256 + mtime + inode) → backup
→ minimal patch → validate (tool-specific) → temp file in same dir → `fsync` file →
`rename` → `fsync` dir → re-read and compare → rollback from backup on failure.

## Samba: verified facts about 45Drives File Sharing
From its source: it manages shares with `net conf` (Samba **registry**), and inserts
`include = registry` into the `[global]` section of `smb.conf` (with a "# inserted by
cockpit-file-sharing" comment). NFS exports default to
`/etc/exports.d/cockpit-file-sharing.exports`.

## Decision (proposed)
- **Detect** 45Drives via: installed package path `/usr/share/cockpit/file-sharing/`,
  `include = registry` in smb.conf, non-empty `net conf list`, the exports.d file above.
- If detected: our Shares page becomes **read-only + "Open in File Sharing"** for those
  shares, and offers to create new shares *only* through our own mechanism if the owner
  explicitly opts in. We never write the registry when 45Drives is present.
- Our own Samba mechanism (M6 decision, two candidates, to be chosen after testing on targets):
  1. dedicated `/etc/samba/cockpit-pool-manager.conf` pulled in by a single managed
     `include =` line in `[global]` (one-time, confirmed edit of a shared file), or
  2. Samba registry when no other registry user exists.
  Every change validated with `testparm -s` against the full effective config, then
  `smbcontrol all reload-config`, then verified by reading back effective shares.
- Removing a share definition **never** touches share data.
- Defaults: no guest access, no world-writable paths, path must be on a verified mounted
  pool or explicitly confirmed plain mountpoint (with warning).

## Licensing note
45Drives File Sharing is GPL-3.0 — license-compatible with this repo (GPL-3.0), but per
REFERENCES.md it is a research input only; no code is copied.
