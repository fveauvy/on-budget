# Offline-first web client: sync, security, hosting

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: §4.6 slots and memory-only mode (later); §3.3 client migrations; the dedicated domain.

Research note, 2026-11-07, revised the same day after fact-checks and a completeness review. Product truth is [PRODUCT.md](../../PRODUCT.md) and ADRs [0001](../adr/0001-self-hosted-single-owner-instances.md), [0002](../adr/0002-one-ui-codebase-native-watch.md) and [0003](../adr/0003-nixos-just-tailscale-infrastructure.md). It builds on [sync-engine.md](sync-engine.md), [apple-client-stack.md](apple-client-stack.md), [bank-aggregation.md](bank-aggregation.md), [cross-platform-ui-stack.md](cross-platform-ui-stack.md), [native-apps-auth.md](native-apps-auth.md), [self-hosting.md](self-hosting.md), [nixos-vps-infra.md](nixos-vps-infra.md), [design-system.md](design-system.md), [watchos-app.md](watchos-app.md) and [i18n-l10n.md](i18n-l10n.md), which it reads but doesn't edit. Where they disagree with an earlier version of this note, native-apps-auth.md (sign-in, RP ID) and self-hosting.md (packaging, hostnames) win. Versions and prices are as of November 2026. "BCD" means MDN [browser-compat-data](https://github.com/mdn/browser-compat-data) on `main`, last commit 2026-11-06. "Secondary" marks third-party sources, and "judgment" marks my own assessment. Confidence is rated **high**, **medium** or **low**.

## Question

on-budget has an offline-first web app (desktop and mobile browsers, installable on macOS and iOS) at full parity with the native apps, offline included. It shares one UI codebase with the iPhone and Mac apps, and the Watch app is native SwiftUI (ADR 0002). What does the web change for the sync engine, security and hosting? Which constraints does it put on the sync protocol, the client stack and the backend language?

## TL;DR

**Recommendation.** Keep one server-authoritative HTTP sync protocol for every client and SQLite on every client. Treat the browser as the least trusted device: shared devices keep data in memory only; trusted devices keep full offline data, as PRODUCT.md's parity requires, locked behind a passkey.

