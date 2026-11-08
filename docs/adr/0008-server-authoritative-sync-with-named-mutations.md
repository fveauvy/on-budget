---
status: accepted
---

# Server-authoritative sync with named mutations

Every app works offline, and two members may edit the shared plan at once, each on several devices. Sync follows the Linear and Replicache model. Each device keeps a SQLite copy of its member's spaces (ADR 0010) and an outbox of named, versioned mutations; the instance applies them in order under one instance-wide sync counter and records every change in a change log that devices pull. A device's stored rows always equal the instance's state at its cursor, and pending mutations are replayed on top. Independent fields, such as a category, a payee or a note, use per-field last-writer-wins in the order the instance applies the edits, behind a source rank (a person's choice beats a categorization rule, which beats an import); an edit made more than 24 hours before it reaches the instance loses to any change the field received after the device's cursor at the time of the edit, so a device back from days offline doesn't overwrite newer edits. Changes whose intent is a delta or a constraint (reallocations, category-line sets, matches, deletes) are operations that the instance validates and applies in one transaction. Why: the instance writes data itself (refreshes, categorization rules), reads it (alerts, the glance) and enforces access, which rules out Actual Budget's model of a relay that never reads the data; and last-writer-wins on absolute values loses concurrent reallocations: when two members each move €50 into one category, one move is lost.

## Considered Options

- CRDTs (Automerge, Yjs, cr-sqlite) merge without the instance's invariants, such as category lines that sum to their transaction's amount or a booked transaction that matches at most one entry.
- CloudKit: a server can't write into members' private or shared databases.
- Last-writer-wins by hybrid logical clock (`sync-engine.md` §4): it keeps the newer intent between any two concurrent edits, but stores a value, a clock and a source for every field in SQLite and PostgreSQL, needs server-side clock bounds and a clock warning in every app, lets an acknowledged edit lose later, and rules out keeping a pending edit over an incoming value. The 24-hour rule covers the case it buys: a device offline for days.

## Consequences

- Budget data isn't end-to-end encrypted, because the instance must categorize, raise alerts and build the glance. It is protected in transit, at rest and on devices instead, and whoever runs the instance can read it, which ADR 0001 makes the owner.
- Derived numbers (safe to spend, balances, unallocated) are never synced; both sides compute them with `packages/domain` (ADR 0007).
- The server keeps each annotation field's source and the sync ID of its last change; devices keep values only, and their clocks matter only to the 24-hour rule.
- The project owns the protocol's correctness: a seeded simulator of several devices with faults, checking convergence and money invariants, runs in CI from the start.
- Mutation versions and restores follow ADR 0018.
