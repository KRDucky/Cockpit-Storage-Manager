# ADR-0003: Privilege, authorization and job execution model

- Status: **Proposed** (M0, awaiting review) — contains a **deviation from ARCHITECTURE.md** that needs owner sign-off (see "Deviation").
- Date: 2026-10-19

## Context: what Cockpit actually gives us (verified in upstream source)

1. `cockpit.spawn(argv, {superuser: "require"})` runs the process as root if the logged-in
   user may escalate (sudo/pkexec); otherwise the channel closes with `access-denied`
   (`doc/modules/guide/pages/cockpit-spawn.adoc`).
2. Once a user enables Cockpit **administrative access**, *any* JavaScript running in any
   Cockpit page of that session can open root channels. There is no per-package sandbox.
3. The bridge registers a polkit authentication agent **only while escalating to
   administrative access** (`src/cockpit/superuser.py` → `PolkitAgent`); it is unregistered
   afterwards. A system D-Bus service that calls `CheckAuthorization` with
   `AllowUserInteraction` would therefore have **no agent to prompt** in normal operation.

Conclusion: inside a Cockpit admin session, our backend **cannot be a security boundary
against the admin's own browser session** — that session is already root. The backend is:

- a **security boundary** against non-admin Cockpit users, against hostile *data*
  (labels, filenames, config contents, command output), and against our own UI bugs;
- a **safety boundary** that enforces preflight, plan freshness, typed confirmation and
  locking regardless of what the UI sent.

## Decision (proposed)

### Helper shape
- One executable, `/usr/libexec/cockpit-pool-manager/cpm-helper`, invoked by
  `cockpit.spawn` with **fixed argv** `["/usr/libexec/cockpit-pool-manager/cpm-helper", "rpc"]`.
  The request is a single JSON object on stdin; the response is a single JSON object on
  stdout. No user-controlled argv, ever.
- Read-only methods are invoked **without** `superuser` first. The helper returns
  partial results with explicit `"privilege": "insufficient"` markers for fields needing
  root (e.g., SMART, `blkid -p` probes) rather than failing the whole view. The UI may
  retry with `superuser: "try"` when the user has admin mode on.
- Mutating methods are invoked with `superuser: "require"`. The helper itself re-checks
  `geteuid() == 0` and refuses mutations otherwise (backend never trusts the UI).
- No persistent daemon (per ARCHITECTURE.md).

### Long-running jobs survive the browser
Sync/scrub/copy/evacuation can take hours. A job started over a `cockpit.spawn` channel
dies with the websocket. Therefore:

- `cpm-helper rpc` with `job.start` validates the plan, takes locks, writes
  `/var/lib/cockpit-pool-manager/jobs/<job-id>/plan.json` (0600 root), then launches
  `systemd-run --unit=cpm-job-<job-id> --property=… /usr/libexec/cockpit-pool-manager/cpm-helper job-run <job-id>`
  and returns immediately. `<job-id>` is a server-generated UUID, never client input.
- The job process owns the locks (see `docs/design/plan-engine.md`), appends a JSON-lines
  journal to `jobs/<id>/events.jsonl` (fsync per phase boundary), and logs to journald.
- The UI follows progress by polling `job.get` / reading the events file through the
  helper; reconnecting browsers resume viewing. Scheduled sync/scrub use the same
  `job-run` entry point from a timer, so manual and scheduled runs share one code path
  and one lock.

### Executable allowlist
`backend/exec.py` holds an immutable table `{logical_name: absolute_path}` resolved at
startup from fixed candidate paths (`/usr/bin`, `/usr/sbin`, `/bin`, `/sbin`), each checked:
root-owned, not group/world-writable, regular file, real path inside those dirs.
Environment is replaced with a fixed minimal set (`PATH`, `LANG=C.UTF-8`, `LC_ALL=C.UTF-8`),
`cwd=/`, stdin closed unless needed, stdout/stderr capped (default 16 MiB, then the process
is killed and the result marked `output-truncated`), wall-clock timeout per call.

### Authorization tiers that are actually enforceable

| Tier | Who | Mechanism | Allowed |
|---|---|---|---|
| Viewer | any Cockpit user | unprivileged helper | discovery, config inspection, job status, logs readable by user |
| Administrator | Cockpit admin mode | root helper | everything, **subject to safety gates** |
| Irreversible | Administrator + typed confirmation | server-side check of `confirm_token` == expected stable ID and `plan_hash` | format/partition, parity replace, `fix`, forced flags |

## Deviation from ARCHITECTURE.md (needs owner decision)

ARCHITECTURE.md asks for "narrowly scoped polkit authorization" with "distinct permissions
for status inspection, configuration changes, share updates, manual jobs and irreversible
disk initialization". Given facts 2–3 above, per-action polkit prompts are not achievable
from Cockpit without a polkit agent, and per-action polkit checks add nothing once the
session is root. Options:

- **A (recommended for v1):** Tiers above. Ship polkit action definitions only for a
  *future* delegated-operator mode (non-admin group allowed to run scrub/status via a
  D-Bus-activated service with `polkit` rules granting `yes` to a configured group,
  no interactive auth). Document plainly that admin mode = root.
- **B:** Build the D-Bus-activated service now with non-interactive polkit rules per action.
  More code and attack surface in M1 for a benefit that only applies to delegated users.
- **C:** Rely on Cockpit admin mode only, no tiers. Rejected: loses viewer mode and server-side irreversible gate.

## Consequences
- Honest threat model (see `docs/design/threat-model.md`).
- Jobs are systemd units: visible in `systemctl`, journald-logged, survive UI disconnect,
  killable only through our cancellation policy (we never send `SIGKILL` to `snapraid fix`).
- `KillMode=none` (used in upstream mergerfs examples) is deprecated in current systemd;
  job and pool units must use supported kill modes. Verify behavior per target in M1.
