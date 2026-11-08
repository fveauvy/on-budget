---
status: accepted
---

# Build the sync engine in TypeScript

The protocol of ADR 0008 is built in-house, as one TypeScript `sync-core` package with injected SQLite, transport, auth and clock ports. It runs on the iPhone's JavaScript thread over expo-sqlite with SQLCipher, and in each browser profile in one leader worker over the official SQLite WASM build on OPFS. Why: the hard part is the domain semantics, which a sync service would leave to the project anyway, since writes stay named mutations through Shoestring's API, and a household's data is a few megabytes, so nothing needs partial sync. Building keeps each instance at one API and one PostgreSQL.

## Considered Options

- PowerSync, whose service moved to Apache-2.0 on 2026-11-03, run as a Nix-built package under systemd: less protocol code to own, but a second service with its own storage, a second hostname and certificate on the tailnet, a token that JavaScript can read, and a replication role that bypasses row-level security, moving household isolation into its sync rules. It comes back only if SP-09 shows the custom protocol failing to converge.

## Consequences

- The project builds the browser plumbing: a leader elected with Web Locks, tabs served over BroadcastChannel, and a new bootstrap whenever the browser evicts the database. expo-sqlite's web backend isn't used: it starts a worker per tab without coordination and needs cross-origin isolation, which would switch Expo's workers to `blob:` URLs that the content security policy blocks.
- Each device keeps one database per household, with its own outbox, cursor and client ID, and a browser profile holds one person's data.
- A browser's database can vanish (Safari clears a plain tab's storage after seven days without interaction), so devices push their outbox within seconds.
- A client schema change keeps the outbox and bootstraps the rest again, so client migrations never touch replicated tables (`docs/architecture.md`, Client mutations and sync).
- On the web, encryption at rest waits for spike SP-25 (WEB-W8); until then the database relies on the system's disk encryption behind a passkey lock, and the app says so.