1. **Storage.** Run SQLite WASM 3.54 on OPFS (`opfs-sahpool`) in **one leader worker per browser profile**, elected with Web Locks; other tabs go through it. The leader opens only the signed-in member's storage slot, one pool per member and one database per household, so partners sharing a profile and the owner's test and production households stay apart (§4.6). Set `preserveOnInitFailure` and hand the VFS over with pause/unpause, or a failed start can wipe it. If W8 picks §4.5's encrypted-snapshot path, the database lives in memory instead. The browser database is a disposable cache. Push edits within seconds, and ask Safari users to install the app (Dock or Home Screen): in a plain tab, Safari deletes script-written storage, OPFS included, after seven days of Safari use without a click or tap on the site. Server-set cookies survive, so the app must handle reopening signed in to an empty database. **Medium-high.**
2. **Sync: build, as sync-engine.md decides, unless Spike A shows PowerSync "fits with less code".**
   - cross-platform-ui-stack.md picks Expo, so the iPhone, Mac and web share one TypeScript sync client. The same core runs in the web leader worker and on React Native's JS thread, over injected SQLite and transport ports (§3.4). The web adds browser storage, re-bootstrap after eviction and multi-tab plumbing, not a second client.
   - PowerSync ships that plumbing, but each instance would also take on a Node service with no NixOS package (ADR 0003 deploys without Docker), a second hostname (PowerSync doesn't support sharing one) with a JWT readable by JavaScript, and a `BYPASSRLS` replication role that moves household isolation into Sync Streams queries and JWT claims (§3.2). W2 widens Spike A to Safari and counts those costs.
   - Either way, writes stay named domain mutations through your API. With PowerSync, a failed upload blocks the client's downloads, so the API answers retired mutation versions with a 2xx rejection.

   **Medium.**
3. **Security.**
   - Web sessions use an HttpOnly `__Host-` cookie with `SameSite=Strict` on the same origin. These are the cookie and CSRF rules RFC 10017 sets for its BFF; the RFC covers OAuth browser clients, so it applies here by analogy. Sibling subdomains count as the same site, so the exact `Origin` check and a required custom header are mandatory. Never put a refresh token in JavaScript.
   - The passkey RP ID is the exact instance host, which is also the web origin, and it can never change. Native apps sign in on the same page inside `ASWebAuthenticationSession` and need no association file (native-apps-auth.md).
   - Use a hash-based strict CSP (`'strict-dynamic'`) plus `'wasm-unsafe-eval'`, Trusted Types (now in all three engines) with a `default` policy for Expo's chunk scripts and workers, a `style-src` that admits the empty runtime `<style>` elements of react-native-web and Unistyles by hash, and no third-party scripts (§3.4).
   - Encrypt the local database with a key wrapped by the passkey PRF output (Safari 18+ for passkeys, 26.4+ for security keys). **No web implementation is verified yet.** SQLite3 Multiple Ciphers' WASM build encrypts `opfs-sahpool` but has an open rollback bug at page sizes of 4 KiB or less, also reported on PowerSync's build of the same codec. An in-memory database with an encrypted snapshot needs only the canonical build. Spike W8 decides, for build and adopt alike (§4.5). Where PRF is missing, keep offline data under a non-extractable device key and a passkey gate in the UI, which protects less.
   - Shared computers get an in-memory database.
   - Logout revokes the session on the server, expires its cookie, sends `Clear-Site-Data: "cache"` (never `"cookies"`, which reaches sibling hosts), logs out every tab over BroadcastChannel and wipes the member's slot.

   **Medium-high; PRF: high for passkeys on iOS 18+ and macOS 15+, lower elsewhere; the encryption code path: low-medium.**
4. **Hosting.** **One origin**, the instance host (`https://<host>`, which is `public_url` and the RP ID), serves the SPA, the API, the OAuth pages, the bank callback, sync and live pokes, so there is no CORS. The instance is reachable only over Tailscale (PRODUCT.md, ADR 0003), so the web app works on the tailnet only. `<host>` is a `*.ts.net` name with Tailscale's certificate, or a name under the owner's domain with A/AAAA records to the Tailscale addresses and an ACME DNS-01 certificate. Nothing has to be public: only the optional per-build native passkey sheet needs an association file at that exact host, through split-horizon DNS. Every browser needs Tailscale. Add Web Push (VAPID) beside APNs; both send outbound. **High for same-origin, medium for the DNS and certificate setup.**
5. **Client stack and backend.**
   - cross-platform-ui-stack.md picks Expo (React Native 0.88) with react-native-web for the iPhone, web and Mac, plus a SwiftUI watch (ADR 0002).
   - SQLite: `expo-sqlite` with SQLCipher on iOS (cross-platform-ui-stack.md §4.8), or op-sqlite if PowerSync is adopted. On the web, the official build in the leader worker: the web backends of `expo-sqlite` and op-sqlite need COOP/COEP and coordinate no tabs (§3.4).
   - **Backend: TypeScript on Node 26 LTS.** Mutators, validation and the watch's GlanceBuilder are then written once, and native-apps-auth.md's authorization server uses `oidc-provider` and SimpleWebAuthn instead of a hand-rolled Swift one. The gate is cross-platform-ui-stack.md's spike S2: if the UI stays TypeScript, so does the server (§3.5).
   - TypeScript-only engines are out on status, not language: Zero has no offline writes, Instant is sunsetting, LiveStore is beta, Jazz is alpha, Triplit is idle. Apple-only sync stays out.
   - Tokens are defined once. Components are written once, plus a SwiftUI subset for the watch and widgets.
   - Hermes's iOS `Intl` is partial (no `formatToParts`, `signDisplay` or `PluralRules`), so FormatJS polyfills are mandatory, with golden tests across platforms.

   **Medium.**

## 1. What changes

| Area | Apple-only plan (apple-client-stack.md) | With the web app |
|---|---|---|
| Local store | GRDB/SQLite in an app group | Adds SQLite WASM in OPFS, which the browser may evict, with one writer per profile and one slot per member (§2, §4.6) |
| Sync | Build a custom engine with mutators shared in Swift; the PowerSync spike is only a benchmark | One TypeScript client on two hosts, the leader worker and React Native's JS thread; build stays the default and Spike A, widened by W2, decides (§3) |
| Auth | Passkeys plus refresh tokens bound to a Secure Enclave key | Adds cookie sessions and CSRF defense; the RP ID is the exact instance host, which serves the web app (§4.2) |
| Data at rest | Data Protection, Keychain, LocalAuthentication lock | No Keychain or Secure Enclave API; a PRF-wrapped key, or a weaker device key where PRF is missing; an encryption code path still to prove; a shared-computer mode (§4.5) |
| Attack surface | App binary and a TLS API | Adds XSS, npm supply chain, extensions, shared computers (§4.1) |
| Background work | BGAppRefresh, background pushes | No Background Sync on Safari or Firefox, so sync only runs while a tab is open (§2.4) |
| Versions in the wild | TestFlight builds (90 days) | Adds stale tabs and cached app shells; deploys are instant (§3.3) |
| Hosting | A public VPS serving the association file for native passkeys (superseded by ADR 0003 and native-apps-auth.md) | Adds static assets, one origin that is also the RP ID, and security headers, all on the tailnet; nothing has to be public; every browser needs Tailscale (§5) |
| Push | APNs | Adds Web Push (§5.4) |

## 2. Browser storage

### 2.1 Engines

| | SQLite WASM (official) | wa-sqlite | PGlite | IndexedDB directly (Dexie, RxDB) |
|---|---|---|---|---|
| Version | 3.54.1, 2026-10-16 ([changes](https://sqlite.org/changes.html)); npm [3.54.1-build1](https://www.npmjs.com/package/@sqlite.org/sqlite-wasm) | [1.2.0](https://github.com/rhashimoto/wa-sqlite/releases), 2026-11-03, MIT; PowerSync fork [2.0.10](https://www.npmjs.com/package/@journeyapps/wa-sqlite) | [0.5.15](https://github.com/electric-sql/pglite/releases), 2026-11-03 | Dexie 4.4.7, RxDB 17.6 (npm) |
| OPFS storage | `opfs-sahpool`: no COOP/COEP, "easily the highest OPFS performance", but a single connection; pause/unpause hands it over. `opfs` and `opfs-wl` (3.53+) need `SharedArrayBuffer`, so COOP/COEP headers ([persistence](https://sqlite.org/wasm/doc/trunk/persistence.md)) | `OPFSCoopSyncVFS`: multiple connections, worker only. `AccessHandlePoolVFS`: single connection. `OPFSWriteAheadVFS` needs Chromium's `readwrite-unsafe` mode ([VFS table](https://github.com/rhashimoto/wa-sqlite/blob/master/src/examples/README.md)) | `opfs-ahp` is blocked on Safari: "a limit of 252 open sync access handles", while Postgres has more than 300 files ([filesystems](https://pglite.dev/docs/filesystems)) | n/a |
| IndexedDB storage | `kvvfs`: localStorage only, about 5 MB | `IDBBatchAtomicVFS` and `IDBMirrorVFS`, in every context | `idb://` flushes whole files after each query | Native |
| Wasm size, uncompressed ([jsDelivr](https://www.jsdelivr.com/package/npm/@sqlite.org/sqlite-wasm)) | `sqlite3.wasm` 0.88 MB | 0.74–1.3 MB sync/JSPI builds; 2.3–2.5 MB Asyncify | 9.1 MB wasm + 6.2 MB data | 0 |
| Same SQL as the apps' SQLite | Yes | Yes | No (Postgres) | No |
| Encryption | Not in the canonical build. [SQLite3 Multiple Ciphers](https://github.com/utelle/SQLite3MultipleCiphers) (SQLite3MC, MIT) [2.6.1](https://github.com/utelle/SQLite3MultipleCiphers/releases/tag/v2.6.1) ships the same JS API, `opfs-sahpool` included, built on SQLite 3.54.0; an open rollback bug (§4.5) | PowerSync's fork builds `mc-wa-sqlite` with SQLite3MC 2.3.3, ChaCha20 by default ([Makefile](https://github.com/powersync-ja/wa-sqlite/blob/master/Makefile), [PowerSync](https://docs.powersync.com/client-sdks/advanced/data-encryption)); same bug reported (§4.5) | No | Application-level only |
| Verdict | **Pick for a custom engine** | **Comes with PowerSync** | ✗ size, Safari, no Swift twin | ✗ no shared SQL |

- **Durability caveats.**
  - SQLite's docs warn: "Users sometimes report that their OPFS databases randomly disappear" ([persistence](https://sqlite.org/wasm/doc/trunk/persistence.md)).
  - `opfs-sahpool` wipes its storage when initialization fails, unless `preserveOnInitFailure` (3.54+) is set (same page).
  - Chrome 121 made IndexedDB's `relaxed` durability the default, "to align with other major browsers". `complete` then fires once the data reaches the OS buffer, "typically flushed every couple seconds" ([Chrome](https://developer.chrome.com/blog/indexeddb-durability-mode-now-defaults-to-relaxed)). An outbox kept in IndexedDB should request `strict` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase/transaction)).
- **Cold start.** Notion found that loading "a few hundred kilobytes" of WASM up front slowed page loads. It loads SQLite asynchronously and races it against the API ([Notion](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite), first-party, 2024-07).

**Conclusion:** SQLite WASM on OPFS keeps one schema, the same migrations and the same SQL on the iPhone (`expo-sqlite`), in Swift extensions (GRDB) and on the web. **Medium-high.**

### 2.2 Quotas, eviction and persistence

| Context | Quota | Eviction | `navigator.storage.persist()` |
|---|---|---|---|
| Safari tab (macOS 14+, iOS 17+) | Up to ~60% of disk per origin; all origins together up to 80% ([WebKit](https://webkit.org/blog/14403/updates-to-storage-policy/)) | LRU under storage pressure. ITP deletes IndexedDB, localStorage, service worker registrations and caches "after seven days of Safari use without user interaction on the site" ([WebKit 2020](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/), [ITP](https://webkit.org/tracking-prevention/)). OPFS goes too: the removal set includes WebKit's file-system storage, which holds OPFS ([WebKit source](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/Classifier/WebResourceLoadStatisticsStore.cpp)). Server-set cookies stay ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)). ITP skips domains "with WebPush that the user interacts with continuously" ([WebKit source](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/Classifier/ResourceLoadStatisticsStore.cpp)) | Granted by "heuristics like whether the website is opened as a Home Screen Web App" ([WebKit](https://webkit.org/blog/14403/updates-to-storage-policy/)). Expect a denial in a plain tab (judgment) |
| Safari web app (iOS Home Screen, macOS Dock) | Same as the browser ([WebKit](https://webkit.org/blog/14403/updates-to-storage-policy/)) | The Home Screen app's domain is "exempt from ITP's 7-day cap", and its data is "kept isolated from Safari" ([ITP](https://webkit.org/tracking-prevention/)). On the Dock, Safari copies cookies at install but "does not copy over any other kind of local storage" ([Safari 17](https://webkit.org/blog/14445/webkit-features-in-safari-17-0/)) | Likely granted, by the heuristic above (not verified) |
| Chrome, Edge | Up to 60% of disk per origin ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)) | LRU for best-effort origins | Granted automatically for engaged, installed or bookmarked sites, or sites with notification permission ([web.dev](https://web.dev/articles/persistent-storage)) |
| Firefox | Best-effort: 10% of disk or 10 GiB per site group, whichever is smaller; persistent: up to 50% ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)) | LRU | The user is prompted |
| Private windows | Reduced. PowerSync lists "OPFS not supported in private tabs" for Safari and Firefox ([PowerSync](https://docs.powersync.com/resources/supported-platforms)) | Wiped when the window closes | — |

- **All or nothing.** Eviction deletes all of an origin's data at once, OPFS and IndexedDB together ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)).
- **iOS 26.** "By default, every website added to the Home Screen opens as a web app", with no manifest required ([Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)).
- **EU.** Home Screen web apps still work in the EU: Apple dropped its plan to remove them in iOS 17.4 on 2024-03-01 (secondary, [MacRumors](https://www.macrumors.com/2024/03/01/apple-walks-back-decision-to-disable-eu-web-apps/)).

**Conclusion:** In a Safari tab the database is volatile. Installed, it behaves like an app's. So:

- the server is the only truth;
- the outbox is pushed within seconds;
- the app calls `persist()` after install and sign-in;
- the app prompts Safari users to Add to Dock or Home Screen;
- a signed-in reopen with an empty database re-bootstraps and tells the user that the browser cleared local data (judgment).

**High** on the policies, **medium** on the `persist()` outcomes.

### 2.3 Multi-tab coordination

- **Primitives** ([BCD](https://github.com/mdn/browser-compat-data/blob/main/api/SharedWorker.json)):
  - Web Locks and BroadcastChannel: Safari 15.4.
  - SharedWorker: Safari and iOS 16, but Chrome Android only from 148.
  - OPFS sync access handles: dedicated workers only, Safari 15.2.
  - The multi-handle `readwrite-unsafe` mode: Chrome 121 only.
- **Notion** (first-party, 2024-07):
  - It chose `opfs-sahpool` because "it didn't have the requirement of cross-origin isolation".
  - Concurrent tabs corrupted some users' databases. The fix was one "active tab" managed by a SharedWorker, with "an infinitely-open Web Lock on each tab" to detect closed tabs.
  - Navigation got 20% faster ([Notion](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite)).
- **PowerSync web** ([docs](https://docs.powersync.com/client-sdk-references/javascript-web)):
  - It uses shared workers, but they are "disabled by default on Android, iOS, and Safari". There, a broadcast fallback lets only one tab sync at a time, and "updates may not reach every tab".
  - It recommends `OPFSCoopSyncVFS` for Safari.
- **PGlite** elects one leader tab that runs Postgres, and the other tabs proxy to it ([PGlite](https://pglite.dev/docs/multi-tab-worker)).
- **For on-budget (judgment).** One leader worker per profile, holding a Web Lock, owns:
  - the SQLite connection;
  - the outbox;
  - the poke stream.

  Other tabs send queries and mutations over BroadcastChannel and receive "tables changed" events. The `clientId` lives in the database, not in the tab, so a leader change mid-push stays idempotent. The leader opens only the signed-in member's pool and the active household's database (§4.6).
- **Handover rules, mandatory with `opfs-sahpool`** ([persistence](https://sqlite.org/wasm/doc/trunk/persistence.md), [source](https://github.com/sqlite/sqlite/blob/master/ext/wasm/api/sqlite3-vfs-opfs-sahpool.c-pp.js)):
  - Install it with `preserveOnInitFailure: true` (3.54+). Init fails while another context still holds the access handles, and by default a failed init calls `removeVfs()`, which tries to delete the pool.
  - Only the Web Lock holder calls `installOpfsSAHPoolVfs()`, because two concurrent installs collide.
  - A leader that steps down closes its databases and calls `pauseVfs()`. The next leader retries `unpauseVfs()`, or init with `forceReinitIfPreviouslyFailed`, until it gets the handles (judgment).
  - PowerSync's `OPFSCoopSyncVFS` path doesn't use `opfs-sahpool`.

**Medium-high.**

### 2.4 Service worker and installation

- **Service worker.** Cache only the app shell (hashed JS, CSS and wasm). Data lives in SQLite. Never put API responses in Cache Storage, because they would outlive logout (judgment).
- **Updates.** A new service worker waits until the old one is released. The leader migrates the database, then broadcasts "reload" to stale tabs (judgment).
- **No background sync.** Background Sync and Periodic Sync exist only in Chromium ([BCD](https://github.com/mdn/browser-compat-data/blob/main/api/SyncManager.json)), so the web client syncs only while open.
- **macOS.** From Safari 17, any site can be added to the Dock. It gets its own window, push, badging and service workers ([Safari 17](https://webkit.org/blog/14445/webkit-features-in-safari-17-0/)).
- **iOS.**
  - Web Push works only for Home Screen web apps (iOS 16.4+), and permission must be requested "in response to direct user interaction" ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)).
  - Declarative Web Push (iOS 18.4, macOS 15.5) needs no service worker ([WebKit](https://webkit.org/blog/16535/meet-declarative-web-push/)).

## 3. Sync: one protocol for every client

### 3.1 Options

Versions are from GitHub releases or npm on 2026-11-07. "Validated writes" means your server can enforce domain rules on every write.

| Option | Web client | Swift client | Offline writes | Validated writes | Status, licence | Verdict |
|---|---|---|---|---|---|---|
| **PowerSync** | `@powersync/web` [2.6.0](https://github.com/powersync-ja/powersync-js/releases): wa-sqlite, IndexedDB or OPFS VFS, multi-tab, encryption. React Native 2.4.0, including RN-web ([platforms](https://docs.powersync.com/resources/supported-platforms)) | [1.17.0](https://github.com/powersync-ja/powersync-swift/releases): iOS 15+, macOS 12+, watchOS 9+, HTTP only; GRDB integration is alpha ([sync-engine.md](sync-engine.md)) | Yes. Through an upload queue; "the client never has to resolve conflicts locally" ([consistency](https://docs.powersync.com/architecture/consistency)) | Yes. `uploadData()` "should call your backend application API" ([client](https://docs.powersync.com/architecture/client-architecture)) | Service [1.28.1](https://github.com/powersync-ja/powersync-service/releases). The service moved from FSL to Apache-2.0 in [v1.28.0](https://github.com/powersync-ja/powersync-service/releases/tag/v1.28.0) on 2026-11-03 ([PR 878](https://github.com/powersync-ja/powersync-service/pull/878)); the [JS](https://github.com/powersync-ja/powersync-js/commits/main/LICENSE) and [Swift](https://github.com/powersync-ja/powersync-swift/commits/main/LICENSE) SDKs were Apache-2.0 from their first commits (2023-10, 2024-10). Sync Streams GA since 2026-05-14, Sync Rules now "legacy" ([announcement](https://releases.powersync.com/announcements/ann_7UgfQuNwRmpQk)). One more service, plus bucket storage | **~ adopt only if Spike A + W2 show less code, all costs counted (§3.2)** |
| **Custom protocol** ([sync-engine.md](sync-engine.md)) | Yours: the shared TypeScript client in a worker + SQLite WASM | Yours, only if Swift extensions write locally (GRDB) | Yes | Yes | Just your API and Postgres | **✓ default (build)** |
| ElectricSQL | [`@electric-sql/client`](https://www.npmjs.com/package/@electric-sql/client) 1.5.28; TanStack DB 1.0 RC | None official; the HTTP shape API is small | Do it yourself: Electric "does not do write-path sync" ([writes](https://electric.ax/docs/guides/writes)) | Through your API | Apache-2.0 | ~ read path only |
| Zero | [1.11.4](https://www.npmjs.com/package/@rocicorp/zero): web, React Native | None | **No**: "Zero does not support offline writes", and it is "not designed for long periods offline" ([offline](https://zero.rocicorp.dev/docs/offline)) | — | Apache-2.0 | ✗ |
| LiveStore | [0.5.0](https://github.com/livestorejs/livestore/releases): web, Expo, Node, Electron, Tauri | None | Yes (event log) | Pluggable backends | Apache-2.0; labelled "BETA" ([site](https://livestore.dev/)) | ✗ |
| Jazz | [2.0.0-alpha.60](https://www.npmjs.com/package/jazz-tools): Rust core compiled to Wasm | None | Yes | "Core authorizes every write" ([site](https://jazz.tools/)) | MIT; v2 alpha | ✗ |
| InstantDB | [1.0.67](https://www.npmjs.com/package/@instantdb/core): web, React Native | None | — | — | "Instant is sunsetting. Services will continue until August 31st, 2027" ([docs](https://www.instantdb.com/docs)) | ✗ |
| Triplit | 1.0.50 (2025-07) | None | Yes | — | AGPL-3.0. The co-founder joined Supabase on 2025-10-08 ([Supabase](https://supabase.com/blog/triplit-joins-supabase)); repo idle since 2026-01 ([repo](https://github.com/aspen-cloud/triplit)) | ✗ |
| Automerge | [3.5.0](https://github.com/automerge/automerge/releases) (Wasm) | [0.7.2](https://github.com/automerge/automerge-swift/releases) (2025-12) | Yes (CRDT) | The server must merge documents | MIT | ✗ for the ledger ([sync-engine.md](sync-engine.md)) |
| Loro, Yjs | [loro-crdt 1.17.1](https://github.com/loro-dev/loro/releases); yjs 13.6.33 | [loro-swift 1.17.1](https://github.com/loro-dev/loro-swift); [yswift 0.2.1](https://github.com/y-crdt/yswift) (2024, stale) | Yes (CRDT) | The server must merge | MIT | ✗ for the ledger; fine for rich-text notes |
| sqlite-sync | [1.2.2](https://github.com/sqliteai/sqlite-sync) (Wasm) | SQLite extension | Yes (CRDT) | Via the vendor's CloudSync | Apache-2.0 | ✗ ([sync-engine.md](sync-engine.md)) |

### 3.2 What the web changes in sync-engine.md's call

- **The decision stands.** sync-engine.md decides "build, unless Spike A shows PowerSync fits with less code" (medium), with the server-authoritative protocol in force under ADR 0002 (its status note). The web doesn't flip that default; it adds browser plumbing to what Spike A must weigh.
- **One client, two storage backends.** ADR 0002 gives the iPhone, Mac and web one UI codebase, so the custom client is written once, in TypeScript, over native SQLite in the apps and SQLite WASM in a worker on the web (§3.4). On top of the outbox, field guards, HLC and pull/apply, the web adds:
  - the bootstrap, plus re-bootstrap after eviction;
  - leader election and fan-out to tabs;
  - the browser's storage and VFS choice.

  A Swift client is needed only if Swift extensions (widgets, App Intents, the notification service extension) write to the local database. They can read shared SQL views or a glance snapshot instead (judgment).
- **No port with a TypeScript server.** The server's mutators must match the clients'. A TypeScript server shares them (§3.5); a Swift one would keep a port checked by golden vectors (§3.3, point 9).
- **What PowerSync takes on.** It already provides the read path on every candidate platform:
  - checkpoints and buckets;
  - the choice of VFS, multi-tab support and encryption;
  - SDKs for React Native (iOS, Expo and RN-web, but react-native-macos isn't listed), Capacitor (iOS, web, Electron), Flutter (iOS, macOS, web) and Swift; Tauri on iOS is untested ([platforms](https://docs.powersync.com/resources/supported-platforms)).

  The write path stays yours, which suits sync-engine.md's semantics: named operations plus LWW fields.
- **PowerSync's costs**, on top of sync-engine.md's (telemetry opt-out, no self-hosted dashboard, bucket storage on format version 2):
  - **A second hostname.** PowerSync: "It is currently required to host the API container on a dedicated subdomain - we do not support running it on the same subdomain as another service", behind a TLS load balancer ([deployment](https://docs.powersync.com/maintenance-ops/self-hosting/deployment-architecture)). So every instance gets a second hostname and certificate (self-hosting.md P7), and the browser reaches it cross-origin, with CORS and a `connect-src` entry.
  - **A JWT in JavaScript.** The browser then holds a short-lived JWT from `fetchCredentials()` ([auth](https://docs.powersync.com/configuration/auth/overview)), a bearer token readable by JavaScript. By analogy, that is RFC 10017's token-mediating backend, which the RFC says to choose only "when the use cases or system requirements would prevent the use of a proxying BFF" (§6.2.4.4). Keep it at 5 minutes or less. A same-origin proxy that swaps a placeholder token for a server-minted one would avoid it: the SDK builds URLs as `endpoint + path`, refuses an empty token and sends `Authorization: Token …` over HTTP, while WebSocket sync carries the token inside its RSocket setup metadata ([AbstractRemote.ts](https://github.com/powersync-ja/powersync-js/blob/main/packages/shared-internals/src/client/sync/stream/AbstractRemote.ts), [WebSocketSupport.ts](https://github.com/powersync-ja/powersync-js/blob/main/packages/shared-internals/src/client/sync/stream/WebSocketSupport.ts)). But such a proxy needs the HTTP connection method and is a layout PowerSync says it doesn't support (judgment).
  - **Packaging under ADR 0003.** The service ships as the `journeyapps/powersync-service` image, built from a Node 24 pnpm monorepo ([Dockerfile](https://github.com/powersync-ja/powersync-service/blob/main/service/Dockerfile)), and nixpkgs has no package for it. A NixOS deployment without Docker either builds it from source with `fetchPnpmDeps`, whose hash changes with every upgrade (self-hosting.md §2.3), or runs the image under `oci-containers` with `dockerTools.pullImage`, as nixos-vps-infra.md plans.
  - **A second authorization layer.** The replication role is `CREATE ROLE powersync_role WITH REPLICATION BYPASSRLS LOGIN …` ([setup](https://docs.powersync.com/configuration/source-db/setup)), and `BYPASSRLS` "bypasses every row-level security (RLS) policy" ([PostgreSQL](https://www.postgresql.org/docs/current/sql-createrole.html)). Household isolation of synced data then rests on Sync Streams queries and JWT claims, beside the API's own checks: "The user ID comes from the `sub` claim in their JWT token", and memberships come from subqueries such as `SELECT org_id FROM org_members WHERE user_id = auth.user_id()` ([queries](https://docs.powersync.com/sync/streams/queries)). Mitigations (judgment): a JWT carrying only `sub`; memberships replicated and joined in every stream; grants and the `powersync` publication limited to synced tables, which the setup page also advises, so provider keys and sessions never replicate; and a CI test that a member of household A never receives a row of household B.
  - **Uploads block downloads.** "While mutations are present in the upload queue, the client does not advance to a new checkpoint", and an error caused by a bug "has to be fixed before the changes from the client can be processed" ([consistency](https://docs.powersync.com/architecture/consistency), [write errors](https://docs.powersync.com/handling-writes/handling-write-validation-errors)). Version skew is permanent (§3.3, point 4), so the API answers a retired mutation version with a 2xx rejection. Otherwise a stale tab stops receiving data.
  - Encryption doesn't tip the balance: PowerSync's web codec is the same SQLite3MC, and the same open bug was reported on its build (§4.5).

**So:** keep building as the default. Spike A stays decisive, and W2 adds the web client and these costs to it. **Medium.**

### 3.3 Constraints the web puts on the protocol

1. **Transport: HTTPS, JSON or NDJSON, plus SSE or WebSocket.**
   - The browser `EventSource` and `WebSocket` constructors take no custom headers ([HTML](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface), [WebSockets](https://websockets.spec.whatwg.org/#the-websocket-interface)). So the web authenticates live streams with the same-origin cookie, and the server checks `Origin` on the WebSocket handshake ([RFC 6455 §10.2](https://www.rfc-editor.org/rfc/rfc6455#section-10.2)).
   - Without HTTP/2, SSE streams count against a small per-domain connection limit ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). So only the leader holds the stream, over HTTP/2.
2. **A client ID can vanish.** Eviction takes the database and the outbox with it. The server must accept a new `clientId` at any time and garbage-collect old ones. Clients push eagerly and show a "not yet synced" badge (judgment).
3. **One writer per profile.** The idempotency key `(clientId, mutationId)` from sync-engine.md holds only if the leader worker owns the database (§2.3).
4. **Version skew is permanent.**
   - Stale tabs and cached app shells run old code while deploys are instant, and TestFlight builds lag.
   - Keep sync-engine.md's versioned mutations (`moveMoney@1`).
   - Add a server-sent `minClientVersion` that triggers a reload on the web and an update screen in the apps.
   - Client migrations must keep the outbox.
   - With PowerSync, the API acknowledges a retired mutation version with a 2xx rejection (§3.2).
5. **Shared schema.** Keep the SQLite DDL and migrations as plain `.sql` files, and define derived numbers (left to spend, category totals) as SQL views. They then give the same results in `expo-sqlite`, SQLite WASM, GRDB and op-sqlite (judgment).
6. **Numbers.** JavaScript numbers lose integer precision beyond 2^53−1 ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER)). Amounts in int64 minor units are safe in practice, but send IDs and 64-bit counters as strings.
7. **Dates and regions.**
   - Calendar dates travel as `YYYY-MM-DD`, instants as UTC RFC 3339, the user's zone as an IANA ID and currencies as ISO 4217 codes.
   - Only clients format values. Browsers have `Intl`. Hermes's docs call `Intl` "Not supported" and advise "a polyfill such as FormatJS" ([Hermes](https://github.com/facebook/hermes/blob/static_h/doc/Features.md)); React Native's builds enable a partial, Foundation-backed `Intl` on iOS, without `formatToParts`, `signDisplay`, `PluralRules` or `RelativeTimeFormat` (i18n-l10n.md §2.2, cross-platform-ui-stack.md §4.6). So the iPhone app loads FormatJS polyfills. The SwiftUI watch and widgets use Foundation. Golden tests across platforms keep European formats aligned ([i18n-l10n.md](i18n-l10n.md)).
   - `Temporal` ships in Chrome 144 and Firefox 139 ([BCD](https://github.com/mdn/browser-compat-data/blob/main/javascript/builtins/Temporal/ZonedDateTime.json)), but released Safari and Hermes lack it (i18n-l10n.md, TL;DR and §4). Load `temporal-polyfill` where it is missing, or keep date math in SQL and on the server (judgment).
8. **Authorization scopes are sync scopes** (sync-engine.md's spaces). The web receives exactly what an app would. No personal data goes in URLs, and data responses send `Cache-Control: no-store` (judgment).
9. **Mutators live in one language when the server is TypeScript** (§3.5): one package, imported by the server and the shared client. Keep golden vectors in the repo anyway: they pin the glance JSON that the Swift watch decodes (watchos-app.md), and they become mandatory if the server ever moves to another language.

### 3.4 The client stack under Expo

- **Decided.** [cross-platform-ui-stack.md](cross-platform-ui-stack.md) picks its strategy B under ADR 0002: Expo SDK 58 (React Native 0.88) for the iPhone, web and Mac, react-native-web through Expo Router, Unistyles for styling, a committed `ios/` project and a SwiftUI watch. Its Swift `Security` module refreshes tokens natively, so JavaScript sees only access tokens (its §4.8).
- **Avoid.** TypeScript-only engines, on status rather than language: Zero has no offline writes, Instant is sunsetting, LiveStore is beta, Jazz is alpha and Triplit is idle (§3.1). CloudKit and SQLiteData sync, which sync-engine.md rejects anyway.
- **The watch doesn't constrain the engine.** watchos-app.md concludes "Don't run the sync engine on the watch": watchOS counts `URLSessionWebSocketTask` as low-level networking, which it blocks for ordinary apps ([TN3135](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos)), and the watch reads a server-built glance. PowerSync's watchOS 9+ support is therefore moot.
- **SQLite bindings** (as of November 2026):

  | | iPhone (and react-native-macos later) | Web |
  |---|---|---|
  | **Pick** | `expo-sqlite` ~58.0.15, which ships with the SDK, plus `useSQLCipher`. If the binary lacks SQLCipher, "`PRAGMA key` has no effect and does not throw", so a test checks `PRAGMA cipher_version` ([Expo](https://docs.expo.dev/versions/latest/sdk/sqlite/), cross-platform-ui-stack.md §4.8). If PowerSync is adopted, op-sqlite [19.0.4](https://github.com/OP-Engineering/op-sqlite/releases) (MIT), which its React Native SDK requires ([PowerSync](https://docs.powersync.com/client-sdk-references/react-native-and-expo)) | The official SQLite WASM 3.54 (or SQLite3MC's build, §4.5) inside the leader worker of §2.3 |
  | **Rejected on the web** | — | `expo-sqlite`: every tab starts its own worker on wa-sqlite's `AccessHandlePoolVFS`, with no cross-tab coordination, and its synchronous API spins on a `SharedArrayBuffer`, hence COOP/COEP ([worker.ts](https://github.com/expo/expo/blob/main/packages/expo-sqlite/web/worker.ts), [WorkerChannel.ts](https://github.com/expo/expo/blob/main/packages/expo-sqlite/web/WorkerChannel.ts)). op-sqlite: async-only on the `opfs` VFS behind COOP/COEP, and "SQLCipher, libsql and turso are not supported on web" ([docs](https://github.com/OP-Engineering/op-sqlite/blob/main/docs/docs/installation.md)) |

- **One TypeScript sync client, two hosts** (design judgment on cited facts; **medium**):
  - `packages/sync-core` holds the outbox, pull/apply, rebase, HLC, mutators and golden vectors, with no DOM or React Native import. It reaches the world through injected ports: an async `SqlDriver` (`exec`, `query`, `transaction`), a `Transport` (requests and the poke stream), an `Auth` (cookie on the web; access token and DPoP proof from the `Security` module on iOS), and a `Clock`.
  - **Web:** the core runs inside the leader worker, over SQLite WASM's synchronous API wrapped in the async driver. Tabs call it over BroadcastChannel and receive "tables changed" events from SQLite's update hook. Expo's Metro bundles workers written as `new Worker(new URL('./worker', window.location.href))`, but the feature "is alpha and subject to breaking changes" and needs bundle splitting ([Metro config](https://docs.expo.dev/versions/latest/config/metro/)).
  - **iPhone:** React Native has no Web Workers; its renderer uses a JS thread and a UI thread ([threading model](https://reactnative.dev/architecture/threading-model)). The core runs on the JS thread and uses only the binding's async API. op-sqlite runs those calls "on a (**single**) separate and dedicated thread" ([API](https://github.com/OP-Engineering/op-sqlite/blob/main/docs/docs/api.md)); Expo documents only that its synchronous calls block the JS thread, so W7 measures where its async calls run.
  - **Transport:** `expo/fetch`, a WinterCG fetch that becomes the global fetch on iOS and Android, streams response bodies ([Expo](https://docs.expo.dev/versions/latest/sdk/expo/)). So one fetch-based reader handles the NDJSON bootstrap and an SSE-format poke stream on both hosts. React Native's `WebSocket` also takes a non-standard `headers` option ([source](https://github.com/facebook/react-native/blob/main/packages/react-native/Libraries/WebSocket/WebSocket.js)), which browsers lack (§3.3, point 1).
  - **Background:** the iPhone syncs while foregrounded and in the system's refresh windows (native-apps-auth.md §7); the web only while a tab is open (§2.4).
- **Does the strict CSP survive?** On paper yes, with three adjustments (**medium**; W4 checks each one).
  - **Runtime styles.** react-native-web inserts an empty `<style id="react-native-stylesheet">` and fills it through `insertRule` ([createCSSStyleSheet.js](https://github.com/necolas/react-native-web/blob/master/packages/react-native-web/src/exports/StyleSheet/dom/createCSSStyleSheet.js)). Unistyles does the same with `<style id="unistyles-web">`, "through the CSSOM, never its text" ([sheet.ts](https://github.com/jpudysz/react-native-unistyles/blob/main/packages/unistyles/src/web/css/sheet.ts)). Neither sets a nonce. Under `style-src 'self'` alone, that empty element counts as blocked inline style (Firefox logs the violation, [bug 1706787](https://bugzil.la/1706787)), which leaves no sheet to fill. Adding the hash of the empty string, `'sha256-47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU='`, admits it: WebKit bug 233071's test page does exactly this with `insertRule`, which works in Chromium and Gecko and in WebKit since a 2022 fix ([WebKit](https://bugs.webkit.org/show_bug.cgi?id=233071)). `insertRule` itself passes: "no browser currently blocks these methods", nor styles set through an element's `style` properties, which React uses ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/style-src)).
  - **The HTML shell.** Expo's default `index.html` carries an inline `<style id="expo-reset">` ([template](https://github.com/expo/expo/blob/main/packages/@expo/cli/static/template/index.html)): hash it at build time, or move it into a file through a custom template. Prefer `web.output: "single"`: `static` pre-renders style contents and `style="…"` attributes into every page, each needing a hash and `'unsafe-hashes'` (judgment).
  - **Chunks and workers.** In production, Expo loads split bundles by creating `<script>` elements and setting `script.src` ([fetchThenEval.web.ts](https://github.com/expo/expo/blob/main/packages/expo/src/async-require/fetchThenEval.web.ts)). `'strict-dynamic'` admits them, and Trusted Types sends that `script.src` and every `Worker()` URL through the `default` policy. Development builds fetch and evaluate bundles, so the dev server needs a looser policy. When `crossOriginIsolated` is true, Expo starts workers from `blob:` URLs ([asyncRequireModule.ts](https://github.com/expo/expo/blob/main/packages/expo/src/async-require/asyncRequireModule.ts)), which `worker-src 'self'` blocks. `opfs-sahpool` needs no `SharedArrayBuffer`, so send COOP without COEP (judgment).
- **Design system.**
  - Define design tokens once in the DTCG format, whose first stable version is 2025.10 ([W3C CG](https://www.w3.org/community/design-tokens/2025/10/28/design-tokens-specification-reaches-first-stable-version/)). Generate them for the shared UI (CSS custom properties or theme objects, light and dark) and for Swift.
  - Each component is written once in the shared codebase. The watch and widget subset is built again in SwiftUI ([cross-platform-ui-stack.md](cross-platform-ui-stack.md) §1, [design-system.md](design-system.md)).

### 3.5 Backend language

apple-client-stack.md §4.1 picked Swift with Hummingbird before ADR 0002, and noted that with a non-Swift shared UI its shared-package argument "shrinks to the watch app". cross-platform-ui-stack.md leaves the question open (its question 8). Under B, the server's language decides three ports and some platform costs:

| | TypeScript, Node 26 LTS | Swift 6.4, Hummingbird 2.28 |
|---|---|---|
| Mutators, validation, HLC | One package, imported by the server and the shared client | Ported to TypeScript for the client, checked by golden vectors |
| GlanceBuilder, server-side, plus the phone's optional local rebuild (watchos-app.md) | One function for both | Swift on the server, a TypeScript port on the phone held to the server's golden JSON |
| Authorization server (native-apps-auth.md, Recommendation 10) | [`oidc-provider`](https://github.com/panva/node-oidc-provider) 9.13.3 (MIT): PKCE, DPoP with `dpop_jkt` ([source](https://github.com/panva/node-oidc-provider/blob/main/lib/actions/authorization/sender_constraints.js)), device grant, RFC 8414; SimpleWebAuthn 14.1.0 | Hand-rolled, on swift-webauthn 1.0.0-beta.2 |
| Jobs and push | [pg-boss](https://www.npmjs.com/package/pg-boss) 12.38.1 or [graphile-worker](https://www.npmjs.com/package/graphile-worker) 0.18.0, both on Postgres; [web-push](https://www.npmjs.com/package/web-push) 4.0.0; [@parse/node-apn](https://www.npmjs.com/package/@parse/node-apn) 8.2.0 | swift-jobs 1.4 on Postgres, APNSwift 8 (apple-client-stack.md) |
| Nix packaging (self-hosting.md §2.3) | `buildNpmPackage`; `importNpmLock` needs no hash, `fetchPnpmDeps` does | Swift 6 needs nixpkgs 26.11, due 2026-11-30, or a Static Linux SDK binary |
| Shared with the watch | The glance JSON contract | The glance codec, which the watch only decodes |

Versions are from npm and GitHub releases on 2026-11-07. Node 26 became LTS on 2026-10-28 and is supported until 2029-04-30 ([schedule](https://github.com/nodejs/Release/blob/main/schedule.json)).

**Recommendation: TypeScript on Node 26 LTS** (medium). It removes the mutator port, the GlanceBuilder port and the hand-rolled authorization server at once; Swift would keep only a codec the watch decodes anyway. Take Hono and Drizzle from apple-client-stack.md §4.1's TypeScript column at their current majors (4.13 and 0.45), since 5.0 and 1.0 are still release candidates on npm. **The gate** is cross-platform-ui-stack.md's spike S2: if B, or its TypeScript fallback E, holds, record this choice in an ADR. Only a reversal of ADR 0002 toward SwiftUI reopens Swift. native-apps-auth.md's S2 then proves `oidc-provider` with Secure Enclave DPoP keys.

## 4. Web security for financial data

### 4.1 What is different on the web

- **XSS means full compromise while the page is open.** "malicious JavaScript code has the same privileges as the legitimate application code" ([RFC 10017](https://www.rfc-editor.org/rfc/rfc10017.html)). Even with sender-constrained tokens, XSS "can produce DPoP proofs", and there is "most likely no feasible defense against this threat except generally preventing XSS" ([RFC 9449](https://www.rfc-editor.org/rfc/rfc9449)).
- **Untrusted text arrives through the bank.** Bank labels and payee names are written by whoever sends you money, so render them as text only (judgment).
- **Extensions.** Content scripts "are able to read details of the web pages the browser visits", and in their isolated worlds the page's CSP doesn't apply ([Chrome](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts)).
- **Missing platform protection.** The page gets no Secure Enclave, Keychain or Data Protection API. The npm supply chain is a risk, and so are shared or work computers.

### 4.2 Sign-in

- **The RP ID is the exact instance host** (native-apps-auth.md §9, self-hosting.md §2.4, where `public_url` is "also the passkey RP ID").
  - The RP ID must equal "the origin's effective domain" or "a registrable domain suffix" of it ([WebAuthn L3](https://www.w3.org/TR/webauthn-3/#rp-id), a W3C Recommendation since 2026-08-25). A parent suffix such as `example.org` would let sibling apps on `*.example.org` start ceremonies with on-budget's passkeys; the exact host doesn't.
  - The web app, the API and the OAuth authorize page share that origin. Native apps sign in on the same page inside `ASWebAuthenticationSession`, where it runs as the top-level site, and return through a custom-scheme callback: no association file and no entitlement (native-apps-auth.md TL;DR 1–2). So the server checks one exact origin, which even swift-webauthn's single `relyingPartyOrigin` would handle (apple-client-stack.md §3.2).
  - Only the optional per-build native passkey sheet needs `webcredentials:<host>` and the association file public at that exact host, which a Tailscale-only instance gets through split-horizon DNS. native-apps-auth.md §9 keeps a parent name used only by on-budget as the alternative for that sheet (its question 6); this note follows its default.
  - **It can never change.** Credentials are scoped to the RP ID, so a new hostname strands every passkey. Related Origin Requests only let a new origin keep using the old RP ID while the old host still resolves and serves `/.well-known/webauthn` listing it ([L3 §5.11](https://www.w3.org/TR/webauthn-3/#sctn-related-origins)): an escape hatch, not a migration. Safari 18 ([WebKit](https://webkit.org/blog/15865/webkit-features-in-safari-18-0/)), Chrome 128 ([Chrome](https://developer.chrome.com/blog/passkeys-updates-chrome-129) covers 128 and 129; the exact version is from [passkeys.dev](https://passkeys.dev/device-support/), secondary) and Firefox 152 ([bug 2010193](https://bugzilla.mozilla.org/show_bug.cgi?id=2010193)) ship it. Firefox asks the user first ([prefs](https://github.com/mozilla-firefox/firefox/blob/main/modules/libpref/init/StaticPrefList.yaml)) and temporarily refuses it with passkey autofill ([bug 2043449](https://bugzilla.mozilla.org/show_bug.cgi?id=2043449)).
  - A `*.ts.net` host is tied to its tailnet and machine name, and renaming either changes it (native-apps-auth.md §6). A name under the owner's own domain survives network changes.
- **Safari support** ([BCD](https://github.com/mdn/browser-compat-data/blob/main/api/PublicKeyCredential.json)):
  - passkey autofill (conditional UI): Safari 16;
  - automatic passkey upgrade (conditional create): Safari 18;
  - JSON option helpers: Safari 18.4;
  - the Signal API, to prune deleted credentials: Safari 26.
- **Libraries.** Server: [SimpleWebAuthn](https://github.com/MasterKale/SimpleWebAuthn) 14.1.0 (2026-11-06), on the TypeScript server of §3.5. Browser: `@simplewebauthn/browser` 14.0.0.
- **Shared PCs.** Signing in with the phone's passkey over the WebAuthn hybrid (QR) flow leaves no credential on the PC (judgment).
- **Sign in with Apple is out.** On the web it needs website URLs registered in the publisher's account, at most 10 for an individual, so it can't follow arbitrary self-hosted hosts (native-apps-auth.md §2, §5).

### 4.3 Sessions

| Option | Where the secret lives | Can XSS steal it? | CSRF exposure | Use |
|---|---|---|---|---|
| **`__Host-` HttpOnly cookie, `Secure`, `SameSite=Strict`, `Path=/`, same origin** | Cookie jar, invisible to JavaScript | No. It can still use the session while the page is open | Yes. An exact `Origin` allow-list and a required static custom header ([RFC 10017 §6.1.3.3.2](https://www.rfc-editor.org/rfc/rfc10017.html)) are mandatory: sibling subdomains are the same site ([MDN](https://developer.mozilla.org/en-US/docs/Glossary/Site)), so `SameSite` doesn't stop a compromised sibling app (native-apps-auth.md §9) | **Web pick**. RFC 10017 sets these cookie rules for its BFF, "strongly recommended" for applications "that handle personal data". It covers OAuth browser clients; with no OAuth tokens here, it applies by analogy |
| Short-lived bearer token in memory | Worker memory | Yes, for minutes | No | Only the PowerSync JWT, if adopted: RFC 10017's token-mediating backend, by analogy (§3.2) |
| Bearer or refresh token in localStorage or IndexedDB | Disk | Yes, persistently | No | **Never** |
| DPoP with a non-extractable WebCrypto key | Key in IndexedDB | A stolen token alone is useless, but XSS signs in place ([RFC 9449](https://www.rfc-editor.org/rfc/rfc9449)) | No | Not worth it on the web |
| DBSC (device-bound cookies) | TPM on Windows, Secure Enclave on macOS | — | — | Chrome 147 on Windows and macOS, earlier versions Windows only ([BCD](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Secure-Session-Registration.json), [Google](https://blog.google/security/protecting-cookies-with-device-bound-session-credentials/)); no Firefox or Safari |

- **Cookie prefixes and SameSite.**
  - RFC 10017 suggests the `__Host-Http-` prefix, which its [COOKIES] reference defines: an Internet-Draft, [draft-ietf-httpbis-layered-cookies-02](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-layered-cookies-02). BCD lists it for Chrome 140 and Firefox 143, but not Safari; `__Host-` works everywhere (Safari 13+).
  - Safari doesn't default to `SameSite=Lax`, so set it explicitly ([BCD](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Set-Cookie.json)).
- **Bank consent returns (judgment).** The bank sends the browser back to the instance's `https` callback through a cross-site navigation, and a `Strict` cookie is sent only "for requests originating from the same site that set the cookie" ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)). So the callback authenticates by `state` alone, exchanges the code on the server (connectors-and-license.md), and lands on a page that navigates within the site, where the cookie applies again. Before Safari 27.2, cookies set by "a top-level navigation's own response" were capped at seven days when the hostname is CNAMEd to another registrable domain ([27.2](https://webkit.org/blog/18515/webkit-features-for-safari-27-2/)), so give `<host>` A/AAAA records rather than a CNAME, or set cookies only from same-origin `fetch` calls.
- **Lifetimes** (judgment):
  - On a trusted device, a 30-day sliding session. When it expires offline, local data stays readable, and the outbox is pushed after the user re-authenticates with a passkey.
  - On a shared device, a session cookie with no `Max-Age` and a 15-minute idle timeout.
  - Sensitive actions need a fresh passkey assertion, as in the apps.

### 4.4 CSP, XSS and supply chain

- **Policy.** Use a strict CSP in web.dev's sense: nonces or hashes plus `'strict-dynamic'`, with `object-src 'none'` and `base-uri 'none'`. For the static SPA, that means hashing a small inline loader. A host allowlist such as `script-src 'self'` "can be bypassed" and isn't strict by that definition ([web.dev](https://web.dev/articles/strict-csp)).
- **WebAssembly.** SQLite WASM needs `'wasm-unsafe-eval'`, which page CSP accepts since Chrome 97, Firefox 102 and Safari 16 ([BCD](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Content-Security-Policy.json)). Safari 26.0's fix for "wasm-unsafe-eval to not get parsed as a valid CSP keyword" sits under Web Extensions, so it concerns extension CSP, not pages ([Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)).
- **DOM sinks.** Use Trusted Types (`require-trusted-types-for 'script'`), now available in Chrome 83, Firefox 148 and Safari 26 ([BCD](https://github.com/mdn/browser-compat-data/blob/main/api/TrustedTypePolicyFactory.json)); they set the Safari 26 floor.
  - They also guard the URLs passed to `Worker()`, `SharedWorker()`, `importScripts()` and `ServiceWorkerContainer.register()` ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API)).
  - So the leader worker, the service worker, Expo's chunk loader (§3.4) and any worker that PowerSync or SQLite WASM creates need a policy. A narrow `default` policy that accepts only same-origin build assets covers library code (judgment).
- **Script integrity.** Use SRI. Safari 26 also enforces the `Integrity-Policy` header for scripts ([Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/)).
- **No third-party JavaScript on the app origin**: no analytics, chat widgets or Apple JS. Notion's third-party scripts made cross-origin isolation "an unrealistic ask" ([Notion](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite)).
- **Dependencies.** Pin versions and keep the lockfile, with guard rails as in [factory-ci-quality.md](factory-ci-quality.md).

### 4.5 Encrypting local data: what it protects

The WebCrypto spec "does not guarantee that the underlying cryptographic key material will not be persisted to disk, possibly unencrypted". It adds: "Any application or user that has access to the device storage may be able to recover the key material" ([WebCrypto §6.2](https://w3c.github.io/webcrypto/)). WebKit on Apple platforms does better: it encrypts each stored CryptoKey with a random key-encryption key, itself wrapped by a per-app "WebCrypto master key" kept in the Keychain ([WebKit source](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/crypto/cocoa/SerializedCryptoKeyWrapCocoa.mm)). Nothing comparable was checked for Chrome or Firefox, so the spec's caveat stands there.

| Approach | Where the key lives | Protects against | Doesn't protect against | Cost |
|---|---|---|---|---|
| None (OS disk encryption only) | — | A stolen device that is off or locked | Anyone using the OS account, malware, backups of the profile | None |
| Non-extractable key in IndexedDB | In the profile: Keychain-wrapped on Safari, maybe in clear elsewhere | Casual reading of OPFS files; crypto-shredding at logout; on Safari, a copy of the profile without the Keychain | XSS, malware while the user is logged in; on Chrome and Firefox, a copy of the whole profile. No user-verification gate | Low |
| **Key wrapped by a passkey PRF output** | Wrapped key on disk; the wrapping key exists only after user verification | The profile at rest while locked: a stolen or shared computer, backups, leftovers after logout | XSS, extensions or malware while unlocked | One passkey prompt per unlock; PRF support varies |
| Key fetched from the server after login | Memory | Data at rest | XSS while open | Breaks an offline cold start |

- **PRF support** (**high** for passkeys on iOS 18+ and macOS 15+, **medium** for security keys, **low-medium** for Windows Hello):
  - Safari 18 "adds support for the WebAuthn prf extension. It allows for retrieving a symmetric key from a passkey" ([Safari 18.0](https://webkit.org/blog/15865/webkit-features-in-safari-18-0/)). Safari 26.4 (2026-03-24) adds PRF for security keys "during both credential creation and authentication flows" ([Safari 26.4](https://webkit.org/blog/17862/webkit-features-for-safari-26-4/)).
  - In 26.4, security-key PRF returned the still-encrypted hmac-secret output. The reporter confirms a fix in Safari Technology Preview 241, but [bug 311099](https://bugs.webkit.org/show_bug.cgi?id=311099) is still open, so W3 tests security keys on shipping Safari.
  - BCD lists PRF in Chrome 116 and Firefox 139 for both calls, and Safari 18 for registration ([BCD](https://github.com/mdn/browser-compat-data/blob/main/api/CredentialsContainer.json)). Its "unsupported" Safari `get()` entry is stale: it points to [bug 259934](https://bugs.webkit.org/show_bug.cgi?id=259934), resolved fixed on 2025-11-21 (303406@main).
  - Apple's native PRF APIs start at iOS 18 and macOS 15, with no watchOS ([Apple](https://developer.apple.com/documentation/authenticationservices/asauthorizationpublickeycredentialprfassertioninputvalues)). Safari 26 also runs on macOS 14, where passkey PRF is unverified.
  - Windows Hello gained PRF only with recent Windows 11 builds (reported from 26200.7840, with Chrome 147), and support was reported unevenly (secondary, [Bitwarden issue](https://github.com/bitwarden/clients/issues/19858)).
- **Flow** (judgment):
  1. Generate a random key for each household database of the member's slot (§4.6).
  2. Wrap them with a key derived by HKDF from the member's PRF output, and store only the wrapped keys and the salt in IndexedDB.
  3. To unlock, run a local passkey ceremony. It works offline and gives the same Touch ID or Face ID gate as the apps.
  4. Keep the keys in worker memory, and drop them after an idle timeout.
  5. At logout, delete the wrapped keys first.
- **Without PRF** (judgment). PRODUCT.md requires offline parity, so a trusted browser without PRF still persists the database. It uses a non-extractable device key (crypto-shredding at logout) and a local passkey assertion as a UI gate. Tell the user that this lock stops a casual user, not malware or, on Chrome and Firefox, a copy of the profile.
- **Which code encrypts the database** (**low-medium**; no path is verified end to end):

  | Path | How | Evidence, November 2026 |
  |---|---|---|
  | **A. SQLite3MC's WASM build on `opfs-sahpool`** | Its [2.6.1 bundle](https://github.com/utelle/SQLite3MultipleCiphers/releases/tag/v2.6.1) (2026-10-12) carries `sqlite3.wasm` on SQLite 3.54.0 with the full JS API. Install the pool, wrap it with `sqlite3mc_vfs_create('<pool>', 0)`, open with `vfs: 'multipleciphers-<pool>'`, then `PRAGMA key`, as the maintainer advises for `opfs` ([#176](https://github.com/utelle/SQLite3MultipleCiphers/issues/176), [sqlite3mc_vfs.c](https://github.com/utelle/SQLite3MultipleCiphers/blob/main/src/sqlite3mc_vfs.c)). On the bare pool, a key fails: "Encryption is not supported by the VFS" ([codecext.c](https://github.com/utelle/SQLite3MultipleCiphers/blob/main/src/codecext.c)) | [#302](https://github.com/utelle/SQLite3MultipleCiphers/issues/302), opened 2026-11-05 and still open, used exactly this setup in Chromium and Firefox. Keys worked, but with `page_size` of 4,096 bytes or less, an encrypted database keeps spilled pages after ROLLBACK, and a worker killed mid-transaction reopens half-applied, while `integrity_check` says ok. The maintainer reproduced it in Chromium and reports it fixed in trunk for 2.7.0. The build's default 8,192-byte page ([sqlite3-wasm.c](https://github.com/sqlite/sqlite/blob/master/ext/wasm/api/sqlite3-wasm.c)) showed no failure. A related rollback fix ([#293](https://github.com/utelle/SQLite3MultipleCiphers/issues/293)) is also only in trunk. The maintainer says "JS WASM is **not** my programming domain" (#176) |
  | **B. Canonical build, in-memory database, encrypted snapshot** | The leader keeps the database in memory, exports it with `sqlite3_js_db_export()` after commits (debounced) and when a tab reports `pagehide`, encrypts it with AES-GCM through WebCrypto, and stores it in IndexedDB. Unlocking decrypts it and calls `sqlite3_deserialize()` ([cookbook](https://sqlite.org/wasm/doc/trunk/cookbook.md)). The outbox is stored apart, one encrypted record per mutation with `strict` durability, so a crash loses only the snapshot's lag, which pull and outbox replay restore (judgment) | Only canonical SQLite and WebCrypto. Costs: whole-file writes of a few MB per household (sync-engine.md) and the database held in memory. SQLite3MC couldn't encrypt an in-memory database anyway: it supports "only databases based on ordinary files" ([architecture](https://utelle.github.io/SQLite3MultipleCiphers/docs/architecture/arch_vfs/)) |
  | **C. PowerSync's `encryptionKey`** | `@powersync/web` on `OPFSCoopSyncVFS` ([README](https://github.com/powersync-ja/powersync-js/blob/main/packages/web/README.md)); its `mc-wa-sqlite` is SQLite3MC 2.3.3 on SQLite 3.53.0 ([Makefile](https://github.com/powersync-ja/wa-sqlite/blob/master/Makefile)) | #302's reporter measured the same failure in `@journeyapps/wa-sqlite` 2.0.10's `mc-wa-sqlite` with `AccessHandlePoolVFS`, at 4,096-byte pages. PowerSync's own VFS path is untested |

  - **Lean (judgment):** A at the default 8,192-byte page size, on 2.7.0 once it ships; B if spike W8 finds another codec bug. The same codec sits under C, so encryption doesn't favor adopting PowerSync. Until W8 passes, a persisted web database has only OS disk encryption behind the UI gate, and the app should say so.

### 4.6 Shared browsers: members, households, logout and wipe

- **At sign-in, ask "Keep data on this device?"** (judgment):
  - No: an in-memory database (SQLite `:memory:`), no `persist()`, the shared-device session rules above, and a warning before closing with unsynced edits.
  - Yes: the PRF lock where the authenticator supports it; otherwise the device-key fallback of §4.5, after saying what it doesn't protect.
- **Several members and households in one profile** (judgment on the cited mechanics; **medium**). A couple may share a home computer's browser profile, and the owner moves between a test and a production household (PRODUCT.md).
  - **Slots.** Each member who keeps data gets a slot: an `opfs-sahpool` pool of their own (`name` and `directory` derived from the member's opaque ID) holding one database per household. Pools coexist when their names and directories differ, and `removeVfs()` "removes its directory from OPFS (which means all client content is destroyed)" ([persistence](https://sqlite.org/wasm/doc/trunk/persistence.md)), so wiping one member leaves the others intact. Under §4.5's path B, a slot is a set of IndexedDB records instead.
  - **Keys.** A slot's keys are wrapped under its member's passkey only. The unlock ceremony lists that credential in `allowCredentials` and passes the salt through PRF's `evalByCredential`, "A record mapping base64url encoded credential IDs to PRF inputs", "Only applicable during assertions when allowCredentials is not empty" ([WebAuthn L3](https://www.w3.org/TR/webauthn-3/#prf-extension)). A partner in the same profile can't unlock the other's slot.
  - **One session per profile.** The `__Host-` cookie belongs to the origin, so a profile has one signed-in member at a time. "Switch member" locks the current slot (keys dropped, databases closed, files kept), ends the session and signs the other member in. The leader broadcasts the change, and every tab reloads into the lock screen.
  - **Households.** Switching from the test household to production keeps the session. The leader closes one database and opens the other. Each keeps its own outbox, cursor and `clientId`, so pending mutations never cross households, and the poke stream resubscribes. A test household may stay in memory only.
  - **Better still, one browser profile or OS account per person.** Safari 17 profiles keep "website data such as cookies, caches, service workers, and Web Push subscriptions" separate ([Safari 17](https://webkit.org/blog/14445/webkit-features-in-safari-17-0/)). Slots cover the couple who share one profile anyway.
- **Logout** (judgment, except where cited):
  1. Flush the outbox, or warn about unsynced edits.
  2. Broadcast `logout` to every tab over BroadcastChannel, and let the leader close the databases. This is the only tab-wide logout: no current engine honors `Clear-Site-Data: "executionContexts"`. Chrome never shipped it, Firefox removed it in 68 and Safari in 18.3 ([BCD](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Clear-Site-Data.json)).
  3. Delete the leaving member's slot: wrapped keys first, then `removeVfs()` on their pool. Other slots stay. When no slot remains, also delete Cache Storage and unregister the service worker.
  4. The server revokes the session and the device's push subscription, and expires its own cookie with `Max-Age=0`: "A zero or negative number will expire the cookie immediately" ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie)). It answers with `Clear-Site-Data: "cache"` (Chrome 61, Firefox 63, Safari 17, with partial `"cache"` support in Chrome; [BCD](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Clear-Site-Data.json)).
  5. "Remove all on-budget data from this browser" wipes every slot, and its response adds `"storage"`.
- **Why not `"cookies"` and `"storage"` on every logout** ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data)):
  - `"cookies"` "affects the entire registered domain, including subdomains". On a parent domain shared with other self-hosted apps, or under `<tailnet>.ts.net`, since `ts.net` is a public suffix, it would sign the member out of the sibling apps. This adopts native-apps-auth.md §9's amendment.
  - `"storage"` stays within the origin, but there it removes every member's slot, so only step 5 sends it. That narrows native-apps-auth.md's `"cache", "storage"` for shared profiles.
  - MDN's list for `"storage"` names "FileSystem API data" but not OPFS. The spec's `"storage"` step does require deleting "any other script-accessible storage mechanism", filesystems included ([spec](https://w3c.github.io/webappsec-clear-site-data/)), so OPFS is covered on paper. Keep the explicit wipe and check it in W5.
- **Remote wipe.** "Sign out other devices" revokes their sessions, and a revoked browser wipes itself at its next contact. A browser that never reconnects keeps its data, which is why the PRF lock matters (judgment).

## 5. Hosting and the shared API

### 5.1 Same origin or not

| Layout | Cookies | CORS | Safari specifics | Verdict |
|---|---|---|---|---|
| **Same origin**: `<host>`, the RP ID, serves the SPA plus `/api` | First-party `__Host-`, `SameSite=Strict` | None: "legitimate interactions … do not require any preflights" ([RFC 10017](https://www.rfc-editor.org/rfc/rfc10017.html)); CSP `connect-src 'self'` | Before 27.2, cookies set by a top-level navigation's response were capped at 7 days if the hostname is CNAMEd to another registrable domain ([27.2](https://webkit.org/blog/18515/webkit-features-for-safari-27-2/)); use A/AAAA records | **Pick** |
| Same site, two origins: `app.` and `api.` | On `api.` only, or `Domain=`, which rules out `__Host-` | Credentialed CORS with preflights | If `api.` is a CNAME to a PaaS hostname, ITP caps cookies set by those responses at 7 days ([ITP](https://webkit.org/tracking-prevention/)). Safari 27.2 fixed a similar cap on top-level responses of CNAMEd sites ([27.2](https://webkit.org/blog/18515/webkit-features-for-safari-27-2/)) | ✗ |
| Cross-site: SPA on a static host, API elsewhere | Third-party. ITP "blocks all third-party cookies" by default; access "can only be granted through the Storage Access API and the temporary compatibility fix for popups" ([ITP](https://webkit.org/tracking-prevention/)). Opt-in `Partitioned` (CHIPS) cookies work in Chrome 114, Firefox 141 and Safari 26.2 ([BCD](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Set-Cookie.json), [26.2](https://webkit.org/blog/17640/webkit-features-for-safari-26-2/)) | Required | An HttpOnly partitioned cookie avoids bearer tokens in JavaScript, but needs `SameSite=None`, so CSRF defense rests on CORS and custom headers (judgment) | ✗ |

### 5.2 Layout and headers

```text
# <host> = public_url = passkey RP ID, on the tailnet (§5.3); nothing below is public
https://<host>/                  SPA shell, hashed assets, sqlite3.wasm, /sw.js (no-cache)
https://<host>/api/v1/…          auth · bootstrap (NDJSON) · pull · push · account
https://<host>/api/v1/live       poke stream (leader tab only)
https://<host>/oauth/…           authorize and token for the native apps (native-apps-auth.md)
https://<host>/.well-known/oauth-authorization-server
https://<host>/join/…            invites; https://<host>/connect/enable-banking/callback (connectors-and-license.md)
# PowerSync, if adopted: a second hostname of its own, which it requires (§3.2)
# optional, per-build native passkey sheet only: /.well-known/apple-app-site-association
#   at the same <host>, served publicly through split-horizon DNS (native-apps-auth.md §9)
```

```text
Content-Security-Policy: default-src 'self';
  script-src 'sha256-<inline loader>' 'strict-dynamic' 'wasm-unsafe-eval';
  worker-src 'self'; connect-src 'self'; img-src 'self' data:;
  style-src 'self' 'sha256-47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU='   # empty runtime <style> (§3.4)
            'sha256-<expo-reset>';                                          # unless moved to a file
  object-src 'none'; base-uri 'none'; frame-ancestors 'none'; form-action 'self';
  require-trusted-types-for 'script'; trusted-types loader workers default
Cross-Origin-Opener-Policy: same-origin
# no COEP: cross-origin isolation switches Expo's workers to blob: URLs (§3.4)
Cross-Origin-Resource-Policy: same-origin
Strict-Transport-Security: max-age=63072000; includeSubDomains
Set-Cookie: __Host-session=…; Secure; HttpOnly; SameSite=Strict; Path=/
API data responses: Cache-Control: no-store; Vary: Cookie
```

- **Sources and judgment.** `Vary: Cookie` follows Electric's advice for cookie-authenticated data behind caches ([Electric](https://electric.ax/docs/guides/auth)). The script policy is web.dev's hash-based strict form (§4.4), and `worker-src 'self'` still governs workers. An allowlist such as `script-src 'self'` is only a fallback if W4 finds that the strict policy breaks the module graph, workers or WASM. The policy names and the rest are judgment, to be checked in spike W4.
- **Serving.** Caddy, or the API binary itself, serves the static files from the same host. A household's few members don't need a CDN, and a CDN couldn't reach a Tailscale-only origin anyway (judgment).

### 5.3 Tailscale-only deployment

PRODUCT.md and ADR 0003 settle the hosting: the maintainer's instance runs on a dedicated EU VPS with a dedicated domain, reachable only over Tailscale, and every feature must work that way. There is no public sign-up and no service run for other people; anyone else hosts their own instance (ADR 0001). This overrides the public VPS in [apple-client-stack.md](apple-client-stack.md) §4.4, which predates ADR 0003.

1. **The origin lives on the tailnet.** `<host>` is either of self-hosting.md's two private options:
   - a `*.ts.net` name with Tailscale's own certificate through Tailscale Serve, tied to the tailnet and sharing a site with every machine in it (§4.2, §4.6);
   - or a name under the owner's domain, with A/AAAA records to the VPS's Tailscale addresses in public DNS or Tailscale split DNS, and an ACME DNS-01 certificate, since no CA can reach the host. A CNAME to the `*.ts.net` name would point at another registrable domain, which hits the pre-27.2 Safari cookie cap (§5.1) (judgment).

   Off the tailnet, the web app behaves as offline: the cached shell and local data still work, and sync resumes when Tailscale reconnects.
2. **Nothing has to be public.** Native apps sign in through the instance's page in `ASWebAuthenticationSession` and bank consent returns through the instance's `https` callback plus a custom scheme, so neither needs an association file (native-apps-auth.md §2, §6; ADR 0003: "bank-consent returns go through the member's browser"). Whether Enable Banking accepts a tailnet callback URL is still open (self-hosting.md P6). Only the optional per-build native passkey sheet needs the association file, public at the exact host through split-horizon DNS (§4.2). The instance takes no inbound connections from the internet.
3. **Every browser needs Tailscale.** A member's own computer can run it, but a work PC that can't install it can't reach the instance (judgment).
4. **Self-hosters.** Each instance has its own host and RP ID. One maintainer-signed app build signs in to any of them through the browser sheet (native-apps-auth.md TL;DR 1), and the web app serves every self-hoster from the first release; PRODUCT.md leaves how self-hosters get the native apps undecided.
5. **A self-hoster who exposes the instance publicly** keeps passkey-only login, invite-only enrollment and rate limits on the auth endpoints (judgment).

### 5.4 What the web means for an API shared by every client

1. **Two authenticators, one principal.**
   - Cookie for the web; a DPoP-bound access token from native-apps-auth.md's authorization server for the apps.
   - CSRF checks apply only to cookie requests: an exact `Origin` allow-list and a required static custom header, which forces a preflight from any other origin ([RFC 10017 §6.1.3.3.2](https://www.rfc-editor.org/rfc/rfc10017.html), by analogy). Both are mandatory, since `SameSite` lets same-site siblings through (§4.3).
2. **Devices include browser profiles,** one per member slot. They have no hardware key. Store a label and a last-seen time, make them revocable, and wipe that slot on its next contact (judgment).
3. **Contract tests cover the last two client versions** of every client kind (§3.3, point 4).
4. **Web Push beside APNs.**
   - Store `PushSubscription`s and sign with VAPID. For Apple's web push, "You don't need to join the Apple Developer Program" ([Apple](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers)).
   - Delivery is outbound only, so it works from a Tailscale-only instance.
   - On iOS it only reaches installed web apps (§2.4).
5. **Same endpoints and semantics** for bootstrap, pull and push; the web gets no special endpoints (judgment).
6. **Account deletion, export and step-up authentication** work on the web exactly as in the apps (judgment).

## Recommendation for on-budget

| Layer | Choice | Confidence | Trade-off accepted |
|---|---|---|---|
| Web storage | SQLite WASM 3.54 `opfs-sahpool` in a leader worker, with `preserveOnInitFailure` and a pause/unpause handover; one pool per member slot, one database per household; with PowerSync, its wa-sqlite `OPFSCoopSyncVFS` | Medium-high | Data may vanish (a Safari tab after seven days of Safari use without interaction, storage pressure), so clients re-bootstrap |
| Offline policy | Persistent database on trusted devices: PRF lock where supported, otherwise a device key and a passkey gate in the UI; in-memory on shared devices | Medium | Shared PCs lose offline data on reload; one passkey prompt per unlock; weaker protection where PRF is missing |
| Encryption code | SQLite3MC ≥ 2.7.0 on `multipleciphers-opfs-sahpool` at 8,192-byte pages, or an in-memory database with an AES-GCM snapshot in IndexedDB; W8 decides | Low-medium | Until W8 passes, OS disk encryption only; a young WASM codec, or whole-file writes |
| Client stack | Expo (cross-platform-ui-stack.md); `expo-sqlite` with SQLCipher on iOS, op-sqlite only with PowerSync; one `sync-core` package on two hosts | Medium | Expo's web workers are alpha; expo-sqlite's web backend unused |
| Sync engine | The custom protocol, one TypeScript client (default); PowerSync only if Spike A + W2 show less code with its ops and authorization costs counted | Medium | The browser plumbing built in-house, or a Node service, a second hostname and Sync Streams as a second authorization layer |
| Contract | One OpenAPI/JSON contract, versioned named mutations, conformance vectors | High | Golden vectors still pin the glance the Swift watch decodes |
| Backend | TypeScript on Node 26 LTS, with `oidc-provider` and SimpleWebAuthn; gate: cross-platform-ui-stack.md S2 | Medium | apple-client-stack.md's Swift pick dropped; a JavaScript runtime on the server |
| Web auth | Passkeys with the exact instance host as RP ID, which also serves the web app, plus a `__Host-` HttpOnly `SameSite=Strict` cookie with mandatory `Origin` and custom-header checks | High | Two auth paths and CSRF tests in the API; a permanent hostname |
| XSS hardening | Hash-based strict CSP with `'strict-dynamic'`, `'wasm-unsafe-eval'`, the empty-string `style-src` hash for runtime styles, Trusted Types with a `default` policy for chunks and workers, SRI, no third-party JavaScript | Medium-high | No analytics or widgets on the app origin; Safari 26 floor (Trusted Types); no COEP |
| Logout and wipe | Server revoke, cookie expiry, `Clear-Site-Data: "cache"` (`"storage"` only to remove every slot), BroadcastChannel logout, the member's slot wiped, crypto-shredding | Medium-high | Browsers that never reconnect can't be wiped |
| Hosting | One origin `https://<host>` on the Tailscale-only instance: a `*.ts.net` name, or a name under the owner's domain with A/AAAA records and a DNS-01 certificate; nothing public | High (same origin) / medium (DNS, certificates) | Every browser needs Tailscale; work PCs that can't run it are out |
| Push | Web Push (VAPID) beside APNs | Medium | A second notifier; iOS web push only when installed |
| Browser floor | Safari and iOS 26+ (Trusted Types), current Chrome, Edge and Firefox | Medium | Older Safari versions excluded |

**Trade-offs the owner must accept.**

- The browser database is a cache that can disappear. Unsynced edits in a tab the browser evicts are lost. Keep that window to seconds.
- The web is where XSS or a malicious extension would expose everything a user can see. The defense is engineering discipline (CSP, Trusted Types, no third-party JavaScript), not cryptography.
- Unlocking stored data on the web needs a passkey prompt; data on a shared PC doesn't survive a reload. Browsers without PRF get a weaker lock, and until W8 passes, no browser gets encryption beyond the OS's.
- Partners who share a browser profile take turns: one signed-in member at a time, each with a locked slot.
- Sync work grows. You build the browser plumbing (storage, eviction recovery, multi-tab) into your own client, or run PowerSync with the costs of §3.2.
- The hostname is permanent, because it is the RP ID.
- Every browser that opens the web app needs Tailscale; computers that can't run it, such as many work PCs, can't reach the instance.

**Constraints on the client stack** (cross-platform-ui-stack.md's B):

1. Its web target must work offline under §2's storage rules, in a leader worker that Expo's alpha worker support bundles.
2. Local storage is SQLite on every client, and schema and views are shared as SQL, which Swift widgets can read too.
3. The sync engine is one TypeScript package with ports for SQLite, transport, auth and clock (§3.4). TypeScript-only engines are out on capability and status, and CloudKit is out. A Swift client is needed only for Swift extensions that write locally; the watch reads a server-built glance.
4. With a TypeScript server, mutators exist once (§3.5).
5. The design system is token-first, generated for the shared UI and for Swift. Components are written once, plus a SwiftUI subset for the watch and widgets.
6. Hermes's iOS `Intl` is partial: load FormatJS polyfills, with golden tests across platforms.
7. On the web, use the official SQLite WASM in the leader worker, not `expo-sqlite`'s or op-sqlite's web backends, which need COOP/COEP and don't coordinate tabs; then send no COEP at all.

## Open questions and spikes

**Spikes** (each 1 day or less unless noted; pass means an agent built it and it is tested from the CLI or Playwright):

- **W1 Browser storage.**
  - Setup: `opfs-sahpool` in a leader worker with Web Locks and BroadcastChannel, `preserveOnInitFailure` and a pause/unpause handover, bundled by Expo's Metro in a `single` export.
  - Targets: Safari tab, Dock app, iOS Home Screen app, Chrome, Firefox.
  - Measure: cold start, wasm transfer size, write latency, the result of `persist()`, private windows, killing a tab mid-transaction, a leader handover while another tab starts, and "Clear website data". Also reopen the app, still signed in, after its storage was evicted.
  - Slots: two member pools in one profile, a member switch, a test-to-production household switch, and one slot's wipe leaving the other intact.
- **W2 PowerSync on React Native plus web (2 days, extends Spike A).**
  - Same household on Safari and on an iPhone, with offline edits on both, through `@powersync/react-native` on op-sqlite and `@powersync/web`.
  - `uploadData()` carries named mutations. A retired mutation version gets a 2xx rejection, and the stale client keeps receiving checkpoints.
  - PowerSync on its own hostname, as its docs require, with a JWT of 5 minutes or less from a cookie-authenticated endpoint. Record whether an unsupported same-origin proxy (prefix stripped, token swapped, HTTP connection method) would work.
  - Count the costs of §3.2: the service built by Nix or run under `oci-containers`, Sync Streams with membership subqueries, and a CI test that a member of household A never receives household B's rows.
  - Two Safari tabs on `OPFSCoopSyncVFS`; encryption runs in W8.
  - Pass: less code than the custom client's browser plumbing, all costs counted, with no data loss.
- **W3 Passkeys and PRF.**
  - Test PRF on `create` and `get` for iCloud Keychain, Google Password Manager, 1Password, a security key (bug 311099, on shipping Safari), Windows Hello on a current Windows 11 build, and the hybrid QR flow.
  - Browsers: Safari on macOS 14, 15 and 26 and on iOS, Chrome on macOS, Windows and Android, Firefox.
  - Also test an offline unlock, the exact-host RP ID from both the web app and an `ASWebAuthenticationSession` sheet, a ceremony attempted from a sibling host, a related origin listed in `/.well-known/webauthn`, and two members' credentials in one profile with `evalByCredential`.
- **W4 CSP.** The policy of §5.2 in report-only, then enforced, on Safari 26 and 27, Chrome and Firefox, against the exported Expo build: the empty-string hash for react-native-web's and Unistyles' runtime `<style>` elements, the `expo-reset` hash, Expo's chunk scripts under `'strict-dynamic'`, the `default` Trusted Types policy for chunks, the leader worker and the service worker, WASM, and the absence of COEP.
- **W5 Logout and wipe.** `Clear-Site-Data: "cache"` plus explicit deletion in each engine. Check that the leaving member's OPFS pool is gone, that other slots and sibling apps' cookies survive, that `"storage"` on "remove all" clears OPFS, and that every tab logs out through BroadcastChannel.
- **W6 Web Push.** VAPID from the server to a macOS Safari tab, a Dock app and an iOS Home Screen app; also declarative payloads.
- **W7 One sync core, two hosts.** Run `packages/sync-core` against SQLite WASM in the leader worker and against `expo-sqlite` on the iPhone's JS thread, with the NDJSON bootstrap and the poke stream read through `expo/fetch` and browser fetch. Record where `expo-sqlite`'s async calls execute. Pass: the sync-engine.md simulator's seeds pass through both drivers.
- **W8 Encryption at rest (1 day; gates the PRF lock).** In the leader worker, keyed from a PRF-derived key: path A on SQLite3MC 2.6.1, then 2.7.0 when released, at 8,192- and 4,096-byte pages; path B; and with W2, PowerSync's `encryptionKey`. Run #302's tests: ROLLBACK after a cache spill, and `Worker.terminate()` mid-transaction, then reopen. Pass: no row left changed, no file readable without the key, and unlock and snapshot times measured for a 20 MB database.

**Questions for the owner.**

1. Who opens the web app, and where: members' own computers, or shared ones? Do partners share one browser profile? Each needs Tailscale. The answers set the default for "Keep data on this device?" and whether slots ship in v1.
2. On a trusted browser without PRF, offline parity means a persistent database behind a UI-only passkey gate. Is that acceptable, or should such browsers stay online-only despite PRODUCT.md's parity? And until W8 passes, may trusted browsers persist unencrypted data?
3. Building is the default. Would you accept PowerSync's costs (a Node service packaged for NixOS, a second hostname with a JWT in the browser, Sync Streams as a second authorization layer) if Spike A shows it saves code?
4. Which hostname becomes the instance's permanent RP ID: a name under your own domain, or a `*.ts.net` name (native-apps-auth.md question 6)?
5. Are web budget alerts needed, or are alerts native-only?
6. Do you accept a TypeScript backend (§3.5) in place of apple-client-stack.md's Swift pick?

**Not verified.**

- Whether macOS Dock apps are exempt from ITP's 7-day cap like Home Screen apps.
- What `persist()` returns in a Safari tab and in a Dock app.
- Whether each engine's `Clear-Site-Data: "storage"` deletes OPFS in practice, as the spec requires.
- Security-key PRF on shipping Safari (bug 311099 is fixed in Technology Preview 241 but open); passkey PRF in Safari on macOS 14; Windows Hello PRF (secondary sources only).
- SQLite3MC's WASM codec in practice: whether 2.7.0 fixes #302 and #293 there, and how PowerSync's `OPFSCoopSyncVFS` path behaves (W8). I didn't run any browser test.
- Whether PowerSync works behind a same-origin, token-swapping proxy, a layout its docs don't support (W2).
- That today's engines all admit react-native-web's and Unistyles' empty `<style>` elements by the empty-string hash, and that an exported Expo `single` build bundles the leader worker (W1, W4).
- Whether cookies carry over when an iOS site is added to the Home Screen.
- Whether public DNS records that point at Tailscale addresses resolve on every network, or need split DNS.
- Whether Enable Banking accepts a callback URL on a tailnet-only host (self-hosting.md P6).

## Sources

Accessed 2026-11-07.

- **Product truth and sibling notes:** [PRODUCT.md](../../PRODUCT.md) · [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md) · [ADR 0002](../adr/0002-one-ui-codebase-native-watch.md) · [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md) · [cross-platform-ui-stack.md](cross-platform-ui-stack.md) · [native-apps-auth.md](native-apps-auth.md) · [self-hosting.md](self-hosting.md) · [nixos-vps-infra.md](nixos-vps-infra.md) · [connectors-and-license.md](connectors-and-license.md) · [design-system.md](design-system.md) · [watchos-app.md](watchos-app.md) · [i18n-l10n.md](i18n-l10n.md)
- **Encryption at rest:** SQLite3MC [2.6.1 release](https://github.com/utelle/SQLite3MultipleCiphers/releases/tag/v2.6.1) (WASM bundle read, not run), issues [#176](https://github.com/utelle/SQLite3MultipleCiphers/issues/176), [#293](https://github.com/utelle/SQLite3MultipleCiphers/issues/293) and [#302](https://github.com/utelle/SQLite3MultipleCiphers/issues/302), source [sqlite3mc_vfs.c](https://github.com/utelle/SQLite3MultipleCiphers/blob/main/src/sqlite3mc_vfs.c) and [codecext.c](https://github.com/utelle/SQLite3MultipleCiphers/blob/main/src/codecext.c), [VFS architecture](https://utelle.github.io/SQLite3MultipleCiphers/docs/architecture/arch_vfs/) · [PowerSync wa-sqlite Makefile](https://github.com/powersync-ja/wa-sqlite/blob/master/Makefile) · [PowerSync web README](https://github.com/powersync-ja/powersync-js/blob/main/packages/web/README.md) · [sqlite3-wasm.c](https://github.com/sqlite/sqlite/blob/master/ext/wasm/api/sqlite3-wasm.c) · [SQLite WASM cookbook](https://sqlite.org/wasm/doc/trunk/cookbook.md)
- **Client stack (Expo, React Native):** [expo-sqlite](https://docs.expo.dev/versions/latest/sdk/sqlite/) and its web [worker.ts](https://github.com/expo/expo/blob/main/packages/expo-sqlite/web/worker.ts) and [WorkerChannel.ts](https://github.com/expo/expo/blob/main/packages/expo-sqlite/web/WorkerChannel.ts) · op-sqlite [releases](https://github.com/OP-Engineering/op-sqlite/releases), [installation](https://github.com/OP-Engineering/op-sqlite/blob/main/docs/docs/installation.md), [API](https://github.com/OP-Engineering/op-sqlite/blob/main/docs/docs/api.md) · [PowerSync React Native](https://docs.powersync.com/client-sdk-references/react-native-and-expo) · [Expo Metro config (web workers)](https://docs.expo.dev/versions/latest/config/metro/) · [expo/fetch](https://docs.expo.dev/versions/latest/sdk/expo/) · Expo source: [asyncRequireModule.ts](https://github.com/expo/expo/blob/main/packages/expo/src/async-require/asyncRequireModule.ts), [fetchThenEval.web.ts](https://github.com/expo/expo/blob/main/packages/expo/src/async-require/fetchThenEval.web.ts), [index.html template](https://github.com/expo/expo/blob/main/packages/@expo/cli/static/template/index.html) · [RN threading model](https://reactnative.dev/architecture/threading-model) · [RN WebSocket.js](https://github.com/facebook/react-native/blob/main/packages/react-native/Libraries/WebSocket/WebSocket.js) · [react-native-web createCSSStyleSheet.js](https://github.com/necolas/react-native-web/blob/master/packages/react-native-web/src/exports/StyleSheet/dom/createCSSStyleSheet.js) · [Unistyles sheet.ts](https://github.com/jpudysz/react-native-unistyles/blob/main/packages/unistyles/src/web/css/sheet.ts) · [WebKit bug 233071](https://bugs.webkit.org/show_bug.cgi?id=233071) · [Firefox bug 1706787](https://bugzil.la/1706787) · [MDN style-src](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/style-src)
- **Backend:** [Node.js release schedule](https://github.com/nodejs/Release/blob/main/schedule.json) · [oidc-provider](https://github.com/panva/node-oidc-provider) ([sender_constraints.js](https://github.com/panva/node-oidc-provider/blob/main/lib/actions/authorization/sender_constraints.js)) · npm: [pg-boss](https://www.npmjs.com/package/pg-boss), [graphile-worker](https://www.npmjs.com/package/graphile-worker), [web-push](https://www.npmjs.com/package/web-push), [@parse/node-apn](https://www.npmjs.com/package/@parse/node-apn), [hono](https://www.npmjs.com/package/hono), [drizzle-orm](https://www.npmjs.com/package/drizzle-orm)
- **SQLite and browser storage:** [SQLite WASM persistence](https://sqlite.org/wasm/doc/trunk/persistence.md) · [opfs-sahpool source](https://github.com/sqlite/sqlite/blob/master/ext/wasm/api/sqlite3-vfs-opfs-sahpool.c-pp.js) · [SQLite changes](https://sqlite.org/changes.html) · [@sqlite.org/sqlite-wasm](https://www.npmjs.com/package/@sqlite.org/sqlite-wasm) · [wa-sqlite](https://github.com/rhashimoto/wa-sqlite) · [wa-sqlite VFS table](https://github.com/rhashimoto/wa-sqlite/blob/master/src/examples/README.md) · [@journeyapps/wa-sqlite](https://www.npmjs.com/package/@journeyapps/wa-sqlite) · [PGlite filesystems](https://pglite.dev/docs/filesystems) · [PGlite multi-tab](https://pglite.dev/docs/multi-tab-worker) · [SQLite3 Multiple Ciphers](https://github.com/utelle/SQLite3MultipleCiphers) · [jsDelivr package listing](https://www.jsdelivr.com/package/npm/@sqlite.org/sqlite-wasm) · [Notion: WASM SQLite](https://www.notion.com/blog/how-we-sped-up-notion-in-the-browser-with-wasm-sqlite) (first-party)
- **Quotas, eviction, durability:** [MDN quotas and eviction](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria) · [WebKit storage policy](https://webkit.org/blog/14403/updates-to-storage-policy/) · [WebKit tracking prevention](https://webkit.org/tracking-prevention/) · [WebKit 7-day cap (2020)](https://webkit.org/blog/10218/full-third-party-cookie-blocking-and-more/) · WebKit ITP source: [WebResourceLoadStatisticsStore.cpp](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/Classifier/WebResourceLoadStatisticsStore.cpp), [ResourceLoadStatisticsStore.cpp](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/Classifier/ResourceLoadStatisticsStore.cpp), [NetworkStorageManager.cpp](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/storage/NetworkStorageManager.cpp) · [web.dev persistent storage](https://web.dev/articles/persistent-storage) · [Chrome IndexedDB durability](https://developer.chrome.com/blog/indexeddb-durability-mode-now-defaults-to-relaxed) · [MDN IDB transaction](https://developer.mozilla.org/en-US/docs/Web/API/IDBDatabase/transaction)
- **WebKit releases and web apps:** [Safari 17.0](https://webkit.org/blog/14445/webkit-features-in-safari-17-0/) · [Safari 18.0](https://webkit.org/blog/15865/webkit-features-in-safari-18-0/) · [Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/) · [Safari 26.2](https://webkit.org/blog/17640/webkit-features-for-safari-26-2/) · [Safari 26.4](https://webkit.org/blog/17862/webkit-features-for-safari-26-4/) · [Safari 27.0](https://webkit.org/blog/18325/webkit-features-for-safari-27-0/) · [Safari 27.2](https://webkit.org/blog/18515/webkit-features-for-safari-27-2/) · [Web Push on iOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/) · [Declarative Web Push](https://webkit.org/blog/16535/meet-declarative-web-push/) · [Apple web push](https://developer.apple.com/documentation/usernotifications/sending-web-push-notifications-in-web-apps-and-browsers) · [MacRumors on EU web apps](https://www.macrumors.com/2024/03/01/apple-walks-back-decision-to-disable-eu-web-apps/) (secondary)
- **Browser compatibility (BCD):** [SharedWorker](https://github.com/mdn/browser-compat-data/blob/main/api/SharedWorker.json) · [FileSystemFileHandle](https://github.com/mdn/browser-compat-data/blob/main/api/FileSystemFileHandle.json) · [StorageManager](https://github.com/mdn/browser-compat-data/blob/main/api/StorageManager.json) · [LockManager](https://github.com/mdn/browser-compat-data/blob/main/api/LockManager.json) · [SyncManager](https://github.com/mdn/browser-compat-data/blob/main/api/SyncManager.json) · [Clear-Site-Data](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Clear-Site-Data.json) · [CSP](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Content-Security-Policy.json) · [Trusted Types](https://github.com/mdn/browser-compat-data/blob/main/api/TrustedTypePolicyFactory.json) · [COEP](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Cross-Origin-Embedder-Policy.json) · [Set-Cookie](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Set-Cookie.json) · [CredentialsContainer](https://github.com/mdn/browser-compat-data/blob/main/api/CredentialsContainer.json) · [PublicKeyCredential](https://github.com/mdn/browser-compat-data/blob/main/api/PublicKeyCredential.json) · [Secure-Session-Registration (DBSC)](https://github.com/mdn/browser-compat-data/blob/main/http/headers/Secure-Session-Registration.json) · [Temporal](https://github.com/mdn/browser-compat-data/blob/main/javascript/builtins/Temporal/ZonedDateTime.json)
- **PowerSync:** [supported platforms](https://docs.powersync.com/resources/supported-platforms) · [web SDK](https://docs.powersync.com/client-sdk-references/javascript-web) · [auth](https://docs.powersync.com/configuration/auth/overview) · [consistency](https://docs.powersync.com/architecture/consistency) · [write validation errors](https://docs.powersync.com/handling-writes/handling-write-validation-errors) · [client architecture](https://docs.powersync.com/architecture/client-architecture) · [encryption](https://docs.powersync.com/client-sdks/advanced/data-encryption) · [raw tables](https://docs.powersync.com/client-sdks/advanced/raw-tables) · [Sync Streams](https://docs.powersync.com/sync/streams/overview) · [Sync Streams GA](https://releases.powersync.com/announcements/ann_7UgfQuNwRmpQk) · [open source](https://www.powersync.com/open-source) · [pricing](https://www.powersync.com/pricing) · [license PR #878](https://github.com/powersync-ja/powersync-service/pull/878) · [service v1.28.0](https://github.com/powersync-ja/powersync-service/releases/tag/v1.28.0) · LICENSE history: [JS](https://github.com/powersync-ja/powersync-js/commits/main/LICENSE), [Swift](https://github.com/powersync-ja/powersync-swift/commits/main/LICENSE) · releases: [service](https://github.com/powersync-ja/powersync-service/releases), [Swift](https://github.com/powersync-ja/powersync-swift/releases), [JS](https://github.com/powersync-ja/powersync-js/releases) · [deployment architecture](https://docs.powersync.com/maintenance-ops/self-hosting/deployment-architecture) · [source database setup](https://docs.powersync.com/configuration/source-db/setup) · [Sync Streams queries](https://docs.powersync.com/sync/streams/queries) · [service Dockerfile](https://github.com/powersync-ja/powersync-service/blob/main/service/Dockerfile) · SDK source: [AbstractRemote.ts](https://github.com/powersync-ja/powersync-js/blob/main/packages/shared-internals/src/client/sync/stream/AbstractRemote.ts), [WebSocketSupport.ts](https://github.com/powersync-ja/powersync-js/blob/main/packages/shared-internals/src/client/sync/stream/WebSocketSupport.ts) · [PostgreSQL CREATE ROLE](https://www.postgresql.org/docs/current/sql-createrole.html)
- **Other engines:** [Electric](https://github.com/electric-sql/electric) · [Electric auth](https://electric.ax/docs/guides/auth) · [Electric writes](https://electric.ax/docs/guides/writes) · [Zero offline](https://zero.rocicorp.dev/docs/offline) · [LiveStore](https://livestore.dev/) · [Jazz](https://jazz.tools/) · [InstantDB docs](https://www.instantdb.com/docs) · [Triplit joins Supabase](https://supabase.com/blog/triplit-joins-supabase) · [Triplit repo](https://github.com/aspen-cloud/triplit) · [Automerge](https://github.com/automerge/automerge) · [automerge-swift](https://github.com/automerge/automerge-swift) · [Loro](https://github.com/loro-dev/loro) · [loro-swift](https://github.com/loro-dev/loro-swift) · [yswift](https://github.com/y-crdt/yswift) · [sqlite-sync](https://github.com/sqliteai/sqlite-sync) · [Expo SQLite](https://docs.expo.dev/versions/latest/sdk/sqlite/) · [Hermes features](https://github.com/facebook/hermes/blob/static_h/doc/Features.md) · [TN3135](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos) · [DTCG 2025.10](https://www.w3.org/community/design-tokens/2025/10/28/design-tokens-specification-reaches-first-stable-version/)
- **Security standards and guidance:** [RFC 10017 (OAuth for browser-based apps)](https://www.rfc-editor.org/rfc/rfc10017.html) · [RFC 9449 (DPoP)](https://www.rfc-editor.org/rfc/rfc9449) · [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455#section-10.2) · [WebAuthn Level 3](https://www.w3.org/TR/webauthn-3/) ([§5.11 related origins](https://www.w3.org/TR/webauthn-3/#sctn-related-origins)) · [Chrome: passkey updates in 128 and 129](https://developer.chrome.com/blog/passkeys-updates-chrome-129) · Firefox: [StaticPrefList.yaml](https://github.com/mozilla-firefox/firefox/blob/main/modules/libpref/init/StaticPrefList.yaml), [bug 2010193](https://bugzilla.mozilla.org/show_bug.cgi?id=2010193), [bug 2043449](https://bugzilla.mozilla.org/show_bug.cgi?id=2043449) · [passkeys.dev device support](https://passkeys.dev/device-support/) (secondary) · [WebCrypto](https://w3c.github.io/webcrypto/) · [WebKit SerializedCryptoKeyWrapCocoa.mm](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/crypto/cocoa/SerializedCryptoKeyWrapCocoa.mm) · [web.dev strict CSP](https://web.dev/articles/strict-csp) · [MDN Trusted Types](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API) · [Clear-Site-Data spec](https://w3c.github.io/webappsec-clear-site-data/) · [MDN Clear-Site-Data](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data) · [draft-ietf-httpbis-layered-cookies-02](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-layered-cookies-02) · [MDN EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) · [HTML EventSource](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface) · [WebSockets standard](https://websockets.spec.whatwg.org/#the-websocket-interface) · [MDN MAX_SAFE_INTEGER](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER) · [Chrome content scripts](https://developer.chrome.com/docs/extensions/develop/concepts/content-scripts) · [Google DBSC](https://blog.google/security/protecting-cookies-with-device-bound-session-credentials/) · [WebKit bug 259934](https://bugs.webkit.org/show_bug.cgi?id=259934) · [WebKit bug 311099](https://bugs.webkit.org/show_bug.cgi?id=311099) · [Bitwarden issue 19858](https://github.com/bitwarden/clients/issues/19858) (secondary) · [Apple PRF assertion input](https://developer.apple.com/documentation/authenticationservices/asauthorizationpublickeycredentialprfassertioninputvalues) · [SimpleWebAuthn](https://github.com/MasterKale/SimpleWebAuthn) · [MDN Set-Cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie) · [MDN Site](https://developer.mozilla.org/en-US/docs/Glossary/Site) · [WebAuthn L3 PRF extension](https://www.w3.org/TR/webauthn-3/#prf-extension)
