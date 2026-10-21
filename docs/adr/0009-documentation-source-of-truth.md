# ADR-0009: Documentation source of truth

- Status: **Accepted** (owner decision Q5, 2026-10-19)
- Date: 2026-10-19

## Decision
- The GitHub repository's `docs/` directory is **authoritative** for requirements, architecture,
  decisions, safety cases and status.
- Claude Project Knowledge (the claude.ai Project documents and instructions) is **contextual
  reference only**. When it conflicts with the repository, the current repository
  documentation and accepted ADRs win.
- Requirement changes are made by editing repository documents through a reviewed branch; the
  change is recorded in `docs/DECISIONS.md` (and an ADR when it is an architectural decision).
- Known divergence at closeout: the Project instructions still name the product
  `cockpit-pool-manager`; the repository name `cockpit-storage-manager` (owner decision Q2) prevails.

## Consequences
- `CLAUDE.md` points contributors and agents to `docs/` first.
- `docs/PRODUCT_SPEC.md`, `ARCHITECTURE.md` and `SAFETY_AND_TESTING.md` started as verbatim copies of
  the Project documents and are now maintained here; amendments are marked in place.
