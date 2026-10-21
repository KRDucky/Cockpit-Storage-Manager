# ADR-0007: Integration with Cockpit Storage

- Status: **Accepted** (M0 closeout, 2026-10-19; R4). The UDisks2 fstab mechanism in D3 is **provisional** pending an M6 spike on each Tier-1 target.
- Date: 2026-10-19

## Research findings (Cockpit source at 369, docs at HEAD, 2026-10-19)
1. Cockpit Storage is the `storage` package (`pkg/storaged/manifest.json`): requires Cockpit ≥266
   and is shown only when `/usr/share/dbus-1/system.d/org.freedesktop.UDisks2.conf` exists.
   Distro package: `cockpit-storaged`. It manages disks, partitions, filesystems, mounts,
   LUKS, LVM, MD RAID, Stratis, btrfs, NFS client mounts and iSCSI through **UDisks2** D-Bus.
2. UDisks2 versions: Debian 13 2.10.1; Ubuntu 26.04 2.10.91; Ubuntu 26.10 2.11.2.
3. Cockpit Storage writes persistent mounts to `/etc/fstab` through UDisks2 configuration items
   (`AddConfigurationItem` in `filesystem/mounting-dialog.jsx` and related files).
4. It detects "mismounted" filesystems (`filesystem/mismounting.jsx`): a filesystem mounted
   somewhere without a matching fstab entry is flagged `mounted-no-config`, with a one-click fix
   that writes fstab.
5. Cross-package navigation is `cockpit.jump(path)`. The guide says only **documented, stable**
   component paths may be relied on (`cockpit-location.adoc`); internal routes inside the
   Storage page (its `#/…` locations) are not documented as stable.
6. Deep integration / embedding internals are documented as "in heavy flux"
   (`embedding.adoc`). Storage's React components and `client.js` are internal, not an API.

## Boundaries
| Concern | Owner | Notes |
|---|---|---|
| Partitioning, formatting, LUKS, LVM, MD, generic mount/unmount | **Cockpit Storage** | We do not duplicate these UIs. |
| Disk health overview | Cockpit Storage (generic) + this plugin (role-aware view) | We show SMART in the context of data/parity roles. |
| mergerfs pools, SnapRAID arrays, roles, recovery, schedules, pool shares | **This plugin** | |
| Persistent mount entries for adopted filesystems | Shared: written via UDisks2 so both UIs agree | D3. |

## Decisions
- **D1 Navigation.** Disks page rows and empty states offer "Open in Storage" via
  `cockpit.jump("/storage")` (package root only). We do not deep-link into Storage's internal
  routes unless a route becomes documented; any experiment with one is behind a version check
  and listed as a compatibility risk. When `cockpit-storaged` is absent the link is hidden and
  a plain-text hint is shown.
- **D2 Inventory source.** The backend keeps `lsblk --json`/`findmnt --json`/`blkid` as its
  inventory (documented CLIs, available without UDisks2, usable by the unprivileged helper).
  UDisks2 is read (`org.freedesktop.UDisks2` object tree) only to correlate the configuration
  items Cockpit Storage will show, so we can predict "mismounted" warnings. We do **not** import
  Cockpit Storage's JavaScript modules.
- **D3 Mount persistence through UDisks2 (provisional).** When adoption needs a new persistent
  mount, the plan uses UDisks2 `AddConfigurationItem` (fstab type) with `UUID=` source and
  `nofail`, so Cockpit Storage shows a correct, non-warning entry and its "fix" button is never
  needed. Same safety protocol as any shared-file write (fingerprint before and after, backup
  of `/etc/fstab`, re-read verify). If the M6 spike finds UDisks2 behavior inadequate on a
  target, the fallback is our own atomic fstab append under ADR-0006 §2.
- **D4 Initialization of blank disks is delegated by default.** The "Initialize blank disk"
  workflow (R2/M8) first offers a handoff to Cockpit Storage for partitioning and formatting,
  then returns to our adoption wizard. Our own `mkfs`/partitioning path is built only if a
  documented gap remains (e.g., typed stable-ID confirmation cannot be guaranteed in Storage),
  and stays behind the irreversible tier.
- **D5 Conflict detection.** Before adopting or mounting, check UDisks2 configuration items,
  fstab lines and systemd mount units for the same UUID or path; any duplicate ⇒ block with
  explanation.
- **D6 Compatibility.** Only public interfaces are used: `cockpit.jump`, the documented
  `storage` package path, UDisks2 D-Bus API. Each is re-verified on Cockpit 337/360/367 in M1
  (navigation) and M6 (UDisks2 fstab).

## Consequences
- Users move naturally between general disk administration and pool/parity management.
- ADR-0004's native mount units are replaced by fstab entries (ADR-0004 amended).
- Our blank-disk initialization code shrinks or disappears, removing a destructive code path.
