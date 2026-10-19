# Privilege and threat model

Status: M0 proposal. Method: assets → trust boundaries → STRIDE-style threats → controls → residual risk.
Re-review at M3 (first mutation) and before any release.

## 1. Assets (in priority order)
1. **User data** on data disks (irreplaceable).
2. **Recoverability**: parity files and content files; the relationship between them and the data at last sync.
3. Host integrity (root filesystem, boot, other services).
4. Confidentiality of share contents and of identifying info (serials, hostnames, usernames) in diagnostics.
5. Availability of the pool and shares.

## 2. Actors
| Actor | Trust | Notes |
|---|---|---|
| Cockpit admin (admin mode on) | Trusted to be root | Can already run anything; we protect them from **mistakes**, not malice. |
| Cockpit non-admin user | Untrusted for mutations | Viewer tier only. |
| Network attacker | Untrusted | Reaches Cockpit login (TLS) and SMB/NFS. |
| Malicious/compromised page JS in admin session | = root | Cannot be contained by us (ADR-0003). |
| Hostile **data**: filenames, labels, serials, fstab/snapraid.conf contents, tool output | Untrusted input | Main injection surface. |
| Other admins / other tools editing config concurrently | Honest but uncoordinated | Race and ownership surface. |
| Hardware/kernel | Unreliable | Disks vanish, enumerate differently, return errors. |

## 3. Trust boundaries
- B1 Browser ↔ cockpit-ws/bridge (Cockpit's auth; not ours).
- B2 Bridge ↔ `cpm-helper` (JSON over stdin/stdout; schema-validated).
- B3 `cpm-helper` ↔ host tools (argv arrays; output parsed as untrusted).
- B4 `cpm-helper` ↔ filesystem (paths from config and discovery are untrusted).
- B5 Share services ↔ network clients (Samba/NFS's own security model; we configure it).

## 4. Threats and controls

| # | Threat | Boundary | Control | Residual |
|---|---|---|---|---|
| T1 | Non-admin user triggers mutation | B2 | Helper refuses mutation unless `euid==0`; mutating methods only on `superuser:"require"` | None known |
| T2 | Command injection via label/path/share name | B3 | No shell anywhere; argv arrays; CI bans `shell=True`/`os.system`; per-field validators (e.g., share names `^[A-Za-z0-9._-]{1,64}$`); `--` before path args where tools support it | Tool-specific argument parsing quirks (e.g., values starting with `-`): reject leading `-` in all path/name fields |
| T3 | Config injection: newline in a value written into smb.conf / exports / units / snapraid.conf | B4 | Reject `\n \r \0` and section/comment metacharacters in every written value; render through per-format encoders; re-parse after render and compare to intended model | — |
| T4 | Colon/comma/`=` in mergerfs branch path breaks branch list | B3 | Reject `:` `,` `=` in managed branch paths (mergerfs branch syntax); imported pools with such paths are inspect-only | — |
| T5 | Symlink/TOCTOU: path swapped between check and use (e.g., marker file, backup path, temp file) | B4 | `openat`-style traversal with `O_NOFOLLOW`; temp files via `O_EXCL` in target dir; state dirs root-owned 0700; verify `st_dev/st_ino` after open | Window remains for paths under user-writable share trees; we never write there except marker files at branch roots (root-owned) |
| T6 | Wrong disk targeted (sdX reorder, duplicate UUID, cloned disks) | B4 | Targets by stable_id + expected serial/size/fs UUID; duplicate UUID ⇒ `E_AMBIGUOUS_IDENTITY`; re-check before irreversible phase | Two disks with identical serial and size (fake/cloned USB bridges) ⇒ treated as ambiguous, inspect-only |
| T7 | Writes land on root fs under missing branch mount | B4 | ADR-0004 verifier + `.mergerfs.branch_mounts_here`; shares `Requires=` pool unit | Imported/external pools: we can warn, not prevent |
| T8 | Mass deletion propagated into parity | — | ADR-0005 guard, thresholds, approved-diff hash, suspend timers | Deletions below threshold are synced by design |
| T9 | Stale plan executed after hardware change | B2 | Snapshot fingerprint + TTL + per-phase revalidation | — |
| T10 | Concurrent jobs / other admin edits | B4 | Lock model; fingerprinted writes; abort on drift | Other tools not using our locks (mitigated by fingerprints only) |
| T11 | Tool output DoS (huge output, hang) | B3 | Output cap, timeouts, kill process group; parser bounds | — |
| T12 | Malicious tool in PATH / env manipulation | B3 | Absolute allowlisted paths with ownership/mode checks; scrubbed env (`LD_*`, `PYTHON*`, `IFS` removed) | Root-level compromise of `/usr/bin` is out of scope |
| T13 | Helper replaced or state tampered | B4 | Package-owned, root 0755; state dir root 0700; plans/jobs files 0600 | Root compromise out of scope |
| T14 | Diagnostic bundle leaks serials/hostnames/usernames/share paths | — | Redaction on by default with explicit per-category opt-out; preview before download | User may opt out |
| T15 | Over-permissive share (guest, world-writable, wide NFS client range) | B5 | Defaults deny; guest and `*`/`0.0.0.0/0` client ranges require separate confirmation; never change existing permissions silently | User may opt in |
| T16 | Unsafe cancel corrupts state (kill mid-fix, mid-mkfs, mid-copy) | — | Per-phase `cancel` policy; only `safe` phases are cancellable in UI and backend | Power loss (tested via fault injection) |
| T17 | Misleading UI ("Protected", "Rollback succeeded") | — | Wording rules: parity freshness shown with timestamp; partial state is first-class | Human misreading |
| T18 | Supply chain (npm deps in UI bundle) | — | Lockfile, minimal deps, SBOM, pinned Cockpit `pkg/lib` commit; Python runtime stdlib-only | Upstream compromise |
| T19 | XSS in UI from hostile labels/filenames rendering | B1 | React escaping only; no `dangerouslySetInnerHTML`; lint rule | XSS in admin session = root (see Actors) — hence priority |

## 5. Explicit non-goals
- Protecting the system from a Cockpit administrator acting maliciously.
- Securing Cockpit's own login, TLS, or websocket transport.
- Encrypting data at rest (LUKS layouts are inspect-only in v1).

## 6. Testable security invariants (become tests in M1+)
- I1 No call path from `rpc` to `subprocess` except via `exec.run(tool_name, argv_list)`.
- I2 Every mutating method returns `E_NOT_ROOT` when `euid != 0`.
- I3 Every irreversible plan requires `confirm_token == expected stable_id`, checked server-side.
- I4 Every value written to a config file round-trips through its parser unchanged.
- I5 Fuzzed JSON requests never produce an unhandled exception or a non-JSON stdout.
