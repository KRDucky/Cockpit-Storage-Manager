# ADR-0010: Recovery Simulation / Replacement Readiness as a mandatory gate

- Status: **Accepted** (M0 closeout, 2026-10-19; R3, R6)
- Date: 2026-10-19

## Decision
1. Every replacement, retirement, evacuation and recovery plan requires a **readiness report**
   produced by the read-only feasibility engine (`docs/design/recovery-simulation.md`).
2. The report is bound to the same host-snapshot fingerprint as the plan. `plan.create` for a
   recovery refuses without a report fresher than the plan TTL; `job.start` **re-runs** the
   evaluation and refuses unless the verdict is still `READY` and the blocking set is empty.
3. Verdicts: `READY` (all verified prerequisites satisfied; success not guaranteed),
   `BLOCKED` (a required prerequisite is absent or a known safety condition prevents
   proceeding), `INDETERMINATE` (evidence insufficient). Only `READY` permits execution.
   An `INDETERMINATE` result can become `READY` only through new evidence (e.g., a
   non-destructive `check` result, a located content copy), never by overriding.
4. The engine is a reusable backend planning component with a typed contract and mock
   fixtures; it runs no mutating command. Its process-execution allowlist is a **strict subset**
   of read-only tools; `mkfs*`, `wipefs`, partitioners, `snapraid fix`, `snapraid sync`,
   `snapraid touch`, `mount`/`umount`, `chattr` and mergerfs runtime writes are not reachable
   from it (enforced by a static test).
5. Two analysis tiers, labelled separately in the UI:
   - **Static preflight** (default, fast): configuration, inventory, identity, last-sync
     evidence, `snapraid status`/`diff`/`list` output.
   - **Non-destructive verification** (optional, long-running, explicitly started):
     `snapraid check` (simulates recovery, writes nothing to the array, per manual §5.10; reads every surviving disk, so it stresses already-aging drives — shown before starting)
     scoped with `-d`/`-f`, or `check -a` where parity is not needed.
   Neither tier is described as a recovery, parity validation or proof of file integrity.
6. Delivery order: the read-only simulator foundation ships (M4) **before** any destructive
   replacement or recovery operation (M8/M9).

## Consequences
- Recovery UX becomes "evaluate → understand → decide", matching the upstream procedure
  (disable writers and schedulers, reconfigure, fix, check, then decide on sync).
- The simulator is useful on its own for "what happens if this disk dies?" planning.
