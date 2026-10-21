# ADR-0006: Configuration ownership, shared files and share-service integration

- Status: **Accepted** (M0 closeout, 2026-10-19; covers R2, R5). Share *writing* is M7; ownership detection and read-only display are M2.
- Date: 2026-10-19

## 1. Ownership model
Every discovered object has `ownership ∈ {managed, external, foreign_tool, unknown}` plus the
**evidence** that produced it:

- `managed` — recorded in `/var/lib/cockpit-storage-manager/state.json` **and** carrying our
  in-file marker (`# managed-by: cockpit-storage-manager id=<uuid>`) where the format allows
  comments. Both must agree; disagreement ⇒ `unknown`.
- `foreign_tool` — positively attributed to another tool by **configuration provenance** (§3),
  never by package presence alone.
- `external` — administrator-written configuration with no tool provenance.
- `unknown` — anything ambiguous. **Read-only.**

Only `managed` objects are writable. External and foreign objects stay read-only until a
specific, designed and validated migration procedure exists for that object type; such a
migration is its own plan with backup, lossless round-trip proof and diff preview.

## 2. Shared files
| File | Policy |
|---|---|
| `/etc/fstab` | Existing lines are never modified or removed. New persistent mounts for adopted filesystems are added only through UDisks2's configuration interface (ADR-0007), after conflict checks, as part of a previewed plan. |
| `/etc/snapraid.conf` and other array configs | Managed arrays use a plugin-owned config passed with `-c`. Taking over an existing config is an explicit lossless import. |
| `/etc/samba/smb.conf`, Samba registry | §4. |
| `/etc/exports`, other `exports.d` files | Never written. Managed exports go to `/etc/exports.d/cockpit-storage-manager.exports` (nfs-utils 2.8.3–2.9.1 on targets; `exports.d` support verified per target in M7). |
| systemd units we did not create | Never written. |

Writing protocol for any shared file: lock → fingerprint (sha256 + mtime + inode) → backup →
minimal patch → tool validation → temp file in same directory → `fsync` → `rename` → `fsync`
directory → re-read and compare → restore from backup on failure, reporting partial state.

## 3. 45Drives Cockpit File Sharing — provenance markers (verified in v4.6.3 source)
- Samba: shares live in the Samba **registry** (`net conf`); File Sharing requires
  `include = registry` in `[global]` and inserts it with an "inserted by cockpit-file-sharing"
  comment.
- NFS: exports file path comes from its settings file **`/etc/cockpit-file-sharing.conf.json`**
  (`nfs.confPath`), default `/etc/exports.d/cockpit-file-sharing.exports`. The default must not
  be assumed — the settings file is read to find the real path.
- Package path `/usr/share/cockpit/file-sharing/` indicates installation, **not** ownership.

Attribution rules:
| Evidence | Result |
|---|---|
| Registry shares present **and** `include = registry` carries the File Sharing comment, or the settings file exists | registry shares ⇒ `foreign_tool: 45drives-file-sharing` |
| Registry shares present, no File Sharing evidence | `unknown` (registry may be used by another tool or an admin) |
| Exports in the file named by File Sharing's settings | `foreign_tool: 45drives-file-sharing` |
| Package installed but none of the above | no attribution; objects classified on their own evidence |
| `smb.conf` sections / other exports with no tool evidence | `external` |

## 4. Share management decisions
- Before share management is enabled for a protocol, the plugin runs ownership detection and
  shows the result; share writing is disabled while any object needed for the change is
  `unknown`.
- Shares owned by File Sharing are displayed as externally managed with an **"Open in File
  Sharing"** handoff (`cockpit.jump` to its documented package path, if present). We never write
  the registry while File Sharing evidence exists.
- Our own Samba mechanism (M7 decision after target testing): a dedicated
  `/etc/samba/cockpit-storage-manager.conf` included by one managed `include =` line (one-time,
  confirmed edit), or the registry when no other registry user exists.
- **Conflict detection before any change:** duplicate share names across smb.conf, registry
  and our file; NFS exports of the same path or overlapping parent/child paths across all
  `exports` sources; one path exported by both protocols with different access models; a share
  path that is a bare mountpoint directory, a single branch of a pool, or outside any verified
  mount.
- **Validation and reload:** `testparm -s` on the full effective Samba configuration; for NFS,
  our own parser validates the generated file (exportfs has no dry-run mode), then `exportfs -ra`
  after backup; read back the effective configuration (`testparm -s`, `exportfs -v`). A failed reload restores the backup, reloads again, and reports the
  exact resulting state (`partial` if the restore also fails).
- Removing a share definition **never** touches share data.
- Defaults: no guest access, no world-writable paths, no wildcard NFS clients; each requires
  separate confirmation.

## Licensing note
45Drives File Sharing is GPL-3.0 — compatible with this repository — but per REFERENCES.md it is
a research input only; no code is copied.
