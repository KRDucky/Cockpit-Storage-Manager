# ADR-0008: Independent mergerfs pools and SnapRAID arrays

- Status: **Accepted** (M0 closeout, 2026-10-19; R1)
- Date: 2026-10-19

## Context
Owner requirement R1: mergerfs and SnapRAID must be independently configurable; no
Unraid-style single array. Upstream supports this: mergerfs pools are independent FUSE mounts;
SnapRAID arrays are independent config files selected with `-c`, each with its own parity and
content files.

## Decision
1. **Three deployment modes**, all first-class, chosen per object not per host:
   - pooling only (mergerfs, no parity);
   - parity only (SnapRAID data disks mounted individually, no pool);
   - combined (a pool whose branches are data disks of one or more arrays).
2. **Data model:** `Pool` and `SnapraidArray` are separate top-level collections.
   `Pool.pool_id` is a stable UUID; `SnapraidArray.array_id` is the realpath of its config file.
   Relationships are derived, never stored as ownership: a *link* is reported when a pool
   branch and an array data path resolve to the same filesystem UUID.
3. **Multiplicity:** any number of pools and arrays per host. A filesystem may be a branch of
   more than one pool (allowed by mergerfs, shown with a warning) but a data member of **at most
   one** array (two arrays protecting the same files is a configuration hazard ⇒ finding,
   writes blocked).
4. **UI:** independent Pools and Parity pages, each listing all objects of its kind; each
   object shows its links ("branch of pool media", "data disk d2 of array main") as navigation,
   never as a prerequisite. Overview aggregates by object, not by assuming one array.
5. **Locks** are per pool and per array (plan-engine §5); work on one array never blocks another.
6. **Workflows** branch on mode: replacing a disk in a pooling-only pool is evacuation/copy,
   never parity recovery; recovery for a parity-only array does not touch any pool.

## Consequences
- No hidden coupling: removing SnapRAID from a host leaves pools untouched, and vice versa.
- Recovery simulation (ADR-0010) must evaluate the mode of every affected object.
