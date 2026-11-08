# Offline-first sync: Linear's engine, CRDTs and options for on-budget

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: Swift client side; HLC; UUIDv5 bank rows; restore semantics.

> **Status (2026-11-07).** The client-side recommendations below (GRDB, Swift SDKs, mutators shared in Swift) assume Swift clients. The owner has since chosen one UI codebase shared by iOS, macOS and web, with a native SwiftUI watch app ([ADR 0002](../adr/0002-one-ui-codebase-native-watch.md)); for the client side, see [cross-platform-ui-stack.md](cross-platform-ui-stack.md) and [web-client-sync.md](web-client-sync.md). The server-authoritative protocol design stays in force.

Researched 2026-11-07 for on-budget, a self-hosted open-source product: each instance (server + Postgres) belongs to one owner, who creates its households and adds their members, and there is no public sign-up ([ADR 0001](../adr/0001-self-hosted-single-owner-instances.md)). The server ingests bank data (transaction history only) through each member's own bank-data provider credentials, and a household is typically shared by two people. Revised the same day after a fact-check: Actual is pinned to master, the minimum viable protocol is corrected, and self-hosting costs are added. Choosing a bank aggregator is out of scope.

Confidence: **high** = read in a primary source (docs, code, first-party talk); **medium** = partial primary or strong secondary source; **low** = inference.

## Question

1. How does Linear's sync engine work? Bootstrap, local store, transactions and offline queue, sync IDs, conflicts, CRDT use, permissions, migrations.
2. How does Actual Budget sync? Clock, Merkle trie, messages, server role, E2EE, limits.
3. Which sync options fit a Swift client and an owned server that also writes data shared by two users?
4. Which conflict semantics fit budget data, and where does per-field last-writer-wins (LWW) lose money-relevant intent?
5. Build or adopt? Which data flow, which minimum protocol, and how to test it?

## TL;DR

