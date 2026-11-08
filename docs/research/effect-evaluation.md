# Effect for the server, the sync engine and the domain package

> **Status (2026-11-08).** Background research for [#13](https://github.com/fveauvy/shoestring/issues/13); [#21](https://github.com/fveauvy/shoestring/issues/21) makes the decision. The ADRs, [`SPEC.md`](../../SPEC.md), [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md) win where this note disagrees. It edits none of them.

> Research note, 2026-11-08. Versions are the newest stable releases or release candidates that day, read from the npm registry and GitHub releases; beta, canary and snapshot tags are left out. T3 Code is read at commit [`6417151`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a) (2026-11-08), from a shallow clone that was read but never installed or run. Confidence: **H** high, **M** medium, **L** low. "Measured" marks a check run for this note, "judgment" my own assessment, "secondary" a non-primary source.

## Question

What would Effect bring to Shoestring's server (`apps/server`: API, worker, migrator and host CLI), sync engine (`packages/sync-core`) and domain package (`packages/domain`), and what would it cost?

- How does T3 Code use Effect in its server, `client-runtime`, `@effect/atom-react`, `@effect/platform-node`, `@effect/vitest` and `@effect/tsgo`?
- Which version is current?
- What does Effect change in these decisions?
  - ADR 0007: Hono, Drizzle and pg-boss.
  - ADRs 0008 and 0009: the sync engine.
  - `packages/domain`, which runs on the server, in the Expo app and on the web.

  Cover typed errors, services and layers, Effect Schema, concurrency and testing.
- What does it cost in learning, in bundle size on Hermes and the web, and in interop with Drizzle and Hono?
- Should Shoestring adopt all of it, part of it, or none?

## TL;DR

**Recommendation: adopt part of it** (M).

- **Use Effect's stable core in two packages:** `apps/server` (API and worker) and `packages/sync-core`.
- **Use Effect Schema in the domain package**, for its codecs only.
- **Keep the domain logic plain.** The budget engine and the mutators' rules stay synchronous, pure TypeScript, so golden vectors and the simulator call them directly. Temporal stays the time library (ADR 0017).
- **ADR 0007 holds.** Hono stays the HTTP edge and runs Effect programs through one `ManagedRuntime`, the bridge Effect itself documents for Hono. Drizzle on node-postgres, pg-boss, oidc-provider and SimpleWebAuthn sit behind Effect services.
- **Stay on stable APIs.** Production code avoids modules tagged `@stability unstable`:
  - HTTP, HttpApi and RPC;
  - SQL and atoms;
  - crypto and `@effect/platform-node`;
  - durable queues and workflows.

  Tests may use `@effect/vitest`, also unstable. The Effect language service's `unstableApiUsage` diagnostic enforces the rule.
- **Two gates:** SP-18 measures Effect on Hermes (bundle and first frame), and SP-09's simulator is written on `TestClock`.

Facts behind it:

1. **Version** (H).
   - `effect` 4.1.2 (2026-11-02) is `latest`; 4.0.0 shipped on 2026-10-01.
   - The `@effect/*` packages release in lockstep at 4.1.2. `@effect/tsgo`, the language service for TypeScript 7, is at 0.67.5.
   - The `rc` and `beta` tags point at stale pre-4.0 builds, and 3.22.4 under `prev` is the old line.
   - Effect 4.x gets bug fixes until September 2029 or one year after 5.0, and security fixes until September 2029 or two years after 5.0, whichever is later.
2. **Stable core, unstable edges** (H, measured).
   - In the 4.1.2 package these are `@stability stable`: `Effect`, `Layer`, `Context`, Schema's core, `Stream`, `Queue`, `Schedule`, `Fiber`, `Scope`, `Clock`, `Random`, `ManagedRuntime` and `TestClock`.
   - Every subpath module is `@stability unstable`: `http`, `http-api`, `rpc`, `sql`, `reactivity`, `crypto`, `persistence`, `workflow`, `cluster` and `cli`. So is every satellite package checked. Such modules "may change in minor or patch releases".
   - 4.1 broke several of them four weeks after 4.0.
3. **T3 Code runs Effect 4.1.2 everywhere** (H).
   - Its server is Effect from the listener down, with no Hono or ORM: 193 services, 458 tagged errors and 3,221 `it.effect` tests.
   - `client-runtime` shares Effect services between the web, desktop and Expo apps, which read state through `@effect/atom-react`.
   - Each package typechecks with `effect-tsgo`. 86 Effect diagnostics set to error, a conventions guide, custom lint rules, a review check and a vendored copy of Effect's source keep the code idiomatic.
4. **What it brings** (M-H):
   - typed errors that separate the push protocol's outcomes: a domain rejection or a permanent error advances the queue, a transient error doesn't;
   - ADR 0009's ports as services with test layers, and the worker-only KEK as a rule the type checker enforces (judgment);
   - a Schema whose `Int` is exactly ADR 0017's safe integer;
   - retries, timeouts, interruption and scoped cleanup for the push loop, pokes and the web leader's handover;
   - virtual time and seeded randomness for the 24-hour rule, consent timers and backoff.
5. **What it costs** (M, measured):
   - **Learning.** Effect's own estimate is "a few focused days" for the basics, then a large API: T3 Code's server imports 149 distinct Effect paths.
   - **Bundle.**
     - On the web (Rolldown, tree-shaken), a sync-core probe is 36 kB gzipped, and 46 kB with Effect's HTTP client or atoms.
     - Metro doesn't tree-shake by default, so on Hermes the same probe is about 0.56 MB minified. That is close to Zod's size there, and 0.48 MB more than Valibot.
     - An import from the package root pulls 0.87 MB on Hermes.
   - **Hermes.**
     - Effect's installation page lists Node, Deno and Bun only.
     - Hermes lacks SubtleCrypto and `FinalizationRegistry`, and Hermes bugs were still being fixed in October 2026.
     - Effect has no driver for expo-sqlite or the official SQLite WASM build, so both `SqlDriver` adapters stay hand-written, as ADR 0009 already plans.
   - **Interop.**
     - Hono works through `ManagedRuntime`, or with Effect routes behind Hono 5's `mount`.
     - Drizzle 1.0's RC ships native Effect drivers, but on the unstable `@effect/sql-pg`, and Drizzle's page still shows Effect 3 APIs.
     - pg-boss has no integration, so a service wraps it.

## 1. Effect in November 2026

### 1.1 Versions

Read on 2026-11-08 from npm dist-tags (H).

| Package | Version (date) | Note |
|---|---|---|
| `effect` | 4.1.2 (2026-11-02) | `latest`. 4.0.0 on 2026-10-01, 4.1.0 on 2026-10-29. No runtime dependencies; MIT |
| `effect` 3.x | 3.22.4 (2026-10-22) | Under `prev` |
| `effect` under `rc` and `beta` | 4.0.0-rc.118, 4.0.0-beta.107 | Pre-4.0 builds, older than `latest` |
| `@effect/platform-node`, `@effect/vitest`, `@effect/atom-react`, `@effect/sql-pg`, `@effect/sql-sqlite-wasm`, `@effect/sql-sqlite-react-native`, `@effect/opentelemetry` | 4.1.2 (2026-11-02) | Released with `effect`, which each needs at `^4.1.2` |
| `@effect/tsgo` | 0.67.5 (2026-11-07) | Effect diagnostics built into TypeScript-Go and tsrs; supports TypeScript 7.0.2 and Oxlint 1.88 to 1.92 |
| `@effect/language-service` | 0.87.5 (2026-10-16) | The same service as a TypeScript plugin. T3 Code configures `effect-tsgo` under this plugin name without installing it |
| `@effect/platform`, `@effect/sql`, `@effect/rpc`, `@effect/schema`, `@effect/experimental`, `@effect/cluster`, `@effect/workflow` | 0.x | Effect 3 packages, now modules of `effect` |
| `@effect/sql-drizzle` | 0.51.0 (2026-07-13) | Effect 3 only: peers `effect ^3.22.0` and `drizzle-orm <0.50` |
| `@effect-atom/atom-react` | 0.7.0 | The community Effect 3 package; `@effect/atom-react` replaces it |

### 1.2 What 4.0 changed

From the release post (H for what it says; the runtime benchmarks are Effect's and weren't rerun):

- **Rebuilt core.** A minimal program shrank from 35.6 kB to 7.1 kB minified and gzipped. Task throughput is 6.4 times higher, and 50,000 fibers use 21.8 MB of heap instead of 157.5 MB.
- **One package.** Many packages moved into `effect`, the others release in lockstep with it, and the core has zero runtime dependencies.
- **Long-term support.** It starts with 4.x: bug fixes until September 2029 or one year after 5.0, and security fixes until September 2029 or two years after 5.0, whichever is later.
- **Stability tags.** "Some of the newer modules are still tagged @stability unstable or @stability experimental, which means they may change in minor or patch releases."

### 1.3 What is stable

Measured: each JSDoc `@stability` tag in the `effect@4.1.2` tarball counted, then the same for the satellite packages (H).

- **Stable**: the top-level modules a server and a sync client need.
  - `Effect` (523 stable tags, 4 unstable), `Layer`, `Context`, `Stream` (including `toReadableStream` and `fromReadableStream`), `Queue`, `PubSub`, `Schedule`, `Fiber`, `Scope`, `Semaphore` and `Deferred`.
  - `Clock`, `Random` (including `withSeed`), `Config`, `Logger`, `ManagedRuntime` and `testing/TestClock`.
  - Schema's core: `Struct`, `Int`, `brand`, the `decode*` and `encode*` functions, `TaggedError`, `Class`, `Literals`, `Union`, `toStandardSchemaV1` and `toJsonSchemaDocument`.
- **Unstable, top level**:
  - `FileSystem`, `Path`, `Arbitrary` and `Graph`;
  - Schema's network, crypto-key and SemVer types.
- **Unstable, every subpath module**:
  - `http` (886 unstable tags, 6 experimental), `http-api`, `rpc` and `sql`;
  - `reactivity` (atoms), `crypto`, `persistence`, `workflow` and `cluster`;
  - `cli`, `process`, `socket`, `workers` and `observability`.
- **Unstable, every satellite package checked**: `@effect/platform-node` (132 unstable, 12 experimental), `@effect/sql-pg`, `@effect/sql-sqlite-wasm`, `@effect/sql-sqlite-react-native`, `@effect/atom-react` and `@effect/vitest`.

The [4.1 release notes](https://effect.website/blog/releases/effect/41) show what "unstable" means in practice. One minor release after 4.0, it lists breaking changes, among them:

- `effect/Crypto` replaced by `effect/crypto`, and crypto services removed from the runtime bundles;
- `sql.insert` and `sql.updateValues` requiring non-empty arrays;
- `Migrator.fromFileSystem` moved "so effect/sql can be imported under Metro in React Native";
- RPC `discard` calls completing on acknowledgment;
- new `KeyValueStore` and Redis key layouts;
- cookie handling in `HttpClient`.

## 2. How T3 Code uses Effect

T3 Code pins `effect` and every `@effect/*` package at 4.1.2 in one pnpm catalog. Its server runs coding-agent CLIs, git and terminals. Its clients are a web app, an Electron desktop app and an Expo app. Counts below are measured on the clone (H).

### 2.1 Server

- **Effect from the listener down.**
  - `NodeHttpServer.layer` wraps a `node:http` server, with `HttpRouter.serve` for routing.
  - REST comes from `HttpApiBuilder.layer(EnvironmentHttpApi)`; WebSocket RPC uses `effect/rpc` groups defined in `packages/contracts`.
  - `Layer.launch` starts the server, and `NodeRuntime.runMain` the CLI ([`server.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/server.ts)).
- **No ORM.** SQLite goes through T3 Code's own `SqlClient` over `node:sqlite`, with a reader worker and Effect migrations ([`Sqlite.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/persistence/Sqlite.ts)).
- **Scale.** 1,360 files, 919 of them importing `effect/Effect`; 193 `Context.Service` classes; 458 `Schema.TaggedError` classes; 3,221 `it.effect` tests in 474 test files; 149 distinct Effect import paths.
- **Shape.** Its guide, [`effect-services.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/internals/effect-services.md), sets these rules:
  - every capability is a method on a service;
  - a transport handler only decodes, calls one method and maps typed errors;
  - dependencies come from the environment, never from parameters;
  - reusable functions are `Effect.fn("Service.operation")`, which creates a span;
  - `ManagedRuntime` and `runPromise` appear only at framework boundaries;
  - errors carry bounded attributes and no raw payloads.
- **Example.** [`Scheduler.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/scheduling/Scheduler.ts) registers due work with a one-permit `Semaphore`, forks it into the caller's scope and removes it with `acquireRelease`. [Its tests](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/scheduling/Scheduler.test.ts) advance `TestClock` instead of sleeping.

### 2.2 Contracts

[`packages/contracts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/contracts/src/environmentHttp.ts) holds 108 files and 159 tagged errors, written in Effect Schema:

- the REST API as `HttpApiGroup` and `HttpApiEndpoint` values, with schemas for headers, params, query, success and errors;
- WebSocket methods as `Rpc.make` groups.

The server and every client import the same contract.

### 2.3 `client-runtime`

[`client-runtime`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/client-runtime/README.md) holds "shared client behavior for web and mobile": 392 files, 28 services and 388 `it.effect` tests.

- **Contents:** authorization, connection supervision with retries, RPC clients, and state modules that export Atom constructors.
- **Ports:** its `platform` subpath declares persistence ports as `Context.Service` classes with typed errors, such as [`ConnectionTargetStore`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/client-runtime/src/platform/persistence.ts). Each app provides them as layers; the Expo app's version is [`storage.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/src/connection/storage.ts).

That is the same design as ADR 0009's injected ports.

### 2.4 `@effect/atom-react`

Both UIs read state through atoms:

| Call | Web | Expo |
|---|---|---|
| `useAtomValue` | 443 | 396 |
| `Atom.make` | 304 | 280 |
| `Atom.family` | 76 | 91 |
| `RegistryContext` | 95 | 66 |

The package's peers are React 19 and `scheduler`.

### 2.5 `@effect/platform-node`

- **Server:** `NodeServices` (file system, paths, child processes), `NodeHttpServer` with `ws` for WebSockets, `NodeRuntime` and `NodeCrypto`.
- **Web and Expo apps:** a development dependency only, for tests.
- **Package:** depends on `undici` and `@effect/platform-node-shared`, and is unstable as a whole.

### 2.6 Effect on Hermes

The Expo app imports Effect in 523 places, always from deep paths (`effect/Effect`); the one root import is in a test. Three practices stand out:

- **No tree-shaking flag.** Its Metro configuration sets none.
- **Hand-written SQLite.** It wraps expo-sqlite in its own Effect services ([`mobile-database.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/src/persistence/mobile-database.ts)).
- **Native crypto.** It provides Effect's `SecureRandom` and `Digest` from expo-crypto, because "Hermes has no SubtleCrypto, so Effect's WebCrypto layers cannot back these services" ([`dpop.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/src/features/cloud/dpop.ts)).

### 2.7 `@effect/vitest`

- **Test kinds.** `it.effect` runs on a `TestClock`, `it.live` on wall-clock time (252 in the server), and `it.layer` shares a layer (159).
- **No sleeps.** [`AGENTS.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/AGENTS.md) forbids waiting on sleeps or polling: "`it.effect` runs on a `TestClock`; `Effect.sleep` only advances through `TestClock.adjust`. `it.live` is wall-clock and only for subprocess or network fixtures."
- **Versions.** The package needs Vitest 5.

### 2.8 `@effect/tsgo` and `tsconfig.effect.json`

- **Typecheck.** Every package runs `effect-tsgo tsrs --noEmit`, "the Rust TypeScript 7 checker with Effect's diagnostics built in. There is no `typescript` package to run" (`AGENTS.md`). The root depends on `@effect/tsgo` 0.67.1.
- **Diagnostics.** [`tsconfig.effect.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/tsconfig.effect.json) sets 86 diagnostics to error. They cover:
  - running an Effect inside another, missing service dependencies and leaking requirements;
  - `try`/`catch`, global `Date`, `console`, `Math.random`, timers and `fetch` inside Effect code;
  - untyped error channels.
- **Unstable APIs allowed.** It turns `unstableApiUsage` and `experimentalApiUsage` off: "We use Effect's unstable modules (http, sql) on purpose."
- **Shared base.** The file stays apart from `tsconfig.base.json` so that the Expo app can combine it with Expo's base.
- **Lint.** Oxlint, run by Vite+, adds T3 Code's own rules `prefer-effect-fn` and `no-manual-effect-runtime-in-tests` ([`vite.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/vite.config.ts)). Ticket #8 covers the rest of the gates.

### 2.9 Agents

- **Agent guidance.** `AGENTS.md` sends agents to the services guide and to `.repos/effect/LLMS.md`.
- **Vendored sources.** `.repos/` holds full copies of Effect's and Alchemy's repositories (24,126 tracked files), pinned to the catalog versions by [`sync-reference-repos.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/lib/reference-repos.ts).
- **Effect's advice.** That follows Effect's [own recommendation](https://effect.website/blog/the-one-weird-git-trick-that-makes-coding-agents-more-effect-ive/) to vendor the source with `git subtree`. Its reason: agents "are deoptimized from exploring node_modules or other gitignore directories".
- **Review check.** A review check enforces the service conventions.
- **Shipped docs.** The `effect` npm package also ships `AGENTS.md`, `ai-docs/` and its TypeScript sources.

**Reading** (judgment):

- T3 Code shows that Effect 4 can carry one TypeScript monorepo across Node, browsers, Electron and Hermes.
- It also shows the price:
  - guard rails in the type checker, the linter, a guide and code review;
  - a vendored copy of Effect for agents;
  - an explicit choice to build on unstable modules.

## 3. What Effect would bring

### 3.1 Typed errors

- **The model.** An `Effect<A, E, R>` lists its expected failures in `E`. Defects (bugs, broken invariants) stay out of `E`, in the `Cause` ([docs](https://effect.website/docs/v4/error-management/two-error-types.md)) (H).
- **Push outcomes.** The push protocol already has three outcomes (`sync-engine.md`, Minimum viable protocol, steps 5 and 6):
  - a domain rejection: record the reason and advance;
  - a permanent database error: roll back to the savepoint and advance;
  - a transient failure (`40001`, `40P01`, a lost connection): abort without advancing.

  As tagged errors, each `catchTags` branch is checked, and only the transient ones are retried: `Effect.retry({ schedule, while: isTransient })` (judgment).
- **Connectors.** The SDK wants typed errors (`docs/architecture.md`, Connectors). Consent expired, rate-limited and provider-unavailable become values that the worker's scheduling and the fetch budget match on.
- **Translation.** It happens once per transport: a Hono error handler maps tagged errors to status codes, and sync-core maps HTTP statuses (`409 client_too_old`, `410`) back to tagged errors.

### 3.2 Services and layers

- **ADR 0009's ports.** `SqlDriver`, `Transport`, `Auth` and `Clock` are already a services design. As `Context.Service` classes:
  - the iPhone's JS thread and the web leader worker each provide a layer;
  - the simulator provides test layers;
  - `Clock` is Effect's own, which `TestClock` replaces (H for the API, judgment for the fit).
- **The server's dependencies** become services: the Postgres pool or Drizzle, pg-boss, the provider connector, APNs, Web Push, config, and the KEK in the worker only.
- **The KEK rule.** A `ManagedRuntime` only runs effects whose requirements its layer provides. If the API's layer has no `Kek`, API code that needs the KEK fails to typecheck. "Only the worker holds the KEK" (`docs/architecture.md`, Server) becomes a compile-time rule (judgment).
- **What the checker catches.** With `@effect/tsgo`, `missingEffectContext` and `missingLayerContext` report unmet requirements, and `leakingRequirements` reports services leaking through an interface.

### 3.3 Effect Schema

- **Money.** `Schema.Int` checks `Number.isSafeInteger` and maps to a JSON Schema integer bounded by the safe range ([`Schema.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/Schema.ts)). That is ADR 0017's rule. `Schema.Int.pipe(Schema.brand("MinorUnits"))` gives the branded type (H). The brand is type-only: "It does not change the schema's runtime AST or add runtime checks."
- **Other codecs.** The same pattern covers ISO 4217 codes, `YYYY-MM-DD` civil dates, `YYYY-MM` budget months and 64-bit IDs as strings. Named, versioned mutations become `Schema.TaggedStruct("categorize@1", …)` members of one `Schema.Union` (measured: the probe in the appendix bundles).
- **One schema per boundary.** The same schemas decode push bodies, pull pages, provider payloads, import rows and golden vectors, and encode the glance.
- **Exports.** `toStandardSchemaV1` serves libraries that accept Standard Schema. `toJsonSchemaDocument` emits JSON Schema, for example for the watch contract (both stable).
- **Rows.** Drizzle's `effect-schema` can derive row schemas from tables ([Drizzle](https://orm.drizzle.team/docs/effect-schema)), on its 1.0 RC.
- **Generated tests.** `Arbitrary` derives test data from the same schemas, with shrinking and replay (unstable).

### 3.4 Concurrency

All of these primitives are stable (H for the APIs, judgment for the fit):

- **Sync-core.**
  - Push "within seconds" with retries: `Schedule.exponential` plus `Schedule.jittered`, and `Effect.timeout`.
  - One pull at a time, restarted by a newer poke: a one-permit `Semaphore` and fiber interruption.
  - Pokes through a `Queue` read as a `Stream`.
- **Leader handover** (`docs/architecture.md`, Client mutations and sync, step 7). The old leader finishes its transaction, closes its databases and steps down. As `Scope` finalizers, that cleanup runs on interruption or failure too.
- **API.**
  - Waiting on a pg-boss job's result becomes an Effect with a timeout.
  - The SSE `live` stream and the NDJSON bootstrap are `Stream`s handed to Hono through `Stream.toReadableStream`. A disconnect interrupts them.
- **Worker.**
  - Alert evaluation runs at most once a minute per plan, and "glance changed" hints are coalesced to two or three an hour: both are stream throttling.
  - Refresh slots use `Schedule`; per-account work is bounded with `Effect.forEach(…, { concurrency })`.

### 3.5 Testing

- **Virtual time.** `@effect/vitest`'s `it.effect` runs on `TestClock` (stable) ([Effect's testing doc](https://github.com/Effect-TS/effect/blob/effect%404.1.2/ai-docs/src/09_testing/10_effect-tests.ts)). Without real waiting, tests cover:
  - the 24-hour stale-edit rule;
  - consent alerts at T−14, T−3 and T−0;
  - refresh slots, backoff, and push within seconds.
- **Seeds and faults.** `Random.withSeed` (stable) gives the simulator reproducible runs (H). Test layers for `Transport` and `SqlDriver` inject the faults SP-09 lists: drops, duplicates, reordering, offline windows, kills between commit and ack.
- **Generators.** `it.effect.prop` takes Schema-derived arbitraries (unstable). Effect's `Arbitrary` re-implements parts of fast-check v4.9.0's runner, but has no model-based command runner or race scheduler. fast-check 4.10.2 stays for SP-09's model-based runs, as `factory-v2.md` plans (H for the APIs, judgment for the split).
- **Versions.** `@effect/vitest` 4.1.2 needs Vitest 5; Vitest 5.1.0 is current.

## 4. Impact on the decisions

### 4.1 ADR 0007: Hono, Drizzle and pg-boss

**Hono.** Three ways to combine them (H for the APIs):

- **(a) Keep Hono** and call `runtime.runPromise` in its handlers, with one `ManagedRuntime` per process.
  - Effect's own docs ship this pattern as "Using ManagedRuntime with Hono" ([`10_managed-runtime.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/ai-docs/src/04_integration/10_managed-runtime.ts)).
  - `ManagedRuntime` is stable. oidc-provider and SimpleWebAuthn don't change.
- **(b) Mount Effect routes in Hono.**
  - `HttpRouter.toWebHandler` turns an Effect router into a `(Request) => Promise<Response>` handler ([`HttpRouter.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/http/HttpRouter.ts)).
  - Hono 5 removed `app.mount()`; its `hono/mount` middleware takes such a handler under `app.all('/prefix/*', …)` ([Hono](https://hono.dev/docs/middleware/builtin/mount), [v5 migration](https://hono.dev/docs/guides/migrating-to-v5)).
  - Both are unstable on Effect's side.
- **(c) Replace Hono with Effect's `NodeHttpServer` and `HttpApi`, as T3 Code does.**
  - It gives one contract for handlers, typed clients and OpenAPI.
  - It amends ADR 0007 and puts the API on unstable modules.
  - `NodeHttpServer` hands its options to Node's `server.listen`, so a Unix socket path works; socket activation wasn't tested ([`NodeHttpServer.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/platform/node/src/NodeHttpServer.ts)).

Recommendation: (a). (c) is worth another look when HttpApi turns stable, or when the watch actions endpoint needs its OpenAPI document (judgment).

**Drizzle.**

- **(a) Drizzle's node-postgres driver behind an Effect `Db` service.** `Effect.tryPromise` maps driver errors to tagged errors, and transactions keep Drizzle's callback API.
- **(b) Drizzle's own Effect driver.** Drizzle 1.0.0-rc.5 (2026-10-21; `latest` is still 0.45.4) ships it, `drizzle-orm/effect-postgres` ([Drizzle](https://orm.drizzle.team/docs/connect-effect-postgres)), with:
  - queries that are Effects;
  - Schema-based errors (`EffectDrizzleQueryError`, `EffectTransactionRollbackError`);
  - an Effect migrator and transaction options;
  - peers `effect >=4.0.0` and `@effect/sql-pg >=4.0.0` (H).

  `@effect/sql-pg` 4.x is unstable. Its own driver speaks the PostgreSQL wire protocol, with no `pg` dependency. It takes a Unix socket path (a host starting with `/` becomes `<host>/.s.PGSQL.<port>`, as in libpq), answers MD5 and SCRAM-SHA-256 challenges, and has `LISTEN`/`NOTIFY` helpers ([`PgClient.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/sql/pg/src/PgClient.ts)). Peer authentication over the socket should need no client support (inference, L).
- **Docs lag.** Drizzle's page still uses `Context.Tag` and `pg`'s type parsers. Effect 4.1.2's `Context` exports only `Service` and `Reference`, and `@effect/sql-pg` takes its own `types` registry (measured).
- **Effect 3 integration.** `@effect/sql-drizzle` 0.51.0 stops below Drizzle 0.50, so it can't serve Drizzle 1.0.

Recommendation: (a) now. Try (b) in SP-04 once Drizzle 1.0 and `effect/sql` are both stable; the change is a driver import plus a layer (judgment). ADR 0007's reviewed SQL migrations run the same under either.

**pg-boss 12.38.1.**

- **No integration.** A `Jobs` service wraps `send`, `work` and `fetch`.
- **Possible replacements.** Effect 4 has a SQL-backed `PersistedQueue` (`layerStoreSql`), durable workflows and `ClusterCron`, all unstable ([`PersistedQueue.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/persistence/PersistedQueue.ts)). Nothing argues for replacing pg-boss.
- **Two drivers.** With option (b) above, the worker would hold two PostgreSQL clients, since pg-boss uses `pg` (inference, M).

**Consequence to record if #21 adopts part of Effect.** Server code uses Effect's stable core; Hono, Drizzle and pg-boss sit behind services; production code avoids unstable modules.

### 4.2 ADRs 0008 and 0009: the sync engine

- **ADR 0008.** The protocol doesn't change. Effect changes the implementation: typed outcomes and retries limited to transient failures (§3.1).
- **ADR 0009, ports.** They become services (§3.2).
- **ADR 0009, `SqlDriver` adapters.** Both stay hand-written, because Effect's SQLite clients wrap other engines (H):
  - `@effect/sql-sqlite-react-native` 4.1.2 wraps op-sqlite at `>=17.1.2 <19.0.0`;
  - `@effect/sql-sqlite-wasm` wraps `@effect/wa-sqlite`, not the official SQLite WASM build on `opfs-sahpool`;
  - Effect's expo-sqlite request ([#4544](https://github.com/Effect-TS/effect/issues/4544)) was closed as stale.

  T3 Code wrote its own adapters too (§2.1, §2.6). One more reason to own them: a bug in which "a statement outside a transaction can run inside another fiber's transaction (lost writes, dirty reads)" was fixed in Effect's SQLite clients on 2026-10-08 ([#8920](https://github.com/Effect-TS/effect/issues/8920)). SP-08 and SP-09 test the adapters Shoestring owns (judgment).
- **Web leader.** Effect runs in the leader worker like any script. The leader's database handles and its Web Lock become scoped resources (judgment).
- **SP-09.** The simulator gains `TestClock`, seeded `Random` and fault layers (§3.5).

### 4.3 `packages/domain` on three runtimes

- **Pure core.** The budget engine (safe to spend, daily budget, pace, rollover, shares, limit states), GlanceBuilder and the mutators' rules stay pure functions, with no Effect types in their signatures. `sync-engine.md`'s testing strategy asks for that: keep `apply(op, state) → state | rejection` pure. Effect code calls them unchanged, and so do golden vectors (judgment).
- **Effect at the edges.** Effect Schema defines the package's codecs (§3.3). Where a mutator reads or writes rows, the shell around it does: sync-core on devices, the push handler on the server.
- **Time.** Temporal stays. Effect's `DateTime` has its own time-zone code over `Intl` with a Temporal-like `compatible` default ([`DateTime.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/DateTime.ts)). It broke on Hermes Android, whose `Intl` drops a `formatToParts` part ([#8689](https://github.com/Effect-TS/effect/issues/8689)). Two time models in one package would be worse than one (judgment).
- **Runtimes.**
  - Node 26: nothing to note.
  - The web: tree-shaken by Rolldown.
  - Hermes: needs deep imports (§5.2).

  Once sync-core uses Effect, Schema in the domain package adds little to the clients' bundles (§5.2).

## 5. Costs

### 5.1 Learning curve

- **Effect's estimate.** "Following this spine takes most developers a few focused days" ([onboarding](https://effect.website/docs/v4/onboarding.md)).
- **What comes after.** The spine covers the `Effect` type, generators, errors and basic concurrency. Then come layers and scopes, fibers and interruption, `Cause` and defects, Schema's decoded and encoded sides, and a large API: T3 Code's server imports 149 distinct Effect paths (measured).
- **Guard rails.** T3 Code keeps the code idiomatic with tooling and process (§2.1, §2.8, §2.9):
  - 86 diagnostics set to error and two custom lint rules;
  - a 163-line services guide and a review check;
  - a vendored copy of Effect.
- **For Shoestring** (judgment):
  - Agents write better Effect with the source at hand and the diagnostics on.
  - The maintainer still has to read and review it.
  - Self-hosters who contribute, a connector for instance, meet a higher bar than plain TypeScript.
  - Keeping Effect to two packages, plus Schema in a third, bounds that cost.

### 5.2 Bundle size on the web and on Hermes

Measured with the probes in the appendix, built as minified ESM for the browser:

- **Web:** Rolldown 1.3.0. Vite 8.3.6 depends on Rolldown, and Vite+ core 1.5.0 vendors it.
- **Hermes:** a Metro proxy. Metro bundles every module it reaches and ignores `sideEffects`; Expo's cross-module tree shaking is still "very experimental" behind `EXPO_UNSTABLE_TREE_SHAKING=1` ([Expo](https://docs.expo.dev/guides/tree-shaking/)). So the proxy is esbuild 0.28.2 with `--tree-shaking=false --ignore-annotations`.

kB are thousands of bytes.

| Probe | Web, min / gzip | Hermes proxy, min |
|---|---|---|
| Minimal: `Effect.runPromise(Effect.succeed(42))` | 22.3 / 7.7 | 165.4 |
| The same, imported from `"effect"` | 22.3 / 7.7 | 871.1 |
| Domain codecs, Effect Schema | 80.4 / 25.5 | 475.9 |
| Domain codecs, Zod 4.6.7 | 92.3 / 25.0 | 454.4 |
| Domain codecs, Zod Mini | 19.8 / 6.4 | 427.7 |
| Domain codecs, Valibot 1.5.0 | 6.1 / 2.1 | 79.8 |
| sync-core probe | 112.5 / 36.0 | 559.3 |
| sync-core probe plus `effect/http`'s client | 143.4 / 45.7 | 706.1 |
| sync-core probe plus atoms | 146.6 / 46.6 | 641.0 |

**Web.**

- Effect's codecs cost about what Zod's do. The concurrency and service pieces add about 10 kB gzipped on top.
- My minimal program is 7.7 kB gzipped, against Effect's published 7.1 kB.

**Hermes.**

- The Schema module alone pulls most of the runtime, so the full sync probe adds little to it: 559 kB against 476 kB.
- Zod is about as large there, because every Zod entry re-exports all 64 locales: 264 kB of the Mini probe.
- A plain stack on Valibot saves about 0.48 MB.

**Import style.**

- Deep imports (`import * as Effect from "effect/Effect"`) are a must. Under Metro, the root import pulls 871 kB for a minimal program.
- esbuild doesn't shake the root import either. Tree-shaken, the minimal program is 90.3 kB against 26.1 kB from deep paths, and a small Schema decoding program 358.0 kB against 84.1 kB. Rolldown shakes both alike.

**Not measured.** Hermes bytecode size and startup time. SP-18's first frame under 400 ms is the gate, and these bytes reach the device only in the app binary or an over-the-air update.

### 5.3 Hermes and Expo gaps

All issues below are closed; the dates show how recent the fixes are (H).

- **Runtime support.** Effect's installation page names "Node.js 18 or newer, Deno, and Bun", and nothing about React Native ([installation](https://effect.website/docs/v4/getting-started/installation.md)).
- **SubtleCrypto.** Hermes has none, so the app provides crypto services itself (§2.6).
- **`FinalizationRegistry`.** [#8750](https://github.com/Effect-TS/effect/issues/8750) reports that Hermes has none.
  - `HttpClient`'s fallback kept every unread response alive. Fixed on 2026-10-06.
  - On such runtimes `Atom.family` caches every atom it creates, with no weak references ([`Atom.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/reactivity/Atom.ts)).
- **`Intl`.** `DateTime` broke on Hermes Android ([#8689](https://github.com/Effect-TS/effect/issues/8689), fixed on 2026-10-03).
- **Beta-era problems, fixed.**
  - Schema decoding broke under `babel-preset-expo` in Jest ([#6578](https://github.com/Effect-TS/effect/issues/6578), closed on 2026-07-27).
  - "Can't use Effect with React Native" on a 4.0 beta ([#6252](https://github.com/Effect-TS/effect/issues/6252), closed on 2026-07-23).
- **op-sqlite.** The React Native SQLite client needed a peer range bump for React Native 0.88 ([#9868](https://github.com/Effect-TS/effect/issues/9868), October 2026).

### 5.4 Interop

| Tool | With Effect | Note |
|---|---|---|
| Hono 5.0.0-rc.1 (`next`; `latest` 4.13.17) on `@hono/node-server` 2.1.6 | `ManagedRuntime` in handlers; `Stream.toReadableStream` for SSE and NDJSON | §4.1 |
| Drizzle 1.0.0-rc.5 | node-postgres behind a service now; `drizzle-orm/effect-postgres` later | §4.1 |
| pg-boss 12.38.1 | Wrapped in a service | §4.1 |
| oidc-provider, SimpleWebAuthn | Unchanged behind Hono; Promise calls wrapped where Effect code needs them | Judgment |
| Temporal | Kept; Effect's `DateTime` unused in `packages/domain` | §4.3 |
| Vitest 5.1.0, fast-check 4.10.2 | `@effect/vitest` 4.1.2 needs Vitest 5; fast-check stays for model-based runs | §3.5 |
| TypeScript 7.0.2, Vite+ | `effect-tsgo` patches `tsc`, tsrs, Oxlint and `vp`, and runs them without patching too ([README](https://github.com/Effect-TS/tsgo)) | Gate choice belongs to #16 |

Each Promise library meets Effect at a seam: an adapter written once per library, plus code reviewers who know both styles (judgment).

### 5.5 Stability and upgrades

- **Core.** It is stable and has long-term support (§1.2).
- **Unstable modules.** They break in minor releases (§1.3).
- **Lockstep versions.** `effect` and every `@effect/*` package move together. The `duplicatePackage` diagnostic catches two copies in one program.
- **Tooling.** `@effect/tsgo` is 0.x and released on 2026-11-07, for TypeScript 7.0.2.
- **Drizzle's Effect drivers.** They sit on a release candidate.

## Recommendation

| Option | What it means | Gains | Costs |
|---|---|---|---|
| **All** (T3 Code's model) | Effect from the listener down: HttpApi on `NodeHttpServer` instead of Hono, `effect/sql` with Drizzle's Effect driver, Effect-typed mutators, atoms in both apps | One contract for handlers, clients and OpenAPI; one style everywhere | Amends ADR 0007; puts the API, the database layer and client state on unstable modules; largest Hermes footprint (0.6 to 0.7 MB in the proxy); every contributor needs Effect |
| **Part** (recommended) | Stable core in `apps/server` and `packages/sync-core`; Schema codecs in `packages/domain`; Hono, Drizzle and pg-boss behind services | Typed failures, services, structured concurrency and virtual time where the hard I/O is | Learning curve in two packages; a seam at every Promise library; about 0.56 MB on Hermes |
| **None** | Plain TypeScript with discriminated-union results, Valibot or Zod, injected clocks, fast-check | Lowest bar for contributors; on Hermes, 0.48 MB less with Valibot | Retries, timeouts, cancellation, cleanup and wiring hand-rolled in sync-core and the worker |

**Why not none** (judgment):

- The sync engine and the worker are where Shoestring does its hardest I/O: offline queues, pokes, retries, leader handover, consent and refresh timers. Effect's stable core covers exactly what would otherwise be hand-written there.
- ADR 0009's ports are already a services design.
- The core's support runs to at least September 2029.

**Why not all** (judgment):

- It trades ADR 0007's Hono for unstable modules.
- It doesn't remove the two SQLite adapters.
- It puts Effect in front of every contributor.
- T3 Code accepted those costs with guard rails a single maintainer would have to keep up.

**If #21 adopts part of it, these rules follow:**

1. **Scope.** Effect goes in `apps/server` and `packages/sync-core`; Schema only in `packages/domain`. UI code reaches sync-core through a small boundary, and atoms wait for the client-state design.
2. **Stable APIs only** in production code: `unstableApiUsage` and `experimentalApiUsage` at error outside tests.
3. **Deep namespace imports** from subpaths, never from `"effect"`.
4. **Runtimes at boundaries.** One `ManagedRuntime` per process (API, worker, CLI) and per client runtime (the iPhone's JS thread, the web leader worker); none inside services.
5. **Errors** are `Schema.TaggedError` classes with bounded attributes and no amounts, payloads or identifiers in messages. T3 Code's error rules transfer to financial data (judgment).
6. **Tests.** `it.effect` on `TestClock`, never sleeps; fast-check drives SP-09.
7. **Versions.** One catalog entry moves `effect` and `@effect/*` together.

## Open questions and spikes

- **E1, inside SP-18: Effect in the Expo app.**
  - Metro bundle bytes with deep imports, with and without `EXPO_UNSTABLE_TREE_SHAKING`.
  - Time to first frame on Hermes.
  - The no-`FinalizationRegistry` and `Intl` paths on iOS.
- **E2, inside SP-09: the simulator on Effect.** `TestClock`, `Random.withSeed` and fault layers, with fast-check generating command sequences. Compare its size with a Promise-based harness.
- **E3, inside SP-04: Drizzle's `effect-postgres` driver against node-postgres behind a service.** Check:
  - peer authentication on the Unix socket;
  - `bigint`, `date` and `timestamptz` codecs;
  - one savepoint per mutation in a push.
- **For #21:**
  - Is the connector SDK's public interface Effect or Promises?
  - Do the apps read sync-core through atoms or `useSyncExternalStore`?
  - How do agents get Effect's source: a `git subtree` (Effect's advice), an ignored clone, or the sources and `ai-docs/` inside `node_modules/effect`?
  - Should the watch actions endpoint's OpenAPI document come from HttpApi later?

**Could not verify**

- Metro's real output. The esbuild proxy omits Metro's module wrappers, and neither Hermes bytecode size nor startup was measured.
- The program behind Effect's 7.1 kB figure, and its runtime benchmarks.
- Whether Expo's experimental tree shaking handles Effect.
- Whether `@effect/sql-pg`'s codecs cover every column type ADR 0007's schema needs.
- Socket activation through `NodeHttpServer`'s typed options; not needed while Hono stays.
- T3 Code's apps at runtime: only their source was read.

## Sources

**Repo:**

- ADRs [0007](../adr/0007-typescript-server-on-node-with-hono-drizzle-and-postgresql.md), [0008](../adr/0008-server-authoritative-sync-with-named-mutations.md), [0009](../adr/0009-build-the-sync-engine-in-typescript.md) and [0017](../adr/0017-money-in-minor-units-time-in-the-household-zone.md);
- [`docs/architecture.md`](../architecture.md), [sync-engine.md](sync-engine.md) and [factory-v2.md](factory-v2.md);
- tickets [#6](https://github.com/fveauvy/shoestring/issues/6), [#7](https://github.com/fveauvy/shoestring/issues/7), [#8](https://github.com/fveauvy/shoestring/issues/8), [#16](https://github.com/fveauvy/shoestring/issues/16) and [#21](https://github.com/fveauvy/shoestring/issues/21).

**Effect, releases and docs:**

- release posts: [4.0](https://effect.website/blog/releases/effect/40), [4.0 RC](https://effect.website/blog/releases/effect/40-rc) and [4.1](https://effect.website/blog/releases/effect/41);
- docs: [onboarding](https://effect.website/docs/v4/onboarding.md), [installation](https://effect.website/docs/v4/getting-started/installation.md), [two error types](https://effect.website/docs/v4/error-management/two-error-types.md) and [getting started](https://effect.website/docs/v4/getting-started.md);
- [vendoring Effect for agents](https://effect.website/blog/the-one-weird-git-trick-that-makes-coding-agents-more-effect-ive/);
- [GitHub releases](https://github.com/Effect-TS/effect/releases).

**Effect, source at `effect@4.1.2`:**

- core: [`Schema.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/Schema.ts), [`DateTime.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/DateTime.ts), [`Random.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/Random.ts) and [`Arbitrary.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/Arbitrary.ts);
- subpath modules: [`http/HttpRouter.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/http/HttpRouter.ts), [`sql/SqlClient.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/sql/SqlClient.ts), [`persistence/PersistedQueue.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/persistence/PersistedQueue.ts) and [`reactivity/Atom.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/effect/src/reactivity/Atom.ts);
- satellite packages: [`sql/pg`'s `PgClient.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/sql/pg/src/PgClient.ts), [`platform/node`'s `NodeHttpServer.ts`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/platform/node/src/NodeHttpServer.ts) and [`@effect/vitest`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/packages/vitest/src/index.ts);
- agent docs: [ManagedRuntime with Hono](https://github.com/Effect-TS/effect/blob/effect%404.1.2/ai-docs/src/04_integration/10_managed-runtime.ts), [tests](https://github.com/Effect-TS/effect/blob/effect%404.1.2/ai-docs/src/09_testing/10_effect-tests.ts) and [`LLMS.md`](https://github.com/Effect-TS/effect/blob/effect%404.1.2/LLMS.md);
- the published [tarball](https://registry.npmjs.org/effect/-/effect-4.1.2.tgz), where the `@stability` tags were counted.

**Effect, issues and tooling:**

- issues [#4544](https://github.com/Effect-TS/effect/issues/4544), [#6252](https://github.com/Effect-TS/effect/issues/6252), [#6578](https://github.com/Effect-TS/effect/issues/6578), [#8689](https://github.com/Effect-TS/effect/issues/8689), [#8750](https://github.com/Effect-TS/effect/issues/8750), [#8920](https://github.com/Effect-TS/effect/issues/8920) and [#9868](https://github.com/Effect-TS/effect/issues/9868);
- [Effect-TS/tsgo](https://github.com/Effect-TS/tsgo) and its [npm record](https://registry.npmjs.org/@effect/tsgo).

**npm, read on 2026-11-08:**

- Effect: [`effect`](https://registry.npmjs.org/effect), [`@effect/sql-pg`](https://registry.npmjs.org/@effect/sql-pg), [`@effect/sql-sqlite-react-native`](https://registry.npmjs.org/@effect/sql-sqlite-react-native), [`@effect/sql-sqlite-wasm`](https://registry.npmjs.org/@effect/sql-sqlite-wasm), [`@effect/atom-react`](https://registry.npmjs.org/@effect/atom-react), [`@effect/vitest`](https://registry.npmjs.org/@effect/vitest), [`@effect/platform-node`](https://registry.npmjs.org/@effect/platform-node) and [`@effect/sql-drizzle`](https://registry.npmjs.org/@effect/sql-drizzle);
- Shoestring's other tools: [`drizzle-orm`](https://registry.npmjs.org/drizzle-orm), [`hono`](https://registry.npmjs.org/hono), [`@hono/node-server`](https://registry.npmjs.org/@hono/node-server), [`pg-boss`](https://registry.npmjs.org/pg-boss), [`vitest`](https://registry.npmjs.org/vitest), [`fast-check`](https://registry.npmjs.org/fast-check) and [`typescript`](https://registry.npmjs.org/typescript);
- comparison and build tools: [`zod`](https://registry.npmjs.org/zod), [`valibot`](https://registry.npmjs.org/valibot), [`rolldown`](https://registry.npmjs.org/rolldown), [`esbuild`](https://registry.npmjs.org/esbuild), [`vite`](https://registry.npmjs.org/vite) and [`@voidzero-dev/vite-plus-core`](https://registry.npmjs.org/@voidzero-dev/vite-plus-core).

**T3 Code, at [`6417151`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a):**

- configuration: [`tsconfig.effect.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/tsconfig.effect.json), [`tsconfig.base.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/tsconfig.base.json), [`pnpm-workspace.yaml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/pnpm-workspace.yaml), [`package.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/package.json) and [`vite.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/vite.config.ts);
- guidance: [`AGENTS.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/AGENTS.md), [`docs/internals/effect-services.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/internals/effect-services.md) and [`scripts/lib/reference-repos.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/lib/reference-repos.ts);
- server: [`server.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/server.ts), [`Sqlite.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/persistence/Sqlite.ts), [`Scheduler.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/scheduling/Scheduler.ts) and [its test](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/server/src/scheduling/Scheduler.test.ts);
- contracts: [`environmentHttp.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/contracts/src/environmentHttp.ts) and [`rpc.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/contracts/src/rpc.ts);
- `client-runtime`: [its README](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/client-runtime/README.md) and [`persistence.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/packages/client-runtime/src/platform/persistence.ts);
- Expo app: [`package.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/package.json), [`storage.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/src/connection/storage.ts), [`mobile-database.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/src/persistence/mobile-database.ts) and [`dpop.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/src/features/cloud/dpop.ts).

**Drizzle, Hono and Expo:**

- Drizzle: [Effect Postgres](https://orm.drizzle.team/docs/connect-effect-postgres), [effect-schema](https://orm.drizzle.team/docs/effect-schema), [v1.0.0-rc.5 release notes](https://orm.drizzle.team/docs/latest-releases/drizzle-orm-v1rc5), and the [1.0.0-rc.5 record](https://registry.npmjs.org/drizzle-orm/1.0.0-rc.5) with its peer dependencies;
- Hono: [Mount Middleware](https://hono.dev/docs/middleware/builtin/mount) and [Migrating to v5](https://hono.dev/docs/guides/migrating-to-v5);
- Expo: [tree shaking](https://docs.expo.dev/guides/tree-shaking/).

## Appendix: probe sources

Every probe printed or exported its values, so bundlers couldn't drop them. Built with:

```sh
rolldown <probe>.ts --format esm --platform browser --minify --file out.js
esbuild <probe>.ts --bundle --minify --format=esm --platform=browser --target=es2022 \
  --tree-shaking=false --ignore-annotations --outfile=out.js   # Hermes proxy
gzip -9 -c out.js | wc -c
```

The domain probe:

```ts
import * as Schema from "effect/Schema"
export const Currency = Schema.String.check(Schema.isPattern(/^[A-Z]{3}$/)).pipe(Schema.brand("Currency"))
export const MinorUnits = Schema.Int.pipe(Schema.brand("MinorUnits"))
export const Money = Schema.Struct({ amount: MinorUnits, currency: Currency })
export const CivilDate = Schema.String.check(Schema.isPattern(/^\d{4}-\d{2}-\d{2}$/)).pipe(Schema.brand("CivilDate"))
export const Categorize = Schema.TaggedStruct("categorize@1", { transactionId: Schema.String, categoryId: Schema.String, at: CivilDate })
export const MoveMoney = Schema.TaggedStruct("moveMoney@1", { from: Schema.String, to: Schema.String, amount: Money })
export const Mutation = Schema.Union([Categorize, MoveMoney])
export class MutationRejected extends Schema.TaggedError<MutationRejected>()("MutationRejected", { reason: Schema.String }) {}
const decode = Schema.decodeUnknownSync(Mutation)
console.log(decode(JSON.parse((globalThis as any).INPUT ?? "{}")), new MutationRejected({ reason: "x" }))
```

The Zod and Valibot probes declare the same schemas with `import * as z from "zod"`, `zod/mini` and `import * as v from "valibot"`.

The sync-core probe:

```ts
import * as Clock from "effect/Clock"
import * as Context from "effect/Context"
import * as Effect from "effect/Effect"
import * as Layer from "effect/Layer"
import * as Queue from "effect/Queue"
import * as Schedule from "effect/Schedule"
import * as Schema from "effect/Schema"
import * as Semaphore from "effect/Semaphore"
import * as Stream from "effect/Stream"
const PullResponse = Schema.Struct({ cursor: Schema.String, rows: Schema.Array(Schema.Struct({ id: Schema.String, v: Schema.Int })) })
class TransportError extends Schema.TaggedError<TransportError>()("TransportError", { status: Schema.Int }) {}
class Transport extends Context.Service<Transport, { readonly pull: (cursor: string) => Effect.Effect<unknown, TransportError> }>()("Transport") {}
class SqlDriver extends Context.Service<SqlDriver, { readonly exec: (sql: string) => Effect.Effect<void> }>()("SqlDriver") {}
const pullOnce = Effect.fn("pullOnce")(function* (cursor: string) {
  const transport = yield* Transport
  const db = yield* SqlDriver
  const raw = yield* transport.pull(cursor).pipe(Effect.retry(Schedule.exponential("200 millis").pipe(Schedule.jittered)), Effect.timeout("10 seconds"))
  const res = yield* Schema.decodeUnknownEffect(PullResponse)(raw)
  yield* db.exec(`-- apply ${res.rows.length} rows at ${yield* Clock.currentTimeMillis}`)
  return res.cursor
})
const program = Effect.gen(function* () {
  const pokes = yield* Queue.unbounded<string>()
  const lock = yield* Semaphore.make(1)
  yield* Stream.fromQueue(pokes).pipe(Stream.runForEach((c) => lock.withPermits(1)(pullOnce(c))), Effect.forkChild)
  yield* Queue.offer(pokes, "0")
})
const Live = Layer.mergeAll(
  Layer.succeed(Transport, Transport.of({ pull: () => Effect.succeed({ cursor: "1", rows: [] }) })),
  Layer.succeed(SqlDriver, SqlDriver.of({ exec: (s) => Effect.log(s) })),
)
Effect.runFork(program.pipe(Effect.provide(Live)))
```

The two extended probes import the sync-core probe and add more:

- the HTTP one makes a `GET` through `HttpClient`, provided by `FetchHttpClient.layer` from `effect/http`;
- the atoms one creates an `Atom.make` value in an `AtomRegistry.make()` registry from `effect/reactivity`.
