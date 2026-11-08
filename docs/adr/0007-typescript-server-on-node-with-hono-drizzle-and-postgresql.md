---
status: accepted
---

# TypeScript server on Node with Hono, Drizzle and PostgreSQL

The maintainer delegated the backend choice, asking that the server share domain code with the apps. The server is TypeScript on Node 26 LTS: Hono on its Node adapter, listening on a socket-activated Unix socket, with server-sent events for live updates; Drizzle ORM for typed queries over vanilla PostgreSQL 19 with no extensions; oidc-provider and SimpleWebAuthn for sign-in (ADR 0011); and a separate worker process running pg-boss jobs, so jobs need nothing but PostgreSQL. Because the apps' shared UI is TypeScript too (ADR 0004), `packages/domain` holds the money and time types, the mutators, the budget engine (safe to spend, daily budget, pace, rollover, shares, limit states) and the glance builder once, and the server and the Expo app run that same code. The server runs the budget engine for alerts and the glance, the apps for their screens, and SQL only fetches and sums, since views would have to exist in both SQLite and PostgreSQL.

## Considered Options

- Swift with Hummingbird, the research's pick while the apps were to be SwiftUI. Beside a TypeScript UI it would share code only with the watch, which merely decodes the glance: every mutator would be ported and kept in step by golden vectors, and the authorization server hand-rolled on a beta WebAuthn library.
- Bun: nixpkgs has no builder for Bun projects, and everything here is packaged with Nix (ADR 0003).

## Consequences

- Schema changes are reviewed SQL migration files, which drizzle-kit may draft and squawk lints, embedded in the package (ADR 0018).
- systemd units keep every hardening option except `MemoryDenyWriteExecute`, which Node supports only in its slow `--jitless` mode.
- Swift code meets the server only through pinned contracts: the glance JSON, and later the watch actions endpoint (ADR 0006).
- The API has no network beyond local sockets, so it hands the worker the calls that reach the data provider as pg-boss jobs and waits for their results.
- SP-04 confirms or amends this choice.