**Recommendation: build a small server-authoritative sync engine on the Linear/Replicache model.** SQLite on every client (GRDB in this note's Swift plan), Postgres on the server, named domain mutations in a client outbox, and one server-assigned monotonic sync counter per instance. Clients keep the server's rows apart from their pending mutations and replay those on top. No CRDT for the ledger, no CloudKit. Before committing, run Spike A, a 2-day self-hosted PowerSync spike, which now decides between building and adopting. Confidence: medium-high for the server-authoritative design; medium for build vs PowerSync.

- **Linear is not a CRDT** (high). The server orders all transactions, every change gets a counter sync ID, and conflicts are last-writer-wins (LWW): "the second person wins". A CRDT is used only for rich text (issue descriptions, documents) since December 2023; the library is Yjs + ProseMirror (medium).
- **Actual Budget is CRDT-style** (high). Per-cell LWW messages stamped with a hybrid logical clock (HLC), a Merkle trie to detect divergence, and a relay server that never reads the data (optionally end-to-end encrypted, E2EE). As a result, bank sync runs only on clients while the app is open, and the docs discourage simultaneous editing.
- **on-budget's server must write** (bank ingestion), **read** (rules, alerts, push) **and enforce household permissions** (high). That rules out E2EE of budget data and calls for a server-authoritative design.
- **CloudKit and SQLiteData don't fit** (high). A server can't write into users' private or shared databases (server-to-server keys reach only the public database), and SQLiteData sharing skips records with more than one foreign key.
- **PowerSync is the strongest option to adopt** (November 2026): native Swift SDK, Postgres as source, service relicensed to Apache-2.0 on 2026-11-03, GRDB integration still alpha. You still write the write API and the conflict rules. Self-hosted, it adds a service and its bucket storage to every instance, shares usage metrics unless that is turned off, and has no dashboard. Facts: high; fit: medium.
- **Conflict semantics** (high). Per-field LWW (HLC order, user > rule > import precedence) for annotations; server-validated operations for money moves, goal contributions, split sets, transaction merges and deletes.
- **The first draft of the protocol had bugs**, found in review and fixed below (high). It dropped server values for fields with pending edits although its LWW isn't arrival-ordered, so a client could keep a value that had lost on the server. It also broke its own `change_log` key, left pulls without one consistent snapshot, let one failing mutation abort a whole push, assumed one writer per device, which browser tabs break, and copied a clock check that stops sync on a device whose clock is off. That is the price of building.
- **Testing** (medium-high). A seeded deterministic simulator of N clients plus the server, with property-based convergence and money invariants in CI.

## 1. How Linear's sync engine works

**Sources.** First-party: the devtools.fm #61 transcript with Tuomas Artman, 2023-07-31 [lin-dt61]; the "Scaling the Linear Sync Engine" talk, 2023-06-29, video only [lin-scaling]; the 2023-12-06 changelog [lin-collab]. Secondary: wzhudev's reverse engineering [re-linear], which Artman called "correct and more complete than what Linear publishes internally" [hn-artman], and Linear's client code captured in that repository [re-code]. YouTube transcripts couldn't be fetched, so talk content comes via the podcast transcript and the secondary write-up.

```
client (web/Electron)                                server
┌───────────────────────────────┐  GraphQL mutation   ┌──────────────────────────┐
│ object pool (MobX models,     │ ──────────────────▶ │ API: business logic,      │
│  lazily hydrated graph)       │ ◀── {lastSyncId} ── │ writes DB, emits sync     │
│ IndexedDB: confirmed rows only│                     │ actions (+ side effects)  │
│  + __transactions (outbox)    │ ◀═ WebSocket deltas │ sync server → all clients │
└───────────────────────────────┘   (sync actions)    │ bootstrap: streamed JSON  │
                                                      │ from a Mongo cache        │
                                                      └──────────────────────────┘
```

**Bootstrapping.**

- **First login** (high) [lin-dt61]:
  - The client makes a "bootstrap call … give me all the objects … as a JSON blob". The server filters it by access control.
  - The payload can reach "a hundred megabytes" unzipped.
  - Linear moved the bootstrap from GraphQL to a streaming REST endpoint.
  - Serialized models are cached in MongoDB ("literally a cache").
- **Bootstrap types** (medium) [re-linear]:
  - Three types: full, local (load from IndexedDB, then catch up on deltas) and partial.
  - Each model has a load strategy: `instant`, `lazy`, `partial`, `explicitlyRequested` or `local`.
  - In late 2024 the bootstrap was split into cacheable requests, with sync groups fetched first.
- **On-demand loading** (high) [lin-dt61]:
  - Comments and history, and later issues and attachments, are left out of the bootstrap.
  - A central loader de-duplicates requests and fetches them in batches when needed.
  - A lazy-loaded snapshot may be older than what the client already has, so "you don't store it on disk if you have it already on disk".

**Local store and object graph.**

- **IndexedDB layout** (medium) [re-linear]:
  - One database per user and workspace, with a table per model.
  - A `_meta` table holds `lastSyncId`, `firstSyncId` and the sync groups.
  - `__transactions` holds the outbox.
- **In memory** (high) [lin-dt61], [re-linear]:
  - Models sit in a pool keyed by UUID.
  - Decorators mount each object into a graph: an issue with a `teamId` joins `team.issues`.
  - MobX re-renders views, and models are hydrated lazily.
  - Property names are the same in the database, the API, GraphQL and the client.

**Writes, offline queue and rejection.**

- **Saving a change** (medium) [re-linear]:
  1. `issue.title = …; issue.save()` updates the in-memory model at once.
  2. An `UpdateTransaction` is created and batched per microtask.
  3. It is persisted to `__transactions`.
  4. Batches go to the server as GraphQL mutations.
- **Completion** (high):
  - The mutation returns a `lastSyncId`.
  - The client "starts waiting on sync" until the delta packets reach that ID [lin-dt61]. Only then is the transaction complete.
  - Until then it waits in `completedButUnsyncedTransactions` [re-linear].
- **Rejection** (medium to high) [re-linear], [lin-dt61]:
  - IndexedDB model tables are only written from server deltas.
  - A rejected transaction rolls back in memory and shows a generic toast with the server's error message.
- **Replay** (medium) [re-linear]:
  - Persisted transactions are replayed after a restart.
  - Not all of them are idempotent, so a replay can fail with "You can't delete a model that doesn't exist".
- **Unsafe offline operations**:
  - Some operations are deliberately unsafe offline: "creating a team in offline … seems like a bad idea".
  - Features can detect that case [lin-dt61] (high).
  - Undo and redo are built on inverse transactions [re-linear] (medium).

**Sync IDs and delta packets.**

- **Sync IDs** [lin-dt61], [re-linear]:
  - "Every packet … has an id. It's just a counting number upwards" (high).
  - On connect, the client sends the last ID it saw and receives everything newer (high).
  - The counter is global across the database, so the IDs inside one packet aren't contiguous (medium).
- **Delta packets** (medium) [re-linear], [lin-dt61]:
  - Action types: I insert, U update, A archive, D delete, C cover, G/S sync-group change, V unarchive.
  - A delta can carry server side effects, such as `IssueHistory` and `Activity` records.
  - Deltas go to every client, including the author.
  - Clients apply one packet at a time, under a lock.

**Conflict resolution.**

- Artman: "no CRDTs … they're atomic operations … literally the merge is like, last one wins … the second person wins" [lin-dt61] (high).
- The server sets a single order, which is closer to operational transformation (OT) than to a CRDT. On the client, pending `UpdateTransaction`s are rebased onto incoming deltas [re-linear] (medium).
- Figma uses the same model [figma] (high):
  - the server is the authority;
  - LWW applies per property, in arrival order: the document keeps "the last value that was sent to the server";
  - to avoid flicker, the client discards incoming server values that conflict with its own unacknowledged edits. That is safe only under arrival order, where the pending edit is "the most recent change we know about in last-to-the-server order". on-budget's HLC-ordered LWW can't reuse the rule (see the client pull steps).

**Where Linear does use a CRDT.**

- **Rich text only** (high):
  - In 2023, descriptions were saved when the field lost focus and overwrote concurrent edits. Artman: "that's where we're putting it in place now" [lin-dt61].
  - On 2023-12-06 Linear announced that "all documents and issue descriptions … can now be edited by multiple people at the same time" [lin-collab].
- **Library** (medium) [re-code]:
  - The client code shows Yjs with ProseMirror (`ProseMirror-yjs-cursor`, `getXmlFragment("prosemirror")`).
  - The Yjs state is serialized into `DocumentContent.contentState`.
  - So the CRDT lives inside one opaque field of an otherwise server-authoritative model.

**Permissions: sync groups.** Access control uses sync groups: the user, their teams and their roles. A "sync groups changed" action tells the client to load more data, for example after the user is added to a private team [lin-dt61], [re-linear] (high).

**Schema migrations.** `ModelRegistry.__schemaHash` hashes all model metadata. A new hash bumps the IndexedDB schema version and triggers a migration or a full bootstrap [re-linear]. I found no first-party source for this (medium-low).

**Lessons for on-budget.**

1. The server orders writes. Clients hold confirmed state plus pending operations.
2. A mutation returns its sync ID, and the client waits until it has pulled that far.
3. Use LWW for scalar fields. Keep any CRDT inside one opaque field.
4. Permission scopes double as sync scopes.
5. The local database is a cache: wipe it and re-bootstrap when needed.
6. Make every replayed mutation idempotent. Linear doesn't fully do this.

## 2. How Actual Budget syncs

Read at `actualbudget/actual` master @`13ee015` (2026-11-07, MIT), not at a release: GitHub reports it diverged from the latest release, v26.11.0 (2026-11-02), 650 commits ahead and 29 behind [act-cmp]. Everything below also holds for v26.11.0 except the server's future-drift check, which landed after it [act-8905], [act-server-rel].

- **Messages** (high):
  - Every write is a message `{dataset (table), row (id), column, value}` plus a timestamp [act-proto].
  - `db.insert` and `db.update` emit one message per column.
  - A delete sets a `tombstone` column [act-db].
- **Clock** (high) [act-ts]:
  - A "Hybrid Unique Logical Clock" (HULC), based on Kulkarni et al.'s HLC [hlc].
  - Each timestamp is a 46-character sortable string: ISO milliseconds, a 16-bit counter and a 16-hex-digit node ID.
  - Maximum clock drift is 5 minutes: receiving a timestamp further ahead than that throws `ClockDriftError`, and a counter overflow throws too.
  - The whole clock is one 356-line file.
- **Conflicts** (high) [act-sync]:
  - Conflicts resolve per cell, by LWW.
  - If a cell already has a later timestamp, the incoming message is marked `old` and not applied. It still goes into the Merkle trie.
- **Merkle trie** (high) [act-merkle], [act-sync]:
  - A ternary trie keyed by minutes since the epoch (in base 3). Each node stores the XOR of the murmurhash of its timestamps.
  - `diff` returns the earliest minute where two tries differ, and the client re-syncs from that time.
  - The client gives up with `out-of-sync` after about 10 identical diffs or 100 passes.
- **Server** (high) [act-server]:
  - The server keeps one SQLite file per sync group, with a `messages_binary(timestamp PK, is_encrypted, content)` table and a `messages_merkles` table.
  - `sync(messages, since)` returns the newer messages and the server's trie.
  - On master only, it rejects timestamps more than 5 minutes ahead of its clock, a window that "Matches the client's Timestamp maxDrift; later messages break sync on other devices". One bad timestamp rolls back the whole batch ("Throwing rolls back the transaction"). This came with PR #8905, merged 2026-11-03, the day after v26.11.0 shipped [act-8905].
  - It never interprets budget data.
- **E2EE** (high):
  - Optional. Uses AES-256-GCM with a key derived by PBKDF2-SHA512 at 10,000 iterations [act-enc].
  - The envelope `timestamp` stays in clear [act-proto].
  - Bank-sync tokens aren't covered, and local data isn't encrypted [act-docs-sync].
- **Bank sync runs on the client** (high):
  - The client downloads transactions through provider proxies on the server: GoCardless, SimpleFIN, Pluggy, Akahu and Enable Banking.
  - It fuzzy-matches and de-duplicates them locally, then writes CRDT messages [act-bank].
  - "Syncing only happens while Actual is open".
  - GoCardless Bank Account Data is "not accepting new accounts" [act-docs-bank].
- **Schema skew** (high): messages for unknown tables or columns are deferred into `messages_pending` [act-sync].
- **Known limits.**
  - **History growth** (high) [act-docs-sync]:
    - The full message history is kept, so the file keeps growing.
    - "Reset sync" compacts it, but reverts unsynced changes on other devices.
    - The "Your data is out of sync" error is also fixed by a reset.
  - **Concurrent editing** (high):
    - On two people editing at once, the docs say it "should work unless the edits conflict. To be safe, avoid simultaneous usage" [act-docs-sync].
    - OpenID Connect (OIDC) multi-user mode adds per-file access with Basic and Admin roles [act-docs-mu].
  - **Lost budget moves**:
    - Budget moves read and rewrite absolute cell values. `transferCategory` reads both budgeted cells, then writes `from − amount` and `to + amount` through `setBudget` [act-budget].
    - So when two devices move money into the same category while offline, only one move survives.
    - Confidence: high for the code; the consequence is inferred, also high.
  - **Disabled property test**: a jsverify convergence property test exists but is marked `it.skip`, on master and in v26.11.0 [act-proptest] (high).
  - **Code size**: James Long's 2019 reference implementation had 132 lines of server code and 639 lines of client code [jlong]. Actual's client sync module is now 1,101 lines on master (1,049 in v26.11.0) [act-sync] (high).

**Lessons for on-budget** (high):

- HLC plus per-field LWW is cheap and is the right tool for annotation fields.
- A Merkle trie is only needed when there's no server-ordered log, and on-budget will have one.
- Actual can offer E2EE only because its server never reads the data.

## 3. Option landscape for a Swift client and an owned server (November 2026)

"Fit" is judged against five requirements. **R1**: the server writes data (bank ingestion) and reads it (rules, alerts, push). **R2**: two users share a household while personal accounts stay private. **R3**: offline writes on iOS and macOS. **R4**: a database you own or can self-host. **R5**: an ops load every self-hoster can carry, since each instance runs the whole server stack. "Status" is the latest GitHub release as of 2026-11-07.

| Option | Status / maintenance | License · cost | Swift support | Offline writes | Conflict model | Sharing / permissions | Server-side writes | E2EE | Ops (per instance) | Fit |
|---|---|---|---|---|---|---|---|---|---|---|
| **CKSyncEngine** (CloudKit) | Apple framework, iOS 17+ / macOS 14+ [cksync] | Apple; no server to run | First-party | Yes; the app persists engine state | Per record; the app handles `serverRecordChanged` [cksync] | CKShare via the shared DB; one engine per DB [cksync] | **No** for private and shared DBs: server-to-server keys reach only the public DB, and web auth tokens expire after 30 min or 2 weeks (archived doc) [ck-ws] | n/a (Apple-hosted) | None | ✗ R1 |
| **SQLiteData** (Point-Free) | 1.12.1, 2026-10-09; active [sqd] | MIT | Excellent: GRDB plus SwiftUI observation | Yes | Per field, "last edit wins"; not customizable. Schema rules: UUID keys, no UNIQUE columns, no column renames or removals [sqd-sync] | Root records only; records with more than one foreign key or many-to-many links aren't shared; read-only and read-write permissions [sqd-share] | Same CloudKit limit | n/a | None | ✗ for sync; ✓ as the local layer |
| **PowerSync** | Swift SDK 1.17.0 (2026-11-02); service 1.28.1; Swift-native since 1.14 (Rust core) [ps-swift] | SDK and service Apache-2.0 (service moved from FSL on 2026-11-03) [ps-lic]; Cloud free tier pauses after 1 week idle, Pro from $49/mo [ps-price] | Good: iOS 15+, macOS 12+, HTTP streaming only; GRDB integration **alpha** [ps-plat], [ps-grdb] | Yes, through an upload queue | Your backend decides; the default is per-field LWW in arrival order; delta patterns are documented; clients never rebase [ps-cons], [ps-conf], [ps-custom] | Sync Streams: SQL-like rules that sync per user or per household [ps-streams] | **Yes**: any write to Postgres replicates | Only encryption at rest of the local SQLite [ps-enc] | One more service (a Docker image) plus bucket storage, MongoDB or Postgres; Postgres stays on storage version 2. No Dashboard when self-hosted; usage metrics are shared unless disabled [ps-deploy], [ps-host], [ps-cfg], [ps-tel] | ✓ best to adopt |
| **ElectricSQL** | sync-service 1.8.1 (2026-09-07), still the latest release [el-rel]. Electric joined Databricks (2026-08-11) and Electric Cloud is "winding down"; the code "stays open source", with no maintenance commitment [el-dbx] | Apache-2.0 | No official Swift client; you'd write one over HTTP | Do it yourself: "does not do write-path sync" [el-writes] | Do it yourself | Shapes plus your own auth proxy | Yes | n/a | One more service | ✗ half a solution, uncertain upkeep |
| **Automerge** | Core 3.x active; automerge-swift 0.7.2 (2025-12); automerge-repo-swift 0.3.2 (2024-11, stale) [am] | MIT | Fair | Yes | Document CRDT | Per document; auth is yours to build | The server must load and merge documents | Only if the server is a blind relay | A sync server, plus a projection into SQL for queries | ✗ for the ledger |
| **Yjs/Yrs** (yswift), **Loro** (loro-swift) | yswift 0.2.1 (2024-04, stale); loro-swift 1.17.1 (2026-10, active) [ys], [loro] | yswift: no SPDX license on GitHub; Loro: MIT | Thin bindings | Yes | Text and collaboration CRDTs | Yours to build | The server must merge | Only with a blind relay | Your own server | ✗ for the ledger; fine for rich text |
| **cr-sqlite** | Last release v0.16.3 (2024-01); only build fixes in 2026 [crsql] | MIT | Loadable SQLite extension | Yes | CRDT tables (multi-master) | Yours to build | The server runs SQLite plus the extension | Hard | Your own server | ✗ maintenance risk |
| **sqlite-sync** (SQLite Cloud) | 1.2.2 (2026-11-06); young [sqls] | Apache-2.0 | SQLite extension | Yes | CRDT | Row-level, through their service | Through a Postgres extension | No | A self-hosted Postgres must still be registered in their CloudSync dashboard [sqls-pg] | ✗ depends on their relay |
| **Ditto** | Swift SDK 5.2.0 (2026-10) | Proprietary; free tier is 10 cloud device connections and 2 GB [ditto-price] | Good | Yes (peer-to-peer mesh) | CRDT: REGISTER is LWW, MAP is add-wins [ditto-cons] | Built in | Through Ditto Server, in the cloud or on-premises [ditto-gloss] | No | Vendor-run | ✗ cost and lock-in |
| **Couchbase Lite** + Sync Gateway | CBL Swift 4.1.2 / 3.4.2 (2026-09) | Couchbase Lite: Apache-2.0; Sync Gateway: BSL 1.1 with a production-use grant [sg-lic] | Good | Yes | Document revisions; deletes win; custom resolvers since 2.6 [cbl] | Sync Gateway channels (not verified) | Yes, through Couchbase Server | No | Couchbase Server plus Sync Gateway: heavy | ✗ ops load |
| **Realm** / Atlas Device Sync | Device Sync reached end-of-life and was removed on 2025-09-30; the local database stays open source [realm-eol] | Apache-2.0 | Good (local only) | — | — | — | — | — | — | ✗ sync discontinued |
| **GRDB + custom protocol** | GRDB 7.12.0 (2026-11-01) [grdb] | MIT | Excellent | Yes | Yours: domain operations plus LWW by HLC | Yours: household "spaces" | **Native** | Not compatible with server-side logic | Just your API and Postgres | ✓ recommended |

**Self-hosting and licenses.** on-budget is self-hosted open source, so R5 falls on every self-hoster, and every server dependency must be one that any self-hoster may run. PowerSync's move to Apache-2.0 settles its FSL question [ps-lic]. Ditto is proprietary, and Sync Gateway's BSL 1.1 "is not an Open Source license", by its own notice [sg-lic].

**Also seen.**

- **Zero 1.11** (Rocicorp) has TypeScript clients only: React, Solid and React Native [zero]. Its predecessor, Replicache, documents the push/pull/rebase protocol reused below [rep].
- **Turso**'s new engine has no Swift binding, and libsql-swift was last pushed in July 2025 [turso].
- **Graft** v0.2.1 (2025-12) is experimental [graft].

**Swift notes.**

- In May 2025 a practitioner on HN called local-first on Swift "pretty immature" and found no good GRDB integration [hn-bhl] (secondary, anecdotal).
- Artman's earlier sync framework at Uber was written in Swift: Jetstream, archived in 2017 [jetstream].

## 4. Conflict semantics for budget data

**Rule** (high):

- Use LWW only for an independent scalar field, where the latest value captures the user's whole intent.
- When the intent is a change (a move, a contribution) or a constraint, send an operation that the server applies inside a transaction. Two constraint examples: splits must sum to the amount, and one bank line matches at most one manual entry.
- YNAB's public API (OpenAPI 1.89.0) shows the data model, but not the write path [ynab-api], [ynab-spec]:
  - money moves are stored as `money_movements` rows with `from_category_id`, `to_category_id`, `amount` (int64 milliunits), `performed_by_user_id` and `moved_at` ("processed on the server"), and clients receive deltas through `server_knowledge` cursors;
  - but those endpoints are GET-only. Allocations are written as absolute values, through `PATCH /plans/{plan_id}/months/{month}/categories/{category_id}`, where "Only `budgeted` (assigned) amount can be updated".
- So the case for `moveMoney` rests on Actual's failure mode: its moves rewrite absolute cells, and one of two concurrent offline moves is lost (section 2, and case 1 below).

| Data | Writers | Semantics | Why |
|---|---|---|---|
| Bank transaction facts: amount, dates, bank label, status, provider ID | Server ingestion only | Immutable. The server moves a transaction from pending to booked, keeping a stable internal ID | These are facts. Annotations attach to the internal ID, so they survive a provider ID change |
| Annotations: category, payee alias, note, tags, excluded flag | Users and rules | Per-field LWW by **HLC**, but **source rank wins first** (user > rule > import) | Independent scalars. A rule must never overwrite a human choice |
| "Reviewed" flag, notification state | Each user | **Per-user rows**, keyed by (transaction, user) | No conflicts by construction |
| Split set | Users | The whole set is **one value**, replaced atomically; the server checks that the lines sum to the amount | Merging line by line can produce an invalid sum. Figma keeps linked properties in one value for the same reason [figma] |
| Matching or merging a manual entry with a bank line | Users; the server suggests | A server operation: the first one wins, the second is rejected with a reason | Enforces uniqueness. Same as Replicache's "reserveRoom" pattern [rep] |
| Monthly allocation | Users | **Operations**: `moveMoney(month, from, to, amount)` and `assign(category, month, Δ)`. A typed absolute value becomes `setAssigned(value, expectedBase)`: it applies, and posts a conflict notice if the base no longer matches | LWW loses concurrent moves, while deltas double-count two concurrent "set to 300". The gesture tells you which intent the user had |
| Goals: target, date, name | Users | Per-field LWW | Scalars |
| Goal contributions | Users and server | Append-only events, or derived from money movements | Additive |
| Categorization rules | Users | Per-field LWW. Order uses a fractional index, as in Linear's fractional `sortOrder` [re-linear]. Rules run **on the server** at ingestion | One deterministic place, with provenance recorded |
| Recurring schedules | Users | Series fields use LWW. Exceptions are separate rows keyed by (schedule, occurrence date) | "Skip March" and "raise the rent" don't collide |
| Deleting a category or an account | Users | A server operation that requires a reassignment target. Deleted rows become tombstones, and a delete beats an edit unless explicitly restored. PowerSync also recommends that deletes win [ps-conf] | No money left orphaned |
| Spending limits, alert thresholds | Users | LWW; alerts are evaluated on the server | Alerts need server-side data |
| Derived numbers: available, balances, amount left to budget | Nobody | Never synced as truth. Both sides recompute them from facts and operations | No write conflicts on totals |

**Where per-field LWW loses money-relevant intent.** Each case below can be reproduced with two offline devices. Confidence is high for all of them.

1. **Concurrent moves.**
   - Both partners move €50 into Groceries, which starts at €100.
   - LWW on the absolute value writes 150 twice, so the result is 150 instead of 200.
   - Each source category still lost €50, so €50 silently reappears in the amount left to budget.
   - Actual's code has this shape [act-budget].
2. **Goal progress stored as one number.** Concurrent contributions overwrite each other.
3. **Split edits merged line by line.** LWW can combine two different splits into a set that no longer sums to the amount.
4. **Automation against a human choice.** With LWW on timestamps alone, a later rule run overwrites a manually set category.
5. **Stale offline edits.**
   - With LWW in arrival order (Linear, Figma, PowerSync's default), a phone that was offline for three days overwrites newer Mac edits when it reconnects.
   - Ordering by HLC keeps the newer intent.
6. **Delete against assign.** Deleting a category while a partner assigns money to it strands that money.
7. **Double merge.** Two users match the same bank line to two different manual entries.

LWW is enough for names, colours, notes, flags, goal targets, rule fields, schedule fields and settings.

**One user on several devices, or two users.** The mechanics are the same; the frequency differs.

- One user rarely edits the same field on two devices at once. Stale offline edits (case 5) are the main risk.
- Two users run into allocation moves and matching at the same time (cases 1, 6 and 7). The answers:
  - operations plus server validation;
  - per-user state for "reviewed";
  - a server-generated activity feed ("{member} moved €50 from Fun to Groceries"), like Linear's `IssueHistory` side effects [re-linear], so that a surprising result can be explained.

## Recommendation for on-budget

**Decision: build, unless Spike A shows PowerSync fits with less code** (medium; lowered from medium-high after the review found protocol bugs and the product became self-hosted).

**Why build:**

- A self-hosted instance holds one owner's households, with a few thousand transactions per household per year, so a full household bootstrap is a few MB. No partial or lazy loading is needed.
- The hard part is domain semantics (section 4), which no library gives you.
- A server-ordered log removes the need for Merkle tries and CRDT metadata.
- Ops stay at one API and one Postgres per instance. That matters more now that every self-hoster runs the whole stack (R5).

**When to adopt PowerSync instead.**

- Adopt it only if the spike shows that its client SDKs (the GRDB integration is alpha) and Sync Streams fit with less code. Its checkpoints already give the read-path properties the review found missing in the first draft: one consistent snapshot, "no intermediate state" within a transaction, and server state kept apart from pending writes [ps-cons].
- You'd still own the write API and the semantics. You'd no longer write the bootstrap, pull, poke and checkpoint code.
- Every instance then runs one more service and its bucket storage. Self-hosting caveats:
  - usage metrics "are shared with PowerSync unless you opt-out", so the configuration on-budget ships must set `telemetry.disable_telemetry_sharing: true` [ps-tel];
  - "The PowerSync Dashboard is currently not available when self-hosting" [ps-host];
  - Postgres bucket storage "does not currently support version 4" of PowerSync's storage format, the MongoDB default, so it stays on version 2 [ps-cfg];
  - the service ships as a Docker image (`journeyapps/powersync-service`) [ps-host], while ADR 0003 deploys with NixOS and no Docker, so it would have to be packaged from source or run as an OCI container (inference).

**What to avoid:**

- CloudKit and SQLiteData sync: they fail R1 and R2.
- CRDT libraries for the ledger: wrong semantics, and no server-side invariants.
- Ditto, Couchbase and Realm: cost, ops load and end-of-life respectively. For an open-source, self-hosted product, Ditto's proprietary license and Sync Gateway's BSL also count against them.
- ElectricSQL: read path only, and uncertain upkeep since it joined Databricks [el-dbx].

**Trade-offs you accept** (high):

- **You own the correctness** of cursor ordering, idempotency, rebase and schema skew. Budget for the simulator from day one. The review of this note found several such bugs in the first protocol draft; the version below fixes them.
- **No E2EE of budget data**, because the server must categorize, alert and push. Mitigations:
  - TLS in transit;
  - encrypted Postgres volumes and backups;
  - SQLite encryption or iOS Data Protection on devices;
  - optionally, later: E2EE for free-text notes only, with a household key.
- **Freshness comes from the server**, not the device. An iOS app is never always on, so ingestion, alerts and push notifications run on the server. This is the reverse of Actual's model.

### Data flow

```
 Aggregator (Enable Banking / Powens / …)  ──poll (no webhooks)──▶  ingestion worker (server)
   normalize → dedupe (provider id, else amount:date:occurrence [ynab-spec]) → deterministic UUID
   → INSERT bank_transactions (facts) → apply rules (source=rule) → recompute balances/alerts
   → ONE DB tx per account batch: takes the instance's next sync_id, appends change_log rows (seq 1…n)
                                  │
                     poke (WebSocket/SSE) + APNs (alerts)
                                  ▼
 Device: SQLite confirmed tables ◀── pull(after cursor) ── change_log   push(outbox) ──▶ /push
         + outbox (named ops), replayed over them for the UI ───────────────────────────┘
```

Ingestion polls: a webhook needs inbound connections from the internet, which ADR 0003 rules out, since an instance may be reachable only over a private network.

### Minimum viable protocol

**Server tables (Postgres).**

- `sync_counter(last_sync_id bigint)`: one row for the whole instance
- `spaces(id, kind personal|household)`
- `memberships(space_id, user_id, role)`
- `clients(client_id, user_id, last_mutation_id)`: one client per local database, not per device (see "One writer per local database")
- `change_log(sync_id, seq, space_id, entity, entity_id, op upsert|delete, row jsonb, client_id, mutation_id, PK(sync_id, seq))`, indexed on `(space_id, sync_id)`
- Every domain row carries `space_id` and `deleted_at`, and stores `(value, hlc, source)` for each LWW field.

**Allocating sync IDs.**

- **One counter per instance**, as Linear's `lastSyncId` "spans the entire database" [re-linear]. A self-hosted instance has few writers, so one row lock is cheap, and a write that touches two spaces still gets a single ID. The cost is that members can infer how busy the instance's other households are, which is acceptable when all households belong to one owner (inference).
- Each writing transaction starts with `UPDATE sync_counter SET last_sync_id = last_sync_id + 1 RETURNING last_sync_id`. Its `change_log` rows share that ID and are numbered by `seq`. Keying rows by the ID alone would fail at the second row of a transaction, and `moveMoney` writes at least two, ingestion many. Linear numbers every change instead and returns the last one, because "the mutation might make, you know, hundreds of other … changes at the same time" [lin-dt61].
- The row lock serializes writers, so mutators validate against the latest committed state, and IDs follow commit order. An aborted transaction rolls its update back; a rejected mutation may leave an ID without rows, which is harmless. No ID ever commits after a higher one, so a cursor never passes a row it hasn't seen.
- **Don't use a bare `bigserial` as the cursor** (high):
  - `nextval` is never rolled back, which leaves gaps [pg-seq].
  - Values are allocated before commit [pg-createseq].
  - A reader could move past ID 11 while ID 10 hasn't committed yet, and then miss 10 forever.

**`POST /push {clientId, mutations[]}`.** Each mutation runs in its own transaction, the "simplest way" in Replicache's guide [rep-push]:

1. Lock the `clients` row.
2. Skip any `mutation_id ≤ last` and return its recorded result, which makes replays idempotent, as in Replicache [rep] and PowerSync [ps-conf]. Require mutation IDs to be contiguous.
3. Take the sync ID, then check membership and role.
4. Open a `SAVEPOINT` and run the mutator: validate invariants, write rows and append `change_log` rows.
5. On a domain rejection or a permanent database error, `ROLLBACK TO SAVEPOINT`, record `{rejected, reason}`, and still advance `last_mutation_id`, so one bad operation never blocks the queue [ps-cons], [rep-push].
   - Postgres needs the savepoint: after an error, "ROLLBACK TO is the only way to regain control of a transaction block that was put in aborted state by the system due to an error, short of rolling it back completely and starting again" [pg-tx].
   - A batch run as one transaction needs a savepoint per mutation. Otherwise one error discards the whole batch along with the `last_mutation_id` bump, and the queue blocks.
6. Transient failures (a lost connection, a serialization failure `40001`, a deadlock `40P01`) abort without advancing, and the client retries. Replicache allows this only when the server "definitely will be able to process the mutation later" [rep-push].
7. Respond with `{results[{mutationId, syncId} | {mutationId, rejected, reason}], lastMutationId}`, then poke.

**`GET /bootstrap`.** Streams the rows of all the member's spaces and the counter value as NDJSON (newline-delimited JSON), from one REPEATABLE READ snapshot, like Linear's streamed bootstrap [lin-dt61]. The counter value becomes the client's cursor.

**`GET /pull?client=&after=`.** One request covers all of the member's spaces, so a write that touches two spaces never shows up half-applied. Each page reads the counter, the rows after the cursor and the client's `last_mutation_id` in one REPEATABLE READ snapshot.

- Why one snapshot: under Postgres's default, READ COMMITTED, "two successive SELECT commands can see different data, even though they are within a single transaction" [pg-iso]. A cursor read on its own could pass rows the response didn't carry, which the client would then skip forever. A separate `lastMutationId` could confirm a mutation whose rows aren't in the response. Replicache likewise requires a mutation's effects and its `lastMutationID` to be "revealed atomically" [rep-push].
- Returns:
  - change rows after the cursor, in pages that end on a whole `sync_id`;
  - the new cursor: the last `sync_id` of a partial page, or the counter value on the final page;
  - `lastMutationId`, on the final page;
  - `410` if the cursor is older than the compaction floor. The client then re-bootstraps.

**Membership changes.** The server writes them as change rows in the member's personal space. They are rare, so the client simply re-bootstraps all its spaces from one snapshot, keeping its outbox, or purges the space it left. Linear's sync-group action plays the same role [re-linear].

**Server jobs.** Bank ingestion and other jobs use the same mutator path, with `client_id = server`. Keep each transaction to one account's batch, because the counter lock serializes every writer in the instance.

**Client tables (SQLite: GRDB in this note's Swift plan, with SQLiteData optional for SwiftUI observation).**

- Confirmed entity tables, written only from bootstrap and pull, so they always equal the server's state at the cursor. Linear likewise writes its IndexedDB tables only from server deltas [re-linear].
- What the UI reads: the confirmed rows with the pending mutations replayed on top, kept as an overlay of the rows they change.
- `outbox(mutation_id, name, version, args, hlc, status, sync_id)`, where `sync_id` comes from the ack
- `sync_state(cursor)`
- These replace the first draft's field guards, which skipped server values for fields with pending edits. Keeping guards would require each guard to hold the last server value and write it back when it drops, and the server to echo the resolved value of every field a mutation touched, even when the write lost.

**One writer per local database.**

- The client ID, the outbox, the mutation counter and the HLC belong to a local database, not to a device or a process.
- A browser runs many clients on one device: Replicache has "typically one client … per tab", grouped into one client group per browser profile [rep]. OPFS adds a hard limit: a sync access handle "takes an exclusive lock on the file" and exists only in dedicated workers [mdn-sah].
- So one leader per browser profile, elected with a Web Lock, owns the SQLite file, the outbox, the HLC and the poke stream, and the other tabs go through it, as web-client-sync.md §2.3 proposes. TanStack DB's offline executor works the same way: "Leader election chooses one tab to process the outbox" [tanstack-ot]. The alternative is Replicache's model, with one client ID per tab inside a group.

**Local storage durability.**

- Browser storage is "best-effort" by default and is evicted under storage pressure. Safari and most Chromium browsers approve or deny `navigator.storage.persist()` "based on the user's history of interaction with the site" [mdn-quota].
- A WebView inside a native app is no safer: "the OS will reclaim local storage from Web Views if a device is running low on space. The same can be said for IndexedDB at least on iOS" [cap-storage].
- So, if the shared UI runs in a WebView inside the iOS and macOS apps, keep their database and outbox in native SQLite. On the web, call `persist()`. Everywhere, push within seconds and show the count of unsynced changes. A lost database is re-bootstrapped, but a lost outbox is lost work.

**Client steps.**

- **Mutate.** In one local write transaction:
  1. run the same mutator locally, over what the UI reads (optimistic);
  2. append the mutation to the outbox.
- **Push.** Send mutations in order, in batches.
  - **On ack**: store the mutation's `sync_id`. It stays pending until the cursor reaches that ID, Linear's "wait on sync" rule [lin-dt61].
  - **On reject**: drop the mutation, rebuild the view without it and show a toast with the reason, as Linear does [lin-dt61].
- **Pull.** Apply the whole response, all pages, in one local transaction:
  1. Write every row to the confirmed tables, in (`sync_id`, `seq`) order. Never skip one. LWW compares `(source_rank, hlc, client_id)`, so an acknowledged edit can still lose on the server: to a partner's later-HLC value that arrived first, or because it is an old offline edit. Only the server's rows say so. Figma's discard rule doesn't transfer, because its LWW follows arrival order [figma].
  2. Set the cursor.
  3. Drop the pending mutations that are now confirmed: `mutation_id ≤ lastMutationId`, or acked with `sync_id ≤ cursor`.
  4. Rebuild the view of the rows that changed and of every row a pending or just-dropped mutation touches: confirmed rows, then the pending mutations replayed in order. Replicache does this: it "rewinds the state of the Client View to the last version it got from the server, applies the patch … and then replays any pending mutations on top" [rep]. PowerSync's simpler rule also works: while mutations are in the upload queue, the client "does not advance to a new checkpoint", and when it does advance, it "updates the state to exactly match that of the checkpoint" [ps-cons].
  - This also covers a pull, triggered by a poke, that brings a mutation's echo before its ack: the echo lands in the confirmed tables, and the mutation is replayed on top until step 3 drops it.
- **On reconnect**: push first, then pull.
- **Insert-only operations** (movements, contributions) use client-generated UUIDs, so replaying one over its own echo is an idempotent upsert.

**Clocks and IDs.**

- **Clocks**:
  - Each local database runs one HLC (on the web, in the leader), using Actual's algorithm [act-ts] without its failure mode.
  - That failure mode: Actual's receive path runs `if (lMsg - phys > config.maxDrift) { throw new Timestamp.ClockDriftError(); }`, with `maxDrift` at 5 minutes, and since PR #8905 (master only) its server rejects messages more than the same 5 minutes ahead [act-server]. So a message the server accepts can still trip a client whose clock runs slightly behind. Users report "Time disagreement causes sync to fail without explanation" [act-3786] and "Failed to download due to device time", open since 2026-03-01 [act-7106].
  - Decide three things up front (inference):
    - the server bounds HLCs to a window well under the clients' `maxDrift` (say 1 minute against 5), and clamps later values to that bound instead of rejecting the edit;
    - clients never throw on server rows, which the server has already bounded;
    - a device whose clock is off shows a visible clock error, measured against the server's `Date` header.
  - LWW compares `(source_rank, hlc, client_id)` in that order.
  - Server-side writes use a server HLC.
- **IDs**:
  - Inserts use client-generated UUIDv7.
  - Bank rows use a deterministic UUIDv5 built from `(account, provider_tx_id)`.
- **Money**: store amounts as Int64 minor units. YNAB uses int64 milliunits [ynab-spec].

**Schema evolution.**

- Mutations are named and versioned, for example `moveMoney@1`.
- The server accepts version N−1 for as long as a device may stay offline.
- Schema changes are additive only.
- After a breaking change the client may wipe its cache and re-bootstrap, but it **keeps the outbox**. This is Linear's schema-hash approach [re-linear].
- Actual's deferred-message mechanism [act-sync] isn't needed.

**Live updates.**

- A WebSocket or SSE poke `{lastSyncId}`, sent to the members of the spaces a transaction touched, triggers a pull.
- APNs delivers user-visible alerts, plus a best-effort refresh hint.

### Testing strategy

1. **Shared, pure mutators.**
   - Keep `apply(op, state) → state | rejection` pure.
   - If the server is written in the clients' language (Swift in this note's plan), share one package between the apps and the server.
   - Otherwise, maintain JSON golden vectors that both implementations must pass.
2. **Deterministic simulation.**
   - Run 2–4 clients and the server in one process, with a seeded RNG for every choice, as in FoundationDB [fdb] and TigerBeetle's VOPR [vopr].
   - Faults to inject:
     - random user operations;
     - offline windows;
     - delayed, reordered or duplicated requests;
     - responses lost after the server has committed;
     - app kills between the local commit, the push and the ack;
     - clock skew of ±10 minutes;
     - clients on different app versions;
     - several tabs of one browser profile, and the browser evicting the local database.
   - Use in-memory GRDB and the real server logic. Control time with `swift-clocks` [swift-clocks].
3. **Invariants to check once the system is quiet.**
   - Each client's confirmed tables equal the server's projection of that client's spaces at its cursor. With an empty outbox, so does what its UI shows.
   - No pull shows half of a transaction, including one that spans two spaces.
   - Every acknowledged mutation is either reflected or explicitly rejected.
   - Replaying the outbox twice gives the same result as replaying it once.
   - Cursors only move forward.
   - No row from a space the user isn't in ever reaches their client.
   - Split lines sum to the transaction amount.
   - Assigned amounts equal the sum of money movements.
   - Balances equal the sum of the facts.
4. **Property-based tests.**
   - Generate and shrink cases with `swift-property-based` [swift-pbt], and store failing seeds as regression tests.
   - Run it in CI on every PR, plus a nightly sweep of 100,000+ seeds.
   - Actual's convergence property test is skipped today [act-proptest]. Keep yours fast enough that it stays on.
5. **Postgres-level tests.**
   - Test the locking and cursor logic against a real Postgres in Docker, with concurrent pushers, aborted transactions, a mutation that fails in the middle of a push, and pulls racing commits.
   - A Jepsen-style check is optional. The third-party Jepsen suite for PowerSync [jepsen-ps] shows what such a check finds:
     - it drives "a simple Dart CLI PowerSync implementation";
     - with a `getCrudBatch()` upload connector it saw "read committed, non-atomic" transactions, and causal consistency only with a "newly developed Causal `getNextCrudTransaction()` backend connector";
     - it reports "full Causal Consistency, Strong Convergence, and Atomicity in a no-fault environment", and "occasional sync pauses" under heavier faults.

## Open questions and spikes

1. **Spike A (2 days): self-hosted PowerSync.** It now decides build versus adopt; web-client-sync.md's W2 widens it to a web client.
   - Setup:
     - Postgres 14 or later as both source and bucket storage, which may then share one server [ps-cfg];
     - the client SDK the shared UI would use, and the GRDB alpha integration only if the watch syncs on its own;
     - Sync Streams for personal and household spaces;
     - `telemetry.disable_telemetry_sharing: true` [ps-tel].
   - Upload: drain `uploadData` one transaction at a time with `getNextCrudTransaction()`, the connector shape that passed the Jepsen suite's causal checks [jepsen-ps]. The SDK expects "one batch (or one transaction) per call" [ps-upload].
   - Measure:
     - how much code it saves;
     - app size;
     - background behaviour;
     - how domain operations travel through `uploadData`;
     - what each instance takes on: packaging the Docker image for ADR 0003's NixOS deployment, upgrades, and monitoring without the Dashboard.
2. **Spike B (3–5 days): a skeleton of the custom protocol, plus the simulator.** Exit criterion: convergence and the money invariants hold across 10,000 seeds, including the multi-tab and clock-skew faults.
3. **Server language.**
   - Swift (Hummingbird or Vapor, with PostgresNIO) would let the server share mutators, the HLC and validation code with the apps.
   - The alternatives are TypeScript or Go. If the shared UI codebase is TypeScript (likely, not decided), a TypeScript server shares mutators with every app except the watch, and Swift only with the watch.
   - I didn't research this.
4. **Household model.**
   - Joint accounts live in the household space, and personal accounts stay private.
   - Can a partner see only the balances of personal accounts?
   - Can a transaction move between spaces? With one sync counter per instance, the move is one transaction with one sync ID, which every pull shows whole, and its mutation waits for a single `sync_id`.
5. **Allocation UX.** "Move money" versus typing absolute values, and how to show conflict notices.
6. **Bank aggregator** for the first supported banks: BoursoBank, Crédit Agricole Aquitaine and Crédit Mutuel du Sud-Ouest.
   - This needs separate research.
   - Actual already integrates Enable Banking, and GoCardless Bank Account Data is closed to new accounts [act-docs-bank].
   - How often bank consent must be renewed affects the ingestion design. So does the product's choice that each member connects their own provider credentials, and the polling-only rule above.
7. **Retention and recovery.**
   - Compaction and retention policy.
   - Backup and restore semantics: a restore forces every client to re-bootstrap.
   - Undo and redo through inverse operations, as Linear does: in v1 or later?

**Could not verify.**

- The content of Linear's YouTube talks (no transcript access).
- Linear's 2026 internals: whether `lastSyncId` is still global, how commit order is guaranteed, the migration mechanism (secondary sources only), and whether the native mobile apps use the same engine.
- CloudKit quotas and pricing in 2026; Ditto's self-hosting terms for small deployments; the Couchbase Community vs Enterprise feature split.
- How stable PowerSync's GRDB alpha is in practice, and the effort to build the custom protocol (no source; low confidence).
- Whether PowerSync's Postgres bucket storage will gain storage version 4, and what staying on version 2 costs in practice.
- Whether Electric's open-source sync service will stay maintained: the Databricks announcement promises only that the code "stays open source".
- The clock window (1 minute against a 5-minute `maxDrift`) is a guess; no source tests it.

## Sources

All links checked on 2026-11-07. Labels are links. "Secondary" marks sources not written by the project's own team.

- **Linear and Figma:** [lin-dt61] devtools.fm #61, "Tuomas Artman – Linear", transcript (2023-07-31) · [lin-scaling] T. Artman, "Scaling the Linear Sync Engine" (2023-06-29, video; no transcript obtained) · [lin-collab] Linear changelog, "Collaborative editing" (2023-12-06) · [re-linear] W. Hu, "Reverse Engineering Linear's Sync Engine" (2024–2025, **secondary**) · [re-code] Linear client bundle annotated in that repository (**secondary**) · [hn-artman] Artman on HN (2025-05-31) · [hn-bhl] HN comment on Swift local-first (**secondary**, anecdotal) · [jetstream] Uber Jetstream for iOS (archived; Artman's pointer: <https://news.ycombinator.com/item?id=44150328>) · [figma] E. Wallace, "How Figma's multiplayer technology works" (2019-10-16).
- **Actual Budget** (code at master @`13ee015`, 2026-11-07, which is not a release; the latest release is v26.11.0, 2026-11-02): [act-cmp] compare v26.11.0…13ee015 · [act-8905] PR #8905, "Reject sync messages too far in the future" (merged 2026-11-03) · [act-server-rel] sync server at v26.11.0, without the drift check · [act-3786] issue #3786 · [act-7106] issue #7106 · [act-ts] HULC clock · [act-merkle] Merkle trie · [act-proto] sync protobuf · [act-sync] client sync module · [act-server] sync server · [act-enc] encryption · [act-db] per-column messages and tombstones · [act-budget] `transferCategory` · [act-bank] bank import and matching · [act-proptest] skipped property test · docs: [act-docs-sync] syncing and E2EE, [act-docs-bank] bank sync, [act-docs-mu] multi-user · [jlong] J. Long, "CRDTs for Mortals" demo app (2019; talk: <https://www.youtube.com/watch?v=DEcwa68f-jY>) · [hlc] Kulkarni et al., "Logical Physical Clocks…" (2014).
- **Apple, Point-Free, GRDB:** [cksync] CKSyncEngine reference · [ck-ws] CloudKit Web Services Reference (archived, revised 2016-06-13) · [sqd] SQLiteData · [sqd-sync] "Getting started with CloudKit" · [sqd-share] "Sharing data with other iCloud users" · [grdb] GRDB.swift.
- **PowerSync:** [ps-swift] Swift SDK · [ps-grdb] GRDB (alpha) · [ps-plat] supported platforms · [ps-cons] consistency · [ps-conf] handling update conflicts · [ps-custom] custom conflict resolution · [ps-streams] Sync Streams · [ps-host] self-hosting · [ps-deploy] deployment architecture · [ps-enc] data encryption · [ps-lic] "Change license from FSL to Apache-2.0" (merged 2026-11-03) · [ps-price] pricing · [ps-tel] self-hosted usage reporting · [ps-cfg] self-hosted instance configuration · [ps-upload] client-side integration (`uploadData`) · [jepsen-ps] Jepsen tests of PowerSync (third party).
- **Other options:** [el] ElectricSQL · [el-rel] Electric releases · [el-dbx] "Electric is joining Databricks" (2026-08-11) · [el-writes] Electric writes guide · [am] automerge-swift (core: <https://github.com/automerge/automerge>) · [ys] yswift · [loro] loro-swift · [crsql] cr-sqlite · [sqls] sqlite-sync · [sqls-pg] its self-hosted Postgres quickstart · [ditto-price] Ditto pricing · [ditto-cons] Ditto consistency models · [ditto-gloss] Ditto glossary · [cbl] Couchbase Lite conflict resolution · [sg-lic] Sync Gateway BSL · [realm-eol] MongoDB Device SDKs end-of-life (also <https://www.mongodb.com/docs/device-sync-deprecation/>) · [zero] Zero · [rep] Replicache, "How Replicache Works" · [rep-push] Replicache push endpoint reference · [turso] Turso (also <https://github.com/tursodatabase/libsql-swift>) · [graft] Graft.
- **Browsers and web storage:** [mdn-sah] MDN, `createSyncAccessHandle()` · [mdn-quota] MDN, "Storage quotas and eviction criteria" · [cap-storage] Capacitor, "Data Storage" · [tanstack-ot] TanStack DB `offline-transactions` README.
- **Domain, databases, testing:** [ynab-api] YNAB API, "Delta Requests" · [ynab-spec] YNAB OpenAPI spec 1.89.0 (`MoneyMovement`, month-category `PATCH`, `import_id`) · [pg-seq] Postgres sequence functions · [pg-createseq] Postgres `CREATE SEQUENCE` · [pg-iso] Postgres transaction isolation · [pg-tx] Postgres tutorial, transactions and savepoints · [fdb] FoundationDB, "Simulation and Testing" · [vopr] TigerBeetle VOPR · [swift-pbt] swift-property-based · [swift-clocks] swift-clocks.

[lin-dt61]: https://www.devtools.fm/episode/61
[lin-scaling]: https://linear.app/now/scaling-the-linear-sync-engine
[lin-collab]: https://linear.app/changelog/2023-12-06-editor-improvements
[re-linear]: https://github.com/wzhudev/reverse-linear-sync-engine
[re-code]: https://github.com/wzhudev/reverse-linear-sync-engine/blob/main/code/Root.js
[hn-artman]: https://news.ycombinator.com/item?id=44143680
[hn-bhl]: https://news.ycombinator.com/item?id=44144755
[jetstream]: https://github.com/uber-archive/jetstream-ios
[figma]: https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
[act-cmp]: https://github.com/actualbudget/actual/compare/v26.11.0...13ee015036813612ffc0482053082a29f7f7dabb
[act-8905]: https://github.com/actualbudget/actual/pull/8905
[act-server-rel]: https://github.com/actualbudget/actual/blob/v26.11.0/packages/sync-server/src/sync-simple.js
[act-3786]: https://github.com/actualbudget/actual/issues/3786
[act-7106]: https://github.com/actualbudget/actual/issues/7106
[act-ts]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/crdt/src/crdt/timestamp.ts
[act-merkle]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/crdt/src/crdt/merkle.ts
[act-proto]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/crdt/src/proto/sync.proto
[act-sync]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/loot-core/src/server/sync/index.ts
[act-server]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/sync-server/src/sync-simple.js
[act-enc]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/loot-core/src/server/encryption/encryption-internals.ts
[act-db]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/loot-core/src/server/db/index.ts#L219-L281
[act-budget]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/loot-core/src/server/budget/actions.ts#L700-L733
[act-bank]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/loot-core/src/server/accounts/sync.ts
[act-proptest]: https://github.com/actualbudget/actual/blob/13ee015036813612ffc0482053082a29f7f7dabb/packages/loot-core/src/server/sync/sync.property.test.ts#L316
[act-docs-sync]: https://actualbudget.org/docs/getting-started/sync
[act-docs-bank]: https://actualbudget.org/docs/advanced/bank-sync
[act-docs-mu]: https://actualbudget.org/docs/config/multi-user
[jlong]: https://github.com/jlongster/crdt-example-app
[hlc]: https://cse.buffalo.edu/tech-reports/2014-04.pdf
[cksync]: https://developer.apple.com/documentation/cloudkit/cksyncengine-5sie5
[ck-ws]: https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/CloudKitWebServicesReference/SettingUpWebServices.html
[sqd]: https://github.com/pointfreeco/sqlite-data
[sqd-sync]: https://github.com/pointfreeco/sqlite-data/blob/main/Sources/SQLiteData/Documentation.docc/Articles/CloudKitSync.md
[sqd-share]: https://github.com/pointfreeco/sqlite-data/blob/main/Sources/SQLiteData/Documentation.docc/Articles/CloudKitSharing.md
[grdb]: https://github.com/groue/GRDB.swift
[ps-swift]: https://docs.powersync.com/client-sdk-references/swift
[ps-grdb]: https://docs.powersync.com/client-sdks/orms/swift/grdb
[ps-plat]: https://docs.powersync.com/resources/supported-platforms
[ps-cons]: https://docs.powersync.com/architecture/consistency
[ps-conf]: https://docs.powersync.com/handling-writes/handling-update-conflicts
[ps-custom]: https://docs.powersync.com/handling-writes/custom-conflict-resolution
[ps-streams]: https://docs.powersync.com/sync/streams/overview
[ps-host]: https://docs.powersync.com/intro/self-hosting
[ps-deploy]: https://docs.powersync.com/maintenance-ops/self-hosting/deployment-architecture
[ps-enc]: https://docs.powersync.com/client-sdks/advanced/data-encryption
[ps-lic]: https://github.com/powersync-ja/powersync-service/pull/878
[ps-price]: https://www.powersync.com/pricing
[ps-tel]: https://docs.powersync.com/maintenance-ops/self-hosting/usage-reporting
[ps-cfg]: https://docs.powersync.com/configuration/powersync-service/self-hosted-instances
[ps-upload]: https://docs.powersync.com/configuration/app-backend/client-side-integration
[jepsen-ps]: https://github.com/nurturenature/jepsen-powersync
[el]: https://github.com/electric-sql/electric
[el-rel]: https://github.com/electric-sql/electric/releases
[el-dbx]: https://electric.ax/blog/2026/08/11/electric-joining-databricks
[el-writes]: https://electric-sql.com/docs/sync/guides/writes
[am]: https://github.com/automerge/automerge-swift
[ys]: https://github.com/y-crdt/yswift
[loro]: https://github.com/loro-dev/loro-swift
[crsql]: https://github.com/vlcn-io/cr-sqlite
[sqls]: https://github.com/sqliteai/sqlite-sync
[sqls-pg]: https://github.com/sqliteai/sqlite-sync/blob/main/docs/postgresql/quickstarts/postgres.md
[ditto-price]: https://www.ditto.com/pricing
[ditto-cons]: https://docs.ditto.live/sdk/v4-7/sync/consistency-models
[ditto-gloss]: https://docs.ditto.live/home/glossary
[cbl]: https://docs.couchbase.com/couchbase-lite/current/swift/conflict.html
[sg-lic]: https://github.com/couchbase/sync_gateway/blob/main/licenses/BSL-Couchbase.txt
[realm-eol]: https://www.mongodb.com/docs/device-sdks-deprecation/
[zero]: https://zero.rocicorp.dev/docs/introduction
[rep]: https://doc.replicache.dev/concepts/how-it-works
[rep-push]: https://doc.replicache.dev/reference/server-push
[turso]: https://github.com/tursodatabase/turso
[graft]: https://github.com/orbitinghail/graft
[ynab-api]: https://api.ynab.com/
[ynab-spec]: https://api.ynab.com/papi/open_api_spec.yaml
[pg-seq]: https://www.postgresql.org/docs/current/functions-sequence.html
[pg-createseq]: https://www.postgresql.org/docs/current/sql-createsequence.html
[pg-iso]: https://www.postgresql.org/docs/current/transaction-iso.html
[pg-tx]: https://www.postgresql.org/docs/current/tutorial-transactions.html
[mdn-sah]: https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle/createSyncAccessHandle
[mdn-quota]: https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria
[cap-storage]: https://capacitorjs.com/docs/guides/storage
[tanstack-ot]: https://github.com/TanStack/db/blob/main/packages/offline-transactions/README.md
[fdb]: https://apple.github.io/foundationdb/testing.html
[vopr]: https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/internals/vopr.md
[swift-pbt]: https://github.com/x-sheep/swift-property-based
[swift-clocks]: https://github.com/pointfreeco/swift-clocks
