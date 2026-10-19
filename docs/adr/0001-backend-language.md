# ADR-0001: Backend helper language — Python vs Rust

- Status: **Proposed** (M0, awaiting review)
- Date: 2026-10-19
- Deciders: project owner

## Context

The privileged helper (`cpm-helper`) parses untrusted host state (`lsblk`, `findmnt`,
`/proc/self/mountinfo`, `fstab`, `snapraid.conf`, SnapRAID log tags, Samba/NFS config),
builds and validates plans, and executes a fixed allowlist of executables with argument
arrays. Its failure modes that matter are **logic errors** (wrong device, stale plan,
missed precondition) far more than memory corruption: the helper never parses binary
formats itself, and all input reaches it as text or JSON.

Constraints:

- Must package cleanly for Debian 13 and Ubuntu 26.04/26.10 (`.deb`), ideally
  installable from distro archives later without a vendored dependency tree.
- Must run where Cockpit runs. Cockpit's own bridge is Python (`src/cockpit/*.py`
  in upstream), so `python3` is already a hard dependency of every target host.
- Contributors are likely home-lab admins; readability and auditability matter.
- Must support strict typing, schema validation, property/fuzz testing.

## Options

| Criterion | Python 3 (stdlib-only runtime) | Rust |
|---|---|---|
| Runtime deps on target | None beyond `python3` (already required by `cockpit-bridge`) | None (static-ish binary) |
| Debian packaging | Trivial (arch: all) | Arch-specific builds; Debian policy discourages vendored crates in main; own `.deb` fine |
| Memory safety | Safe | Safe |
| Type safety | Opt-in: `mypy --strict`, frozen dataclasses, `typing.Literal`, exhaustive `match` | Enforced by compiler; enums with exhaustive match are a genuine advantage for state machines |
| Subprocess hygiene | `subprocess.run(list, shell=False, env=…, cwd=…, timeout=…)` — easy to get right, easy to wrap in one audited function | `std::process::Command` — equally good |
| Parsing text configs | Fast to write and review | More code, more ceremony |
| Fuzz / property tests | `hypothesis` (test-only dep) | `cargo-fuzz`, `proptest` — stronger tooling |
| Startup cost per call | ~30–60 ms interpreter start; acceptable for request/response helper | ~1 ms |
| Reviewability by target audience | High | Medium |
| Build in CI / reproducible builds | Simple | Toolchain pinning; reproducible but heavier |
| Iteration speed during M1–M4 | High | Medium |

## Decision (proposed)

**Python ≥ 3.11, standard library only at runtime**, with these compensating controls
for Python's weaker static guarantees:

1. `mypy --strict` and `ruff` as CI gates; no `Any` in contract modules.
2. Contracts defined once as **JSON Schema** (`api/schema/*.json`), from which both the
   Python dataclasses' validators and the TypeScript types are generated/checked in CI.
   The backend validates every request against the schema before dispatch.
3. Exactly one module (`backend/exec.py`) may call `subprocess`; a CI grep test fails the
   build on `shell=True`, `os.system`, `os.popen`, or any other `subprocess` import site.
4. Plan/job states are `enum.Enum` with transition tables tested exhaustively.
5. `hypothesis` property tests for every parser (round-trip, never-crash, bounded output).

Rust stays a recorded alternative: if a later milestone needs a long-running component
(e.g., an event watcher) or performance-sensitive hashing, that component may be Rust
behind the same JSON contract without rewriting the rest.

## Consequences

- `+` No compiled artifacts; `arch: all` package; same code path for mock mode and real mode.
- `+` Aligns with Cockpit's own bridge language; maintainers can read both.
- `−` Type errors are caught by CI, not by the compiler; discipline must be enforced by gates.
- `−` Python version drift across distros: floor of 3.11 is below every target
  (Debian 13 ships 3.13). Re-verify Ubuntu 26.04/26.10 interpreter versions in the M1 CI matrix.
