# Bring-your-own bank connectors and the open-source license

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: §6 license; `bigint` amounts; server-only parsing; the owner's view of every connection; per-person provider applications.

Research note, 2026-11-07, revised the same day after a fact-check. Versions, prices and terms are as of November 2026 unless dated otherwise. "Secondary" marks issue threads, aggregators and third-party write-ups; "judgment" marks my own assessment. Confidence: **H** high, **M** medium, **L** low. It builds on [bank-aggregation.md](bank-aggregation.md), [apple-client-stack.md](apple-client-stack.md), [sync-engine.md](sync-engine.md), [web-client-sync.md](web-client-sync.md) and [factory-ci-quality.md](factory-ci-quality.md), and edits none of them. It reads terms and licenses; it is not legal advice.

## Question

on-budget is self-hosted: each instance has one owner, holds a few isolated households, and has members the owner adds; there is no public sign-up. Two questions:

1. How should each person bring their own bank-data credentials (setup, key custody, consent callbacks on a self-hosted domain, renewal, limits)? How should connectors be built so that provider code stays isolated and testable?
2. Which open-source license fits a self-hosted personal-finance app maintained by one person and shipped through Apple's stores? How should the name be protected?

## TL;DR

1. **Bring your own Enable Banking app, one per member, never one per instance.** Each member who wants bank sync registers their own free restricted production app. Their instance stores that app's key, scoped to that member and encrypted, and uses it only for the accounts that member linked. EB's terms (updated 2026-01-09) allow free production use "solely for evaluation purposes or for the personal use of private individuals". They grant no right to "account information that does not belong to the Control Panel user who associated the Linked Accounts", nor to "account information belonging to businesses" ([EB ToS](https://enablebanking.com/terms)). That limit follows whoever linked each account, not whoever registered the app. EB lets an app be shared with another Control Panel user, but doesn't say whether that user may link their own accounts to it (spike 1). Per-member apps avoid the question and keep each member's key and request log apart. **H** for each member's own accounts. **M–L** that EB accepts a member's key held on a server another member administers: ask EB in writing (spike 1).
2. **Generate the key on the instance.** The member pastes only the public certificate into EB's Control Panel, which offers "Generate outside the browser and import public certificate". The private key never leaves the server, and `GET /application` lets on-budget check the app (active, redirect URLs). **H** that EB supports this flow.
3. **Callbacks: one HTTPS callback per instance, with the code exchanged on the server, then a custom-scheme bounce back to the app.** EB's own docs set no rule for redirect URLs. Firefly III's and Actual's docs say production needs `https`, and Firefly III adds that "the URL does not have to be public". An App Store build catches `https` callbacks only on domains in its associated-domains entitlement, so self-hosters' domains need `onbudget://`-style completion plus polling. **H** on the Apple side; **M** that EB accepts a callback on a tailnet-only host, until spike 3 confirms it.
4. **Budget the fetches.** At most 4 unattended accesses per account per day ([RTS art. 36(5)(b)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389)). Start with 2 background slots per consent, and add more at a bank only after measuring how it counts calls, as bank-aggregation.md does. Send PSU headers only on an explicit refresh such as pull-to-refresh: opening the app is not an "active request" ([EBA Q&A 2018_4172](https://www.eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4172)). An instance reachable only over Tailscale sees tailnet addresses, so whether banks accept one as `Psu-Ip-Address` is open (spike 4). Renew each consent within the bank's `maximum_consent_validity`: 180 days at most banks, 90 at some. **H** on the rules; **M** on the slot count and the PSU IP.
5. **Use one connector SDK, with connectors as in-repo modules in a static registry.** v1 ships Enable Banking, OFX, QIF and CSV; contracted aggregators or token providers can come later. The core owns scheduling, dedupe, storage, sharing and notifications. Connectors are stateless translators that raise typed errors and must pass a shared contract test kit. **M–H**.
6. **License: AGPL-3.0-or-later for the whole repo, with an AGPLv3 §7 App Store additional permission named in every file's SPDX header, DCO sign-off from the first commit, and a TRADEMARKS.md.**
   - It matches the self-hosted finance peers (Firefly III, Ghostfolio, Kresus, Maybe/Sure, Immich).
   - It keeps modified hosted versions open (§13).
   - The permission removes the App Store conflict the FSF describes, without a CLA. Nextcloud iOS's non-enforcement pledge is the established model without a CLA. Signal's §7 wording comes from an archived library last updated in 2020, whose successor uses a CLA instead; Bulwark Mobile adopted a provisional AGPL app-store permission in September 2026.
   - **M**: this is a values call. Pick **Apache-2.0** instead if no licensing ceremony and maximum reuse matter more than reciprocity.
7. **No license protects the name.** "on-budget" is descriptive for budgeting software (EUTMR art. 7(1)(b)–(c)), and similar names are in use. Choose a distinctive name before the first store listing, then register it: INPI €190 + €40 per extra class; EU trade mark €850 + €50 + €150 per class from the third. **M**.

## 1. What the self-hosted model changes

- bank-aggregation.md was first written for one hosted service for ~20 friends. That made the partner and the friends third parties under EB's terms and pointed to a paid AISP contract. Now each household member can be their own EB "Control Panel user" with their own app. The free mode then covers the household's personal accounts, as long as each account is linked by the person it belongs to (H, §3).
- The PSD2 position is unchanged: EB is the licensed AISP, and each member is both PSU and app owner (bank-aggregation.md §7.1). The maintainer runs no service, so the project itself processes no one's data. A single-household instance plausibly falls under the GDPR household exemption, which bank-aggregation.md §7.2 had ruled out for friends (judgment, M).
- Still valid from bank-aggregation.md, and not repeated here: bank coverage, the EB data model, the 180-day consent, the 4/day cap with 2 background slots to start, PSU headers, the lack of AIS webhooks, and file import as the fallback. One exception: an app coming to the foreground is not a user-present fetch (§4.5).
- New constraints:
  - arbitrary self-hosted domains, which matter for callbacks;
  - one App Store build serving every instance;
  - every member must do the EB setup themselves;
  - the instance owner holds other members' keys as a trusted operator.

## 2. How self-hosted finance apps handle provider credentials

| App (version, license) | Bank-data providers | Where keys live | Scope | Renewal UX | Connector structure |
|---|---|---|---|---|---|
| **Actual Budget** v26.11.0 (2026-11-02), MIT | Enable Banking (experimental), GoCardless (closed to new accounts since July 2025), SimpleFIN, Pluggy.ai, Akahu ([docs](https://actualbudget.org/docs/advanced/bank-sync/)) | Plain values in a `secrets` table of the server's account DB, set through the UI ([secrets-service.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/services/secrets-service.js)) | Global (admin) or per budget file (file owners) ([app-secrets.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app-secrets.js)). EB: "a single set of Enable Banking credentials for the whole server" ([EB doc](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/)) | Account banner "experiencing connection problems" → Reauthorize → re-map each bank account by hand | One server module per provider; the client branches on `account_sync_source` in if/else chains ([sync.ts](https://github.com/actualbudget/actual/blob/master/packages/loot-core/src/server/accounts/sync.ts)) |
| **Firefly III Data Importer** v2.3.5 (2026-09-23), AGPL-3.0 | Enable Banking (since v2.1.0, 2026-02-14), GoCardless, SimpleFIN, Lunch Flow (v1.9.0), Sophtron (v2.0.0, sponsored), Akahu, CSV, CAMT. Spectre (Salt Edge) was deprecated in v2.0.0 and its code removed in v2.2.2 ([releases](https://github.com/firefly-iii/data-importer/releases)) | `.env`: `ENABLE_BANKING_APP_ID`, `ENABLE_BANKING_PRIVATE_KEY`, or `_FILE` variants ([.env.example](https://github.com/firefly-iii/data-importer/blob/main/.env.example)). Or typed into the UI, "less convenient as you need to provide these each time" ([EB tutorial](https://github.com/firefly-iii/docs/blob/main/docs/docs/tutorials/data-importer/eb.md)) | One importer instance; EB session IDs are saved in the user's import config | Re-run the import flow | `RoutineManagerInterface` plus a `ConversionRoutineFactory` that picks a routine per flow ([interface](https://github.com/firefly-iii/data-importer/blob/main/app/Services/Shared/Conversion/RoutineManagerInterface.php)) |
| **Maybe** v0.6.0, AGPL-3.0, archived 2025-07-24 | Plaid (US, EU), Synth | Env vars or an instance `Setting` | Instance | — | `Provider::Registry` keyed by concept ([registry.rb](https://github.com/maybe-finance/maybe/blob/main/app/models/provider/registry.rb)) |
| **Sure** (Maybe fork) v0.7.6 (2026-10-31), AGPL-3.0 | ~28 adapters, incl. Enable Banking (since 2025-11-29), Plaid EU, TrueLayer, SimpleFIN, Lunch Flow | Per household (`belongs_to :family`). Certificate and session are encrypted with Rails `encrypts`, but only when encryption keys are configured ([item](https://github.com/we-promise/sure/blob/main/app/models/enable_banking_item.rb)) | Household. Since 2026-09-18 an item can declare `credential_scope :per_connection` (a member may connect their own) or `:tenant_wide` (admin only); so far only Plaid has opted in ([ownable](https://github.com/we-promise/sure/blob/main/app/models/concerns/provider_item_ownable.rb), [PR #3613](https://github.com/we-promise/sure/pull/3613)) | `needs_authorization?` → re-auth | `Provider::Factory.register` + `Provider::Base` adapters ([adapter](https://github.com/we-promise/sure/blob/main/app/models/provider/enable_banking_adapter.rb)) |
| **Ghostfolio** 3.98.0 (2026-11-04), AGPL-3.0 | None for banks; market data only. Import via CSV/JSON or API | Env vars (`API_KEY_COINGECKO_*`…). The maintainers' paid data provider is "included with a Ghostfolio Premium subscription via an API key" ([README](https://github.com/ghostfolio/ghostfolio), [FAQ](https://github.com/ghostfolio/ghostfolio/blob/main/apps/client/src/app/pages/faq/self-hosting/self-hosting-page.html)) | Instance | — | Data-provider services |
| **Kresus** 0.26.0 (2026-09-25), AGPL-3.0-or-later ([npm](https://registry.npmjs.org/kresus)); the repo moved to [git.kresus.org](https://git.kresus.org/kresus/kresus) in September 2026 | Scraping through woob (LGPL-3.0-or-later, [pyproject](https://gitlab.com/woob/woob/-/blob/master/pyproject.toml)) | Bank passwords in the DB. In standalone mode, "No data will be encrypted … not even your banking password" ([FAQ](https://kresus.org/en/faq.html)) | Per bank access | Bank 2FA comes back to the user as a user action | `Provider {getBankVendors, fetchAccounts, fetchTransactions}` with a `SessionManager`, an `isInteractive` flag and `UserActionResponse` ([providers](https://codeberg.org/kresus/kresus/src/branch/main/server/providers/index.ts), Codeberg archive of 2026-09-21; unchanged in the 0.26.0 npm package) |

Lessons:

- **Instance-wide credentials leak across members.** Actual's threat model says roles "are **not** a security boundary between users on the same server". For providers whose global credential is tied to one person, "every user on the server can then read the accounts and transactions behind it" ([SECURITY.md](https://github.com/actualbudget/actual/blob/master/SECURITY.md)). Sure retrofitted per-member ownership in September 2026. on-budget's "private by default" principle needs member-scoped credentials from day one. (H)
- **The code is exchanged on the server.** Actual's unauthenticated `/auth_callback` swaps the code for a session, then shows "Authorization successful. This window will close"; the client polls by `state`. The pending state lives in memory, so "Multi-instance deployments require sticky routing" ([app-enablebanking.ts](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app-enablebanking/app-enablebanking.ts)). Its iOS app doesn't come back from Safari on its own ([#9443](https://github.com/actualbudget/actual/issues/9443), open, secondary). (H)
- **EB bugs to test for** (secondary, from issue titles and release notes):
  - a bank that repeats the `continuation_key` caused duplicates ([Actual #8846](https://github.com/actualbudget/actual/issues/8846));
  - a bank renumbering a day's batch overwrote unrelated rows ([Actual #8701](https://github.com/actualbudget/actual/issues/8701));
  - session-scoped account IDs broke dedupe ([Firefly #12093](https://github.com/firefly-iii/firefly-iii/issues/12093));
  - a 90-day consent computed across a DST change got a 422 ([Firefly #12553](https://github.com/firefly-iii/firefly-iii/issues/12553));
  - a private IP sent as `Psu-Ip-Address` was rejected by the bank ([Actual #8345](https://github.com/actualbudget/actual/issues/8345)).
- **Credential shapes differ, and the interface must hide them.** (H)
  - EB: an app key that signs RS256 JWTs and reaches every session of that app.
  - SimpleFIN: a one-time token, claimed for an Access URL with Basic-auth credentials, to be stored "at least as securely as the user's financial data" ([protocol](https://www.simplefin.org/protocol.html)).
  - woob: the bank password itself.

## 3. Enable Banking terms for one app per member

Terms of Service, last updated 2026-01-09 ([EB ToS](https://enablebanking.com/terms)), plus the docs and the registration form:

| Clause (verbatim) | What it means for on-budget |
|---|---|
| "Use of the API in the Production Environment is limited to Linked Accounts and is available solely for evaluation purposes or for the personal use of private individuals. The number of Linked Accounts per Application may be limited." | A household instance is personal use. EB publishes no cap (spike 1). |
| "These Terms do not grant any right or license to use the API for any business or professional purpose, including, without limitation, (i) accessing account information belonging to businesses, (ii) accessing account information that does not belong to the Control Panel user who associated the Linked Accounts with the application accessing them, or (iii) using the API for any commercial purpose whatsoever." | (ii) follows whoever linked each account, not whoever registered the app: nobody may link someone else's accounts. EB lets an app be shared "with another user who has registered an Enable Banking account", with MFA required ([Control Panel](https://enablebanking.com/docs/api/control-panel/)), but says nothing on whether that user may link their own accounts to it (spike 1). So one app per member is on-budget's choice, not EB's rule. A joint account belongs to both co-holders, so either one may link it (M). |
| "business users may register and activate Applications in the Production Environment … Any such use shall be considered solely for evaluation purposes", until EB "provides written notice" | (i) leaves business accounts out of personal use. Treat a member's professional account, sole-trader ones included, as outside EB's free mode (judgment: the ToS don't define "businesses"). It needs file import or an EB contract, and self-hoster docs must say so. |
| "'Linked Account' refers to a Payment Account, which you associate with an Application through the Control Panel by going through the SCA and consent flows" | Each account needs one SCA in the Control Panel, then a separate API consent with another SCA: "This applies even when using the same account" ([linked accounts](https://enablebanking.com/docs/api/linked-accounts/)). |
| "You shall not license, sublicense, … distribute, disclose, or make accessible to any third party … or grant any right to access or use the Control Panel or the API to any third party." | Open point: is a household server administered by another member a "third party" holding the key? The ToS is silent. EB's registration form adds: "Keep the private key in secret (e.g. don't expose it to client-side, share with anyone nor embed into mobile or other apps…)". Unresolved until EB confirms (M–L). |
| "You shall not … automate usage in an abusive manner", nor "access, use, or copy any portion of the Control Panel or the API, through the use of bots, spiders, web crawlers, indexing agents, or other automated devices or mechanisms" | Read literally, the second clause covers the server-side scheduler the whole design needs. EB's FAQ expects fetches "based on a predefined schedule", so it likely targets scraping (judgment). Ask EB in writing (spike 1). |
| "You must provide accurate information, keep credentials secure, and protect any private keys or tokens used for authentication with the API." | Encrypt the key at rest, never export it, never ship it to a client. |
| A restricted app lets "you (and any user of your application)" authorise, but "the API will only return the specific accounts that were originally linked" ([FAQ](https://enablebanking.com/docs/faq/)) | An account that wasn't linked comes back as an empty list, so show a hint to link it in the Control Panel first. Even an operator holding the key can't add accounts without the member's SCA. |
| "applications can't be made available to the public before a contract is signed" ([FAQ](https://enablebanking.com/docs/faq/)) | Instances stay closed to their households, as PRODUCT.md already says. Anyone hosting on-budget for others needs their own EB contract, so the code gives a SaaS clone no free bank access. |
| In restricted activation, "the terms and privacy links and the data protection email are not checked" ([FAQ](https://enablebanking.com/docs/faq/)) | The wizard can prefill the instance URL in those fields, as Firefly III's tutorial does. |
| End users accept EB's own terms ("separate consent") before any bank interaction ([ToS](https://enablebanking.com/terms)) | Every member sees EB's terms screen during each consent. |
| "These Terms grant only a limited, revocable right to access the Control Panel and the API"; "Enable Banking may modify or discontinue any functionality at any time"; "Continued use of the Control Panel or API after changes constitutes acceptance of the updated Terms"; liability "limited to EUR 100" | The free tier can change or end at EB's will. Keep file import first-class and the connector SDK ready for another provider (trade-offs). |

Limits:

- **Unattended fetches.** RTS art. 36(5)(b): "where the payment service user does not actively request such information, no more than four times in a 24-hour period, unless a higher frequency is agreed between the account information service provider and the account servicing payment service provider, with the payment service user's consent". EB: "many ASPSPs have the limit of 4 times a day", detected from the absence of PSU headers; after `ASPSP_RATE_LIMIT_EXCEEDED`, "continue data fetching after 6 hours" ([FAQ](https://enablebanking.com/docs/faq/)). (H)
  - An "active request" is "the PSU being in session at the time of the request and proactively performing an action, predefined by the AISP". It excludes the case where the AISP is mandated "to refresh automatically the information whenever the PSU is accessing the AISP application(s)" ([EBA Q&A 2018_4172](https://www.eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4172)). (H)
  - Each of the four accesses "may, for example, include both the account balance and associated payment transactions", through "one or several API calls" ([EBA Q&A 2018_4210](https://www.eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4210)). Banks don't all count that way: "Banks may impose rate limits, down to 4 API calls per day for every account. Each endpoint (details, balances, transactions) has it own rate limit" ([GoCardless](https://docs.gocardless.com/docs/bank-account-data)). (H)
- **PSU headers.** "When at least one PSU header is included", the bank treats the user as present, and its background limits "do not apply". Send "either all required PSU headers (in case of online data fetching) or none of PSU headers (in case of background data fetching)"; each bank's required headers come from `GET /aspsps` ([FAQ](https://enablebanking.com/docs/faq/)). Since January 2026, EB's gateway rejects a request in which "A required PSU header is missing" or "A PSU header value is in an invalid format" ([changelog](https://enablebanking.com/blog/2026/02/17/enable-banking-changelog-january-2026)). (H)
- **Consent length.** `valid_until` can go up to `maximum_consent_validity`, which "for the majority of ASPSPs … corresponds to 180 days". Sessions can still end early with `EXPIRED_SESSION`, for example after a yearly KYC survey, when a new session cancels the old one, or when the bank doesn't support "seamless migration between old and new certificates" (§4.4) ([FAQ](https://enablebanking.com/docs/faq/)). Actual's docs say "90 days at most" (secondary), and Firefly #12553 shows a bank capped at 7,776,000 s (90 days). Read the value for each bank. (H)
- **Isolation by key.** "With one authorisation token you are able to access all account information sessions and payments initiated by the application." So one member's key never reaches another member's sessions ([FAQ](https://enablebanking.com/docs/faq/)). (H)
- **API quotas** "per second, per hour and per day may apply" (ToS); the figures aren't published.
- **Matching across sessions.** Re-authorisation changes both the session ID and the account IDs. Match on `identification_hash`, which is stable "across different sessions and even different users". This is also how two partners' links to one joint account get merged ([FAQ](https://enablebanking.com/docs/faq/)). (H)
- **The PSR will change these rules.** They come from the PSD2 RTS. The Payment Services Regulation was agreed on 2025-11-27 and "needs to be formally adopted" ([EP](https://www.europarl.europa.eu/news/en/press-room/20251121IPR31540/payment-services-deal-more-protection-from-online-fraud-and-hidden-fees)). It awaits Parliament's second reading ([OEIL](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0210(COD))) and would apply around late 2028 (bank-aggregation.md §7.3). Users get a dashboard "to monitor and manage the permissions they have given", run by the bank in the Commission's proposal ([COM(2023) 367](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52023PC0367)), so a member can also revoke a consent there. That proposal also moved the 180-day re-authentication from the bank to the AISP. Whether the final text keeps the 4/day and 180-day rules is unverified: re-check them once it is published.

## 4. Setup, key custody and callbacks

### 4.1 Member onboarding (once per member, about 10 minutes, judgment)

1. In on-budget, the member opens Settings → Bank connections → Set up Enable Banking. The server generates an RSA-4096 key pair and a self-signed certificate for that member. The screen shows the certificate PEM, an app name, the callback URL and the instance URL, each with a copy button.
2. The member signs in to the [EB Control Panel](https://enablebanking.com/cp/applications) (email link) and registers a **Production** app:
   - choose "Generate outside the browser and import public certificate" and paste the certificate;
   - put the callback URL under "Allowed redirect URLs (one per line)";
   - put the instance URL in the privacy and terms fields, which restricted mode doesn't check.
3. The member presses "Activate by linking accounts" and passes the bank's SCA for each account. The app becomes "Active", with Account Information "Restricted".
4. The member pastes the Application ID into on-budget. The server calls `GET /application` with a JWT signed by the key ([API ref](https://enablebanking.com/docs/api/reference/)). It checks `active`, AIS in `services`, `environment: PRODUCTION`, and that `redirect_urls` contains the instance callback. On a mismatch, it says what to fix.
5. The member connects a bank:
   - choose it from `GET /aspsps`;
   - on-budget sends `POST /auth` with `valid_until`, `state`, `redirect_url` and `psu_type: personal`;
   - bank SCA, callback, `POST /sessions`;
   - the member picks which accounts the household can see; the default is private.

Fallback: accept an uploaded PEM from members who already generated a key in the browser, as Actual and Firefly III do, and tell them to delete the downloaded file.

### 4.2 Where keys live

| Item | Storage | Who can use it |
|---|---|---|
| Instance key-encryption key (KEK) | A secret file outside the DB and outside DB backups: a systemd credential, sops/agenix on NixOS, or a Docker secret | The server process |
| Member's EB private key | DB, envelope-encrypted with a per-record data key (AEAD). The associated data is the member ID plus the credential ID, so a row can't be moved to another member | Server jobs acting for that member. No API returns it; only the certificate can be exported |
| EB `session_id`, `valid_until`, bank, status | DB; the session ID is encrypted | Same |
| Raw provider payloads | DB, encrypted, kept so data can be re-processed | Same |
| PSU IP and user agent | Request memory only, never persisted. Sure stores `last_psu_ip`, then clears it once stale | Explicit refreshes only (§4.5) |

- The owner sees each connection's health (ok, expiring, needs re-auth, bank down, rate-limited). On offboarding the owner can disable or delete a member's connector: wipe the key and call `DELETE /sessions/{id}`, which closes the bank consent "if possible". There is no way to reveal a key.
- Self-hoster docs must say it plainly: whoever controls the server can, with effort, extract any key. on-budget makes members' credentials an application-level boundary, not a defence against the operator (judgment). Actual, by contrast, treats every user on a server as trusted.
- Losing the KEK loses every key, and members must then register a new certificate. Whether an EB app can swap its certificate is unverified (spike 1).

### 4.3 Consent callback on a self-hosted domain

- **Redirect URL.** Use `https://<instance>/connect/enable-banking/callback`, whitelisted in each member's app.
  - EB's own docs set no rule: `redirect_urls` is a "List of allowed redirect URLs for the application" ([API ref](https://enablebanking.com/docs/api/reference/)). Third-party docs report one: Actual says EB "only sends you back to addresses that start with https://" and rejects its `app://` URL; Firefly III says "Production environment requires HTTPS … The URL does not have to be public."
  - The redirect is a hop in the browser, so a VPN-only or LAN-only instance should work if the device reaches it at that moment. **M** until the callback spike confirms that EB accepts a tailnet-only host (spike 3).
  - If the instance URL changes, every member must update their app; the `GET /application` check catches a mismatch.
- **State.** 128 random bits, single use, valid ~30 minutes, stored in Postgres rather than process memory, together with the member, connector, bank and a nonce. The callback route has to be unauthenticated: it only completes the pending record and never renders account data.
- **Exchange.** The server calls `POST /sessions` with the `code` at once, stores the session, then responds:
  - web app: same-origin redirect to the connection page;
  - native app: the link started in `ASWebAuthenticationSession`, so the callback page redirects to `onbudget://connections/<pending-id>?result=ok`, with no code or session in the URL. Apple's `.https(host:path:)` callback (iOS 17.4+, macOS 14.4+) requires that "the host must be a member of a domain associated with the app", which an App Store build can't list for arbitrary self-hosted domains. A custom scheme works with any domain: `.customScheme(_:)` on iOS 17.4+ and macOS 14.4+, or the older `callbackURLScheme` initializer on iOS 12+ and macOS 10.15+, which Apple's docs mark deprecated at 27.3 ([https](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/https(host:path:)), [custom scheme](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:)), [initializer](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/init(url:callbackurlscheme:completionhandler:)-7fpox)).
  - EB warns that an SCA app "may not always recognise which app it was launched from" and that you "should assume that the user might be returned to another web browser" ([FAQ](https://enablebanking.com/docs/faq/)). So the page also shows a "Return to on-budget" button, and the app polls the pending record when it comes to the foreground.
- Never open the bank flow in a WebView: the switch to the bank's SCA app can fail there ([FAQ](https://enablebanking.com/docs/faq/)).

### 4.4 Consent lifecycle and renewal UX

- Request the bank's `maximum_consent_validity`, capped at 180 days, minus a small margin. Compute it in UTC seconds, which avoids the DST bug Firefly #12553 hit.
- Remind only the member who owns the connection, at T−14, T−3 and T−0 days, in the app and with local notifications that each of that member's devices schedules from the synced `valid_until`. These need neither APNs nor a relay ([Apple](https://developer.apple.com/documentation/usernotifications/scheduling-a-notification-locally-from-your-app)). An instance can't push to the maintainer's store builds: the APNs signing key "must remain private to prevent anyone else from generating those tokens" ([Apple](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns)), and a relay would be a service run for others (self-hosting.md). Early expiries surface at the next sync, or by Web Push in the web app. Other members see a neutral "needs renewal" badge on shared accounts only (judgment).
- **Renew** sends a new `POST /auth` for the same bank. On success, accounts are re-attached automatically by `identification_hash`, then `identification_hashes`, then IBAN; Actual makes users re-map by hand. Then the old session is deleted.
- Treat `EXPIRED_SESSION` at any time as "needs re-auth". EB says to "base their logic on errors rather than response codes" ([FAQ](https://enablebanking.com/docs/faq/)).
- **Bank-wide waves.** At banks that "bind account information consents to specific certificates", EB's QWAC renewal in January 2026 meant "certain account information sessions expired prematurely" ([EB](https://enablebanking.com/blog/2026/02/17/enable-banking-changelog-january-2026)). QWAC lifetimes fall to 100 days from March 2027 and 47 days by March 2029, which means "re-registering certificates with every connected bank every few months", unless EB's test of EU-Trusted-List-only CAs, whose QWACs last up to three years, succeeds ([EB](https://enablebanking.com/blog/2026/12/08/enable-banking-changelog-july-2026)). Expect waves that end every member's consent at one bank at once, on top of each member's own cycle (judgment). The owner's health view groups failures by bank, and the reminder says the bank ended the consent.
- A new account at the bank must be linked in the Control Panel first. Detect "authorised, but 0 accounts" and explain this.
- Link to EB's [consent page](https://enablebanking.com/data-sharing-consents/), where members can review or revoke consents outside on-budget.

### 4.5 Fetch budget and PSU headers

- **Background:** start with 2 slots per consent per day (for example 07:00 and 19:00 instance time, plus or minus jitter), as bank-aggregation.md does. A slot is at least 2 calls (balances, then paginated transactions), and some banks count every call (§3). Move a bank to 3–4 slots only after measuring how it counts (spike 4). Send no PSU headers, and back off at least 6 hours after a rate limit. **M** on the slot count.
- **Explicit refresh:** only a predefined action, such as pull-to-refresh or a refresh button, makes a fetch user-present. The server then fetches the consents that member authorised, with that member's PSU headers: at least every header in the bank's `required_psu_headers` (§3). An app coming to the foreground is not an active request ([EBA Q&A 2018_4172](https://www.eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4172)): it syncs from the instance, and may use a background slot left that day, without PSU headers.
- **PSU IP on a private network.** An instance reachable only over Tailscale ([ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md)) sees each device's tailnet address in 100.64.0.0/10 ([Tailscale](https://tailscale.com/docs/concepts/tailscale-ip-addresses)), never its public IP. Options, settled per bank in spike 4; until then, an explicit refresh that a bank rejects falls back to a background slot:
  - send the tailnet address, if the bank accepts it; one bank rejected a private IP ([Actual #8345](https://github.com/actualbudget/actual/issues/8345), secondary);
  - where `required_psu_headers` doesn't list `Psu-Ip-Address`, send only the headers it lists;
  - let the device look up its own public IP and send it with the refresh, which needs an outside lookup from the device;
  - avoid Firefly III's approach of a configured IP or one auto-detected through icanhazip.com ([.env.example](https://github.com/firefly-iii/data-importer/blob/main/.env.example)): that is the server's public address, which matches the member's only when both share a connection, so on a VPS it misstates the PSU's IP (judgment).
- **Joint account linked by both partners:** run one fetch per slot across both consents. Whether banks count the cap per consent, per app or per account is unknown (spike 2).
- File imports don't count against the budget.

## 5. A connector interface for on-budget

### 5.1 Responsibilities

| The core (one implementation) | Each connector |
|---|---|
| Members, households, sharing, permissions | Provider auth: JWT signing, token claims, file decoding |
| Credential vault, KEK, audit log | Link flow: start URL, callback completion, revoke |
| Scheduling, fetch budget, back-off | Account and transaction fetch, pagination |
| Dedupe, pending→booked matching, merges, rules, categories | Mapping provider fields to the normalised model |
| Notifications, health, renewal prompts | Mapping provider errors to typed errors |
| Encrypted storage of raw payloads | Declaring its capabilities and limits |

### 5.2 Interface sketch (TypeScript; a Swift protocol maps one to one)

```ts
interface ConnectorManifest {
  id: "enable-banking" | "ofx" | "qif" | "csv";
  link: "redirect" | "file" | "token";        // token = SimpleFIN-style, later
  credential: "per-member" | "none";          // "per-instance" only for a contracted provider
  background: false | { maxPerDay: number };  // EB: 4 (RTS art. 36(5)(b))
  features: ("balances" | "pending" | "counterparty" | "mcc")[];
}
interface Connector {
  manifest: ConnectorManifest;
  checkCredential?(ctx: Ctx): Promise<CredentialStatus>;           // EB: GET /application
  startLink?(ctx: Ctx, r: { bank: string; validUntil: Date; state: string }): Promise<{ url: string }>;
  completeLink?(ctx: Ctx, cb: { code: string; state: string }): Promise<Consent>;
  listAccounts(ctx: Ctx, c: Consent): Promise<ProviderAccount[]>;
  fetch(ctx: Ctx, a: AccountRef, cursor: Cursor | null,
        mode: "background" | { psu: PsuHeaders }): Promise<Page<ProviderTxn>>;
  revoke?(ctx: Ctx, c: Consent): Promise<void>;                    // EB: DELETE /sessions/{id}
  parse?(ctx: Ctx, file: Uint8Array, hints: FileHints): AsyncIterable<ProviderTxn>;
}
interface ProviderTxn {
  accountKey: string;       // EB identification_hash; OFX BANKID + ACCTID
  externalId?: string;      // EB entry_reference; OFX FITID; none for QIF/CSV
  status: "booked" | "pending";
  bookedOn?: string; valueOn?: string;                 // ISO dates
  amount: { minor: bigint; currency: string };         // signed
  description: string; counterparty?: string; mcc?: string;
  raw: unknown;                                        // stored encrypted
}
// Typed errors: NeedsReauth | RateLimited(retryAt) | BankUnavailable
//               | Misconfigured(hint) | UserActionRequired | InvalidFile
```

- **Context.** `Ctx` gives a connector an HTTP client (recordable, with redaction), a clock, that member's own secrets and nothing else, a logger that strips personal data, and the instance callback URL. A connector can't read the DB, other members or other connectors.
- **Fingerprints.** The SDK computes a fingerprint (date, signed amount, normalised description, running balance when present) for rows without a stable `externalId`, so dedupe lives in one place.
- **A static registry, not runtime plugins.** Connectors are in-repo packages listed in one file and enabled per instance by config. Loading third-party code at runtime next to bank keys is a supply-chain risk (judgment). Precedents: Firefly III's factory, Sure's `Provider::Factory`, and Kresus, which scans a directory.
- **Where connectors run: on the server.** A file picked on a device is queued offline and uploaded as is. One parser then serves iOS, macOS, web and the watch (judgment). The cost: no offline preview of an import.

### 5.3 Capabilities per connector

| Connector | Link | Credential | Background | Stable IDs | Notes |
|---|---|---|---|---|---|
| Enable Banking (restricted) | Redirect, after linking in the Control Panel | Per member: app ID + key | ≤4 per account per day | `identification_hash`; `entry_reference` (booked rows) | Consent usually 180 days |
| OFX 1.x / 2.x | File | None | — | FITID: unique per account per the spec, but some banks reissue it (secondary, [KMyMoney](https://mail.kde.org/pipermail/kmymoney-devel/2018-February/020182.html)) | SGML (1.x) vs XML (2.x) |
| QIF | File | None | — | None → fingerprint | Ambiguous date formats |
| CSV (one profile per bank) | File | None | — | None → fingerprint | `;` separators, decimal comma, encodings (bank-aggregation.md §8) |
| Later: contracted AISP (EB unrestricted, Powens, Bridge) | Redirect or webview | Per instance, under a contract | Provider-scheduled, webhooks | Provider IDs | Needs a legal entity; off by default for self-hosters |
| Later: token providers (SimpleFIN-style, Lunch Flow) | Token | Per member | Provider cadence | Provider IDs | Lunch Flow resells GoCardless (bank-aggregation.md); SimpleFIN covers North America |

### 5.4 Testing

- **A contract kit in the SDK** runs every connector against a fake provider server. Its scenarios:
  - idempotent re-fetch and pagination;
  - a repeated `continuation_key` and a renumbered batch;
  - pending→booked with a changed ID, and session-scoped IDs after re-auth;
  - `EXPIRED_SESSION` mid-fetch or across every consent at one bank, and a 429 with `ASPSP_RATE_LIMIT_EXCEEDED`;
  - a restricted app that returns zero accounts;
  - a 90-day consent cap across a DST change.
- **The EB sandbox,** which per the ToS has mock data and a "Mock ASPSP" in the Control Panel, runs a nightly live smoke test with no real accounts.
- **Golden files** for OFX, QIF and CSV, built synthetically from each supported bank's export layout. Property tests check that parsing twice gives the same keys and that overlapping exports produce no duplicates.
- **No real bank data in the repo**, which is public (AGENTS.md). Captures from real accounts stay local and go through a scrubber before they become fixtures.

## 6. The license

### 6.1 What peers use

| Project | License | Contributions | Maintainers' hosted offer | Name |
|---|---|---|---|---|
| Actual Budget | MIT | No CLA or DCO in CONTRIBUTING (checked) | None; [PikaPods](https://www.pikapods.com/apps) hosts it | — |
| Firefly III | AGPL-3.0-or-later | No CLA or DCO found | None; PikaPods hosts it | — |
| Immich | AGPL-3.0, iOS app [in the App Store](https://apps.apple.com/app/id1613945652) | No CLA or DCO found; no App Store exception found | Paid product keys; PikaPods hosts it | Resellers are asked not to display Immich trademarks ([FAQ](https://github.com/immich-app/immich/blob/main/docs/docs/FAQ.mdx)) |
| Ghostfolio | AGPL-3.0 | — | Ghostfolio Premium: SaaS and a data API | — |
| Maybe → Sure | AGPL-3.0 | — | Maybe pivoted to B2B and archived the repo ([v0.6.0](https://github.com/maybe-finance/maybe/releases/tag/v0.6.0)) | "'Maybe' is a trademark of Maybe Finance Inc. and therefore, use of it is NOT allowed in forked repositories", so the fork is called Sure ([README](https://github.com/we-promise/sure)) |
| Kresus | AGPL-3.0-or-later | — | None | — |
| Nextcloud iOS | GPL-3.0 "with Apple app store exception" | "doesn't require a CLA"; DCO enforced ([README](https://github.com/nextcloud/ios)) | — | — |
| Mastodon iOS | GPL-3.0, "also dual-licensed to Apple" | CLA ([README](https://github.com/mastodon/mastodon-ios)) | — | Modified versions must "use a unique name and icon" |
| Signal iOS, libsignal / Element X iOS | AGPL-3.0 | CLA ([Signal](https://github.com/signalapp/Signal-iOS/blob/main/CONTRIBUTING.md); libsignal: CLA "required for all contributions", no App Store permission, [README](https://github.com/signalapp/libsignal)) / dual AGPL + commercial license ([Element](https://github.com/element-hq/element-x-ios)) | — | — |
| Bulwark Mobile (mail, iOS and Android) | AGPL-3.0-only with a provisional §7 app-store permission since 2026-09-30 ([LICENSE](https://github.com/bulwarkmail/native/blob/main/LICENSE)) | Earlier contributors asked to consent ([discussion #1113](https://github.com/orgs/bulwarkmail/discussions/1113)) | — | — |
| Swiftfin (Jellyfin) / Home Assistant iOS | MPL-2.0 / Apache-2.0 | — | — | — |

PikaPods hosts Actual, Firefly III, Immich and Sure, with a "20% revenue share with project authors where possible" ([PikaPods](https://www.pikapods.com/)). AGPL doesn't stop commercial hosting of unmodified code; it only stops closed modifications. (H)

### 6.2 Options compared

| | MIT | Apache-2.0 | MPL-2.0 | AGPL-3.0 | EUPL-1.2 |
|---|---|---|---|---|---|
| A modified server offered over a network must publish its source | No | No | No | Yes (§13) | Yes: "providing access to its essential functionalities" counts as Communication |
| A closed fork of the apps | Allowed | Allowed | Modified MPL files stay open; new files can be closed | Not allowed | Not allowed, but combined works may move to any listed "Compatible Licence", MPL-2.0 and LGPL included |
| App Store / TestFlight | No known conflict | No known conflict | §3.2(b): executables can be sublicensed if source rights are kept | Conflict as the FSF reads it; needs an exception or a CLA | Same problem: "cannot offer or impose any additional terms" |
| Explicit patent grant | No | Yes (§3) | Yes | Yes (§11) | Yes |
| Trademarks | Silent | Excluded (§6) | Excluded (§2.3) | §7(e) lets the author decline trademark rights | Excluded |
| Friction for contributors and companies | Lowest | Low | Low | High for some: at Google it "MUST NOT be used" ([policy](https://opensource.google/documentation/reference/using/agpl-policy)) | Medium (less known); French text has equal legal value |
| Changing license later | Easy to tighten | Easy to tighten | File by file | Needs every contributor's consent, or a CLA | Same as AGPL |

FSL-1.1 is a "Fair Source" license that becomes Apache-2.0 or MIT after two years ([fsl.software](https://fsl.software/)). It is not open source, and neither is BSL, so both conflict with PRODUCT.md's open-source commitment.

### 6.3 GPL-family code in Apple's stores

- **The FSF's reading (2010, after Apple removed GNU Go).** App Store Usage Rules are "exactly the kind of 'further restrictions' that are prohibited by the GPL", and "this analysis would apply to all versions of the GNU GPL and AGPL" ([FSF](https://www.fsf.org/blogs/licensing/more-about-the-app-store-gpl-enforcement)). AGPL §10: "You may not impose any further restrictions on the exercise of the rights granted".
- **Apple's terms today.** The clause the FSF quoted, "The Usage Rules shall govern your rights with respect to the Products, in addition to any other terms or rules that may have been established between you and another party", is gone from the Media Services Terms (last updated 2026-09-14, [Apple](https://www.apple.com/legal/internet-services/itunes/us/terms.html)). Other clauses now have the same effect; cite them in the lawyer review:
  - "Your use of the Services and Content must follow" the Usage Rules. Content includes apps, and the rules include use "only for personal, noncommercial purposes" and device limits.
  - An app is licensed under the Standard EULA "unless Apple or the App Provider provides an overriding custom license agreement". The Standard EULA grants use "as permitted by the Usage Rules" and says "You may not transfer, redistribute or sublicense the Licensed Application". Its carve-out for "open-sourced components" covers only its limits on copying, reverse engineering and modifying, not redistribution or the Usage Rules ([Standard EULA](https://www.apple.com/legal/internet-services/itunes/dev/stdeula/)).
  - So the FSF's conclusion likely still holds (judgment, M).
- **Who is at risk.** A copyright holder can always publish their own code. The risk is contributors' code. Immich ships AGPL code to the App Store with no exception found, which shows the risk is often simply accepted (secondary inference).
- **How peers solved it:**
  - Nextcloud iOS: "we have committed not to pursue any license violation that results solely from the conflict between the GNU GPLv3 … and the Apple App Store terms of service", plus a DCO and no CLA ([COPYING.iOS](https://github.com/nextcloud/ios/blob/master/COPYING.iOS)).
  - Signal's libsignal-protocol-c, archived, last pushed in July 2020: a GPLv3 "additional permission to convey through the Apple App Store non-source executable versions of the Program … only under the Mozilla Public License version 2.0" ([README](https://github.com/signalapp/libsignal-protocol-c#license)). Its successor, libsignal, is AGPLv3 with a mandatory CLA and no such permission ([README](https://github.com/signalapp/libsignal)), so the wording is a historical model, not current Signal practice.
  - Bulwark Mobile, AGPL-3.0-only: since 2026-09-30, a provisional §7 permission to convey "through an app store", such as "the Apple App Store and TestFlight", provided the Corresponding Source is available "at no charge", the listing or the app says where, and no further restrictions are added ([LICENSE](https://github.com/bulwarkmail/native/blob/main/LICENSE)). Earlier contributors were asked to consent ([discussion #1113](https://github.com/orgs/bulwarkmail/discussions/1113)), which is the cost of adding it late.
  - Mastodon iOS: dual-licensed to Apple, so "any contributors are required to sign a Contributor License Agreement".
- **GPLv3/AGPLv3 §7.** Additional permissions "applicable to the entire Program shall be treated as though they were included in this License", and downstream copies may remove them ([AGPL](https://www.gnu.org/licenses/agpl-3.0.txt)).
- **Third-party GPL code in the app isn't covered** by our exception. Keep store-shipped dependencies permissive or MPL. Apache-2.0 is "compatible with version 3 of the GNU GPL" ([FSF list](https://www.gnu.org/licenses/license-list.html#apache2)). MPL-2.0 names AGPL-3.0 as a "Secondary License" ([MPL](https://www.mozilla.org/en-US/MPL/2.0/)).

### 6.4 Contributions

- **inbound=outbound.** Under GitHub's ToS D.6, content added to a repo with a license notice is licensed "under the same terms", unless a separate agreement such as a CLA says otherwise ([GitHub ToS](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service#6-contributions-under-repository-license)). Apache-2.0 §5 writes the same rule into the license.
- **DCO 1.1.** A `Signed-off-by` line certifies the right to submit "under the open source license indicated in the file" ([DCO](https://developercertificate.org/)). It costs contributors one flag; enforce it with a CI check. A header reading only `AGPL-3.0-or-later` wouldn't indicate the App Store permission, so name it in every file: `SPDX-License-Identifier: AGPL-3.0-or-later WITH AdditionRef-<name>`, an expression SPDX 3.0.1 allows for user-defined additions ([SPDX](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)).
- **CLA.** It lets the maintainer relicense or dual-license (Mastodon, Element, Signal), but it adds friction and an imbalance of rights. Not needed if the exception exists from the first commit.

### 6.5 Trademark and the name

- **No license above grants trademark rights.** Forks and hosted copies are controlled through trademark, not copyright: Maybe's fork had to become "Sure", and Mastodon iOS and Immich publish usage rules.
- **France.** "La propriété de la marque s'acquiert par l'enregistrement" (CPI L712-1, [reproduction](https://www.baumann-avocats.com/CPI/article-l712-1-du-code-de-la-propriete-intellectuelle.php), secondary). Without registration, only weaker claims such as unfair competition remain (judgment).
- **Costs, as of Nov 2026, 10 years each:**
  - INPI: €190 for one class + €40 per extra class (fees applicable from 2026-07-02, [INPI](https://www.inpi.fr/inpi-block/download-document?id=20516));
  - EU trade mark: €850 online + €50 for the second class + €150 for each further class, renewable "10 years at a time" ([EUIPO](https://www.euipo.europa.eu/en/trade-marks/before-applying/fees-payments), matching [EUTMR Annex I](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32017R1001));
  - classes to consider: 9 (downloadable software), 42 (software and hosting services), possibly 36 (financial information).
- **The working name is weak.** EUTMR art. 7(1)(b) refuses marks "devoid of any distinctive character", and (c) refuses signs that "may serve, in trade, to designate the kind, quality, quantity, intended purpose…"; "on-budget" for budgeting software is exposed to both (judgment, M). Similar names are in use: App Store apps listed as "On Budget" and "onBudget" ([listing](https://appgoblin.info/apps/6499044750), secondary, L), and a live US mark ONBUDGET (Reg. 4498672, renewed, status dated 2023-11-17) for "Credit and debit card services" in class 36 ([USPTO TSDR](https://tsdr.uspto.gov/statusview/sn85607743)): a card-services mark, not software.

## Recommendation for on-budget

### Bank connectors

1. **Per-member Enable Banking apps only.** on-budget has no instance-wide EB setting. A credential belongs to exactly one member and can only fetch connections that member created. This is on-budget's design, not an EB rule: it keeps keys, sessions and request logs apart (§3). (H)
2. **Generate keys on the server.** The member pastes the certificate into EB; on-budget validates with `GET /application`; PEM upload remains a fallback. (H)
3. **Use an envelope-encrypted vault.** The KEK lives outside the DB and backups, the associated data binds each row to its member, no path reveals a key, and every use is logged. (M–H)
4. **Callbacks:** one HTTPS route per instance, `POST /sessions` on the server, pending state in Postgres, and a custom-scheme bounce plus polling for the native apps. (H; M that EB accepts a tailnet-only host, until spike 3)
5. **Consent lifecycle:** the bank's `maximum_consent_validity` minus a margin; reminders at T−14, T−3 and T−0 to the connection's owner only, in the app and as local notifications; bank-wide expiry waves grouped in the owner's health view; automatic re-match by `identification_hash`; `DELETE /sessions` on disconnect. (H)
6. **Fetch budget:** 2 background slots per consent per day, raised per bank only after measuring. PSU headers only on an explicit refresh, with every header the bank requires; whether a tailnet `Psu-Ip-Address` works decides how that runs on a Tailscale-only instance (spike 4). (M)
7. **Connector SDK and static registry:** Enable Banking, OFX, QIF and CSV in v1. A new connector merges only once it passes the contract kit. (M–H)
8. **Self-hoster docs.** One page per connector: EB's terms in plain words (personal use; each account linked by the person it belongs to; no business accounts, sole-trader ones included; no service for others), why on-budget uses one app per member, the operator-trust caveat, and the wizard's steps. (H)

Trade-offs the owner must accept:

- Every member does a ~10-minute EB setup and passes two SCAs per account. They renew every 90–180 days themselves, and again when a bank-wide wave ends their consents early; the owner can't do it for them.
- EB has no AIS webhooks, so sync polls: twice a day to start, 4 times at most. New transactions arrive hours late unless the member pulls to refresh, and on a Tailscale-only instance even that depends on spike 4.
- Until EB answers spike 1, hosting a partner's key on a server the owner administers rests on a reading of the terms, not a confirmed permission.
- Bank sync in v1 rests on EB's free tier, "a limited, revocable right": EB "may modify or discontinue any functionality at any time", continued use accepts new terms, and its liability is capped at EUR 100. Keep file import first-class and the connector SDK open to another provider, so losing EB costs latency, not data (bank-aggregation.md).
- Members' professional accounts stay on file import.
- There are no runtime plugins: a new provider needs a code change and a release.
- Parsing happens on the server, so an imported file appears only once the server is reachable.

### License

1. **AGPL-3.0-or-later** for the whole monorepo (server, web, iOS, macOS, watch), with an SPDX header in every file that names the permission: `AGPL-3.0-or-later WITH AdditionRef-<name>` (§6.4). Firefly III and Kresus use "-or-later"; the alternative "-only" doesn't depend on any future FSF version. (M)
2. **An App Store additional permission under §7**, in a `LICENSE-EXCEPTIONS.md` present from the first public commit:
   - modelled on Bulwark's AGPL permission or Signal's archived wording, or on Nextcloud's non-enforcement pledge;
   - covering Apple's App Store and TestFlight;
   - with source remaining available under the AGPL;
   - reviewed by a lawyer before the first store build. (M)
3. **DCO sign-off** enforced in CI, with no CLA. CONTRIBUTING and every file's SPDX header state that contributions come under AGPL-3.0-or-later together with the exception. (M–H)
4. **A dependency rule for the clients:** anything shipped through a store uses only permissive or MPL dependencies, enforced by a license check in `mise run check` (factory-ci-quality.md). (M)
5. **A source link** on every app's About screen, pointing at the running commit (AGPL §13). (H)
6. **A TRADEMARKS.md now:**
   - forks and hosted services must rename;
   - nominative use ("unofficial", "for on-budget") is allowed;
   - official store builds come only from the maintainer.

   Pick a **distinctive final name** before the first App Store listing, domain or filing. Register it once final: INPI first (€190–230), then an EU trade mark if users outside France matter. (M)

Alternatives:

- **Apache-2.0**, if simplicity wins: patent grant, contribution and trademark clauses, no store friction. Closed forks and closed hosted modifications become legal; keep the TRADEMARKS.md anyway.
- **AGPL-3.0 for the server and MPL-2.0 for all client code** (the Swiftfin model), if counsel advises against a custom exception. It uses only standard licenses, but the repo then carries two licenses split by path.

Trade-offs of the AGPL route:

- Some companies and developers won't touch AGPL code, and the connectors and parsers can't be reused in proprietary apps.
- Moving to a permissive license later needs every contributor's consent.
- The exception must be in place before the first outside contribution. Adding it later needs everyone's consent, which Bulwark Mobile is collecting now (§6.3).
- The owner must keep third-party GPL code out of the apps.

## Open questions and spikes

1. **Email Enable Banking** (support.api@enablebanking.com) and get answers in writing:
   - Is a member's restricted app, whose key a household server administered by another member holds, "personal use" rather than making the API "accessible to any third party"?
   - What is the linked-account cap per app?
   - Do Control Panel links expire, or only API sessions?
   - Can an app replace its certificate?
   - Is the 4/day cap counted per consent, per app, or per account across apps?
   - May a Control Panel user given shared access to another member's app link their own accounts to it?
   - Does the clause against "automated devices or mechanisms" cover a server-side scheduler fetching 2–4 times a day?
   - Which supported banks bind consents to EB's certificates, as in the January 2026 wave?
2. **Joint-account spike.** Both partners link one joint account to their own apps. Compare `identification_hash`, and check whether one consent's background fetches use up the other's budget.
3. **Callback spike.** iOS and macOS `ASWebAuthenticationSession` with a custom-scheme bounce from a self-hosted HTTPS domain reachable only over Tailscale, including the app switch to each supported bank's SCA app; then the web flow in Safari and Chrome. Confirm that EB accepts the tailnet-only redirect URL (bank-aggregation.md spike 4).
4. **PSU and budget spike.** For each supported bank, record `required_psu_headers`. Try an explicit refresh with a tailnet `Psu-Ip-Address`, then with only the required headers, and check whether it counts against the background cap. Run 2 background slots a day for a week, then 3–4, to learn how the bank counts calls (bank-aggregation.md spikes 1–2).
5. **Exception text.** Draft it from the Bulwark, Signal and Nextcloud models with its SPDX `AdditionRef-` name, get a lawyer's review that covers the Apple clauses in §6.3, and settle "-or-later" vs "-only".
6. **Name.** Shortlist distinctive names, search TMview, INPI, USPTO and the App Store, then file.
7. **Cross-cutting, not settled here.** An App Store build can't catch https callbacks, or use native or `WKWebView` passkeys, on self-hosters' domains: "Your app can't use passkeys to authenticate for services that you haven't configured as the app's associated domains" sits in Apple's guidance on passkeys in a web view ([Apple](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys)). A sign-in page in `ASWebAuthenticationSession` runs "in the context of the top level site" and "can use passkeys" ([passkeys.dev](https://passkeys.dev/docs/reference/ios/), secondary). So official builds keep passkeys through the instance's web sign-in plus a custom-scheme callback, as self-hosting.md and native-apps-auth.md propose. Still open there: how self-hosters get the native apps, and push to official builds without a relay.

**Not verified:**

- EB's linked-account cap, and whether EB checks certificate expiry (its sample makes a 365-day self-signed certificate).
- Whether EB lets a shared app hold another Control Panel user's linked accounts, and how it reads its automation clause (spike 1).
- Whether EB accepts a tailnet-only redirect URL, and whether the supported banks accept a tailnet `Psu-Ip-Address` (spikes 3, 4).
- The final PSR text on access frequency, re-authentication and dashboards.
- How the Nextcloud-style pledge compares legally with a §7 permission.
- Immich's reasons for shipping without an exception.
- The Firefly III docs site, which returned 403; its GitHub source was read instead. git.kresus.org and the Council's press release also blocked automated fetching: Kresus was read in its Codeberg archive and its 0.26.0 npm package, the PSR deal in Parliament's press release.

## Sources

- **Enable Banking (primary):** [Terms of Service](https://enablebanking.com/terms) · [Linked accounts](https://enablebanking.com/docs/api/linked-accounts/) · [FAQ](https://enablebanking.com/docs/faq/) · [API reference](https://enablebanking.com/docs/api/reference/) · [Quick start](https://enablebanking.com/docs/api/quick-start/) · [Control Panel](https://enablebanking.com/cp/applications) (registration-form wording read from the site bundle) · [Control Panel guide](https://enablebanking.com/docs/api/control-panel/) · [Data-sharing consents](https://enablebanking.com/data-sharing-consents/) · [Changelog, January 2026](https://enablebanking.com/blog/2026/02/17/enable-banking-changelog-january-2026) · [Changelog, July 2026](https://enablebanking.com/blog/2026/12/08/enable-banking-changelog-july-2026)
- **Regulation (primary):** [RTS 2018/389 art. 36(5)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389) · [EBA Q&A 2018_4172](https://www.eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4172) · [EBA Q&A 2018_4210](https://www.eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4210) · [PSR proposal COM(2023) 367](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52023PC0367) · [European Parliament on the PSR deal (2025-11-27)](https://www.europarl.europa.eu/news/en/press-room/20251121IPR31540/payment-services-deal-more-protection-from-online-fraud-and-hidden-fees) · [OEIL PSR 2023/0210(COD)](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0210(COD)) · [EUTMR 2017/1001](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32017R1001) · [EUPL-1.2 (Decision 2017/863)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32017D0863)
- **Actual Budget (primary):** [Bank sync](https://actualbudget.org/docs/advanced/bank-sync/) · [Enable Banking](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/) · [SimpleFIN](https://actualbudget.org/docs/advanced/bank-sync/simplefin/) · [GoCardless](https://actualbudget.org/docs/advanced/bank-sync/gocardless/) · [SECURITY.md](https://github.com/actualbudget/actual/blob/master/SECURITY.md) · [secrets-service.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/services/secrets-service.js) · [app-secrets.js](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app-secrets.js) · [app-enablebanking.ts](https://github.com/actualbudget/actual/blob/master/packages/sync-server/src/app-enablebanking/app-enablebanking.ts) · [sync.ts](https://github.com/actualbudget/actual/blob/master/packages/loot-core/src/server/accounts/sync.ts)
- **Firefly III (primary):** [Data importer releases](https://github.com/firefly-iii/data-importer/releases) · [.env.example](https://github.com/firefly-iii/data-importer/blob/main/.env.example) · [EB tutorial source](https://github.com/firefly-iii/docs/blob/main/docs/docs/tutorials/data-importer/eb.md) ([site](https://docs.firefly-iii.org/tutorials/data-importer/eb/)) · [RoutineManagerInterface](https://github.com/firefly-iii/data-importer/blob/main/app/Services/Shared/Conversion/RoutineManagerInterface.php)
- **Maybe, Sure, Ghostfolio, Kresus, woob, SimpleFIN (primary):** [Maybe README](https://github.com/maybe-finance/maybe) · [Maybe v0.6.0](https://github.com/maybe-finance/maybe/releases/tag/v0.6.0) · [Maybe registry.rb](https://github.com/maybe-finance/maybe/blob/main/app/models/provider/registry.rb) · [Sure README](https://github.com/we-promise/sure) · [Sure EB item](https://github.com/we-promise/sure/blob/main/app/models/enable_banking_item.rb) · [Sure EB adapter](https://github.com/we-promise/sure/blob/main/app/models/provider/enable_banking_adapter.rb) · [Sure ownership concern](https://github.com/we-promise/sure/blob/main/app/models/concerns/provider_item_ownable.rb) · [Sure PR #3613](https://github.com/we-promise/sure/pull/3613) · [Ghostfolio README](https://github.com/ghostfolio/ghostfolio) · [Ghostfolio self-hosting FAQ](https://github.com/ghostfolio/ghostfolio/blob/main/apps/client/src/app/pages/faq/self-hosting/self-hosting-page.html) · [Kresus repo](https://git.kresus.org/kresus/kresus) · [Kresus on npm](https://registry.npmjs.org/kresus) · [Kresus providers, Codeberg archive](https://codeberg.org/kresus/kresus/src/branch/main/server/providers/index.ts) · [Kresus FAQ](https://kresus.org/en/faq.html) · [woob pyproject](https://gitlab.com/woob/woob/-/blob/master/pyproject.toml) · [SimpleFIN protocol](https://www.simplefin.org/protocol.html)
- **Apple (primary):** [ASWebAuthenticationSession https callback](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/https(host:path:)) · [custom-scheme callback](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:)) · [callbackURLScheme initializer](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/init(url:callbackurlscheme:completionhandler:)-7fpox) · [Supporting passkeys](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys) · [Local notifications](https://developer.apple.com/documentation/usernotifications/scheduling-a-notification-locally-from-your-app) · [APNs token auth](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns) · [Media Services Terms](https://www.apple.com/legal/internet-services/itunes/us/terms.html) · [Standard EULA](https://www.apple.com/legal/internet-services/itunes/dev/stdeula/)
- **Network and other providers (primary):** [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md) · [Tailscale IP addresses](https://tailscale.com/docs/concepts/tailscale-ip-addresses) · [GoCardless rate limits](https://docs.gocardless.com/docs/bank-account-data)
- **Licenses and contribution terms (primary):** [AGPL-3.0](https://www.gnu.org/licenses/agpl-3.0.txt) · [Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0.txt) · [MPL-2.0](https://www.mozilla.org/en-US/MPL/2.0/) · [FSF license list](https://www.gnu.org/licenses/license-list.html#apache2) · [FSF on the App Store (2010)](https://www.fsf.org/news/2010-05-app-store-compliance) · [FSF details](https://www.fsf.org/blogs/licensing/more-about-the-app-store-gpl-enforcement) · [FSL](https://fsl.software/) · [GitHub ToS D.6](https://docs.github.com/en/site-policy/github-terms/github-terms-of-service#6-contributions-under-repository-license) · [DCO 1.1](https://developercertificate.org/) · [SPDX 3.0.1 license expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/) · [Google AGPL policy](https://opensource.google/documentation/reference/using/agpl-policy)
- **Peer licensing (primary):** [Nextcloud iOS README](https://github.com/nextcloud/ios) · [COPYING.iOS](https://github.com/nextcloud/ios/blob/master/COPYING.iOS) · [Mastodon iOS README](https://github.com/mastodon/mastodon-ios) · [Signal-iOS CONTRIBUTING](https://github.com/signalapp/Signal-iOS/blob/main/CONTRIBUTING.md) · [libsignal-protocol-c license](https://github.com/signalapp/libsignal-protocol-c#license) · [libsignal README](https://github.com/signalapp/libsignal) · [Bulwark Mobile LICENSE](https://github.com/bulwarkmail/native/blob/main/LICENSE) · [Bulwark consent request](https://github.com/orgs/bulwarkmail/discussions/1113) · [Element X iOS](https://github.com/element-hq/element-x-ios) · [Immich FAQ](https://github.com/immich-app/immich/blob/main/docs/docs/FAQ.mdx) · [Immich on the App Store](https://apps.apple.com/app/id1613945652) · [PikaPods apps](https://www.pikapods.com/apps) · [PikaPods](https://www.pikapods.com/)
- **Trademark (primary):** [INPI fees from 2026-07-02](https://www.inpi.fr/inpi-block/download-document?id=20516) · [EUIPO fees](https://www.euipo.europa.eu/en/trade-marks/before-applying/fees-payments) · [USPTO TSDR, ONBUDGET](https://tsdr.uspto.gov/statusview/sn85607743) · EUTMR above
- **Secondary:** Actual issues [#9443](https://github.com/actualbudget/actual/issues/9443), [#8345](https://github.com/actualbudget/actual/issues/8345), [#8846](https://github.com/actualbudget/actual/issues/8846), [#8701](https://github.com/actualbudget/actual/issues/8701) · Firefly III issues [#12553](https://github.com/firefly-iii/firefly-iii/issues/12553), [#12093](https://github.com/firefly-iii/firefly-iii/issues/12093) · [KMyMoney on FITID](https://mail.kde.org/pipermail/kmymoney-devel/2018-February/020182.html) · [CPI L712-1 reproduction](https://www.baumann-avocats.com/CPI/article-l712-1-du-code-de-la-propriete-intellectuelle.php) · ["On Budget" App Store listing](https://appgoblin.info/apps/6499044750) · [passkeys.dev iOS reference](https://passkeys.dev/docs/reference/ios/)
