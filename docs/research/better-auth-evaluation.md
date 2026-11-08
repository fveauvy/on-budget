# better-auth for Shoestring's sign-in

> **Status (2026-11-08).** Background research for [#24](https://github.com/fveauvy/shoestring/issues/24); [#25](https://github.com/fveauvy/shoestring/issues/25) makes the decision. The ADRs, [`SPEC.md`](../../SPEC.md), [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md) win where this note disagrees. It edits none of them.

> Research note, 2026-11-08. Versions are the newest stable releases or release candidates that day, read from npm dist-tags and GitHub releases; beta, alpha, canary and snapshot tags are left out. better-auth is read at tag [`v1.7.18`](https://github.com/better-auth/better-auth/tree/v1.7.18) and oidc-provider at [`v9.13.3`](https://github.com/panva/node-oidc-provider/tree/v9.13.3), from shallow clones that were read but never installed or run. Line counts are estimates. Confidence: **H** high, **M** medium, **L** low. "Judgment" marks my own assessment.

## Question

Can better-auth carry the sign-in of [ADR 0011](../adr/0011-passkeys-on-the-instance-sign-in-page.md) in place of oidc-provider and SimpleWebAuthn ([ADR 0007](../adr/0007-typescript-server-on-node-with-hono-drizzle-and-postgresql.md)), and what would it cost? The ticket's checklist:

- **R1.** Each instance is its own OAuth 2.0 authorization server, with its instance address as issuer.
- **R2.** Passkeys are the only credential, and the relying-party ID is the exact instance host. The owner bootstraps from a one-hour link the host CLI prints, members redeem single-use 72-hour invites, and nothing sends e-mail.
- **R3.** The web app holds a `__Host-` session cookie (HttpOnly, Secure, SameSite=Strict), behind an exact `Origin` check and a required custom header.
- **R4.** The Expo iPhone app and the Electron desktop app sign in through the system browser, with PKCE and a custom-scheme or loopback callback. They hold a bearer access token of about 15 minutes and a rotating 60-day refresh token, and reusing a rotated refresh token revokes the device.
- **R5.** Step-up before bank connections, member changes, sharing and export: a biometry-bound Secure Enclave key in the apps, a passkey on the web.
- **R6.** Sessions per device, their revocation, and recovery through the owner or the host CLI.

Also: the fit with Hono 5, Drizzle 1.0 and PostgreSQL 19 behind an Effect service ([#21](https://github.com/fveauvy/shoestring/issues/21)); its tables beside Shoestring's schema and the sync engine; its Expo and Electron packages; the maturity, release cadence and advisories of each plugin it needs; and what SP-10 and SP-11 would test instead.

## TL;DR

**Recommendation: keep oidc-provider with SimpleWebAuthn, as ADRs 0007 and 0011 plan** (**M-H**). Adopt no part of better-auth.

1. **better-auth covers the protocol** (H). Version 1.7.18, with `@better-auth/oauth-provider`, `@better-auth/passkey` and its JWT plugin, has:
   - PKCE S256, required for public clients;
   - custom-scheme and loopback redirects;
   - rotating, sliding refresh tokens with replay revocation;
   - RFC 8414 metadata and a configurable issuer;
   - hooks for passkey registration without a session, which invites need.

   Hono mounts it in one line, and its Drizzle adapter accepts Drizzle 1.0 release candidates.
2. **The hard parts stay Shoestring's code** (H):
   - step-up, in the apps and on the web;
   - bootstrap, invites and recovery;
   - device listing and revocation;
   - the custom-header CSRF check;
   - both native sign-in flows.

   Its Expo and Electron packages don't fit ADRs 0005 and 0011. Both hold a better-auth session token instead of an access and refresh token pair, and the Expo one drives `expo-web-browser`.
3. **It would save an estimated 500 to 900 lines, and open new gaps** (M). It replaces the oidc-provider adapter, the Koa bridge, the WebAuthn ceremonies and the web session. In exchange:
   - 1.7.18 can't emit a `__Host-` cookie; a pull request for it is open;
   - every user row needs a unique e-mail address;
   - a reused refresh token revokes all of the member's tokens for that app, not one device;
   - clients are database rows, not configuration;
   - Shoestring would own 13 more tables, migrate them at each minor release, and disable about 20 of the 30 core routes;
   - better-auth runs its own transactions beside Shoestring's Drizzle transactions.
4. **Its track record is weaker where Shoestring would use it** (H on the counts, M on the reading):
   - better-auth's repository lists 49 advisories, 39 of them published in 2026 (4 critical, 19 high).
   - Its OAuth provider let parallel requests reuse one authorization code, and fork a refresh token family, until 1.6.11 (2026-05-12; disclosed on 2026-05-31). Those are the two properties SP-10 exists to prove.
   - Its passkey plugin enforced neither `userVerification: "required"` nor the configured origin until 1.7.16 (2026-11-03).
   - oidc-provider has no advisory in ten years and 385 versions, and is OpenID Certified.
   - better-auth's passkey plugin depends on SimpleWebAuthn 13, which never got the fixes of 14.0.2 for two medium advisories.
5. **Its upgrades cost more** (H). better-auth ships a minor release every one to five months, with breaking changes: the 1.7 guide lists 27 for identity providers alone. Its security policy supports only the latest version. oidc-provider's v9 line has run since April 2025, and its security policy names v9 as supported.
6. **Either way, SP-10 must test concurrency** (H). oidc-provider leaves single use of codes and refresh tokens to the storage adapter, which Shoestring writes. SP-10 should send parallel redemptions and parallel refreshes and expect exactly one success.

## 1. Versions on 2026-11-08

| Package | Version (date) | Note |
|---|---|---|
| `better-auth` | 1.7.18 (2026-11-05) | `latest`; MIT. `rc` points at 1.7.0-rc.6, older than `latest`; `beta` at 1.8.0-beta.6 (left out); `release-1.6` at 1.6.34 (2026-11-01) |
| `@better-auth/passkey`, `@better-auth/oauth-provider`, `@better-auth/expo`, `@better-auth/electron`, `@better-auth/core`, `@better-auth/drizzle-adapter`, `auth` (the CLI) | 1.7.18 (2026-11-05) | Released in lockstep with `better-auth`. `@better-auth/cli` stopped at 1.4.x; the CLI is now the `auth` package |
| JWT and bearer plugins | 1.7.18 | Inside `better-auth` (`better-auth/plugins/jwt`, `better-auth/plugins/bearer`) |
| `@simplewebauthn/server` under `@better-auth/passkey` | `^13.3.2`, so 13.3.3 (2026-08-26) | One major behind |
| `oidc-provider` | 9.13.3 (2026-10-29) | `latest`; `previous` points at 6.31.1. Depends on Koa 3, jose 6 and debug |
| `@simplewebauthn/server` | 14.1.0 (2026-11-06) | Needs Node 22 LTS or later since 14.0.0 |
| `@simplewebauthn/browser` | 14.0.0 (2026-09-02) | |
| `hono` | 5.0.0-rc.1 (2026-10-10) | Under `next`; `latest` is 4.13.17 |
| `@hono/node-server` | 2.1.6 (2026-11-02) | |
| `drizzle-orm` | 1.0.0-rc.5 (2026-10-21) | Under `rc`; `latest` is 0.45.4 |
| `kysely` | 0.29.8 (2026-11-01) | A dependency of `better-auth`, even with the Drizzle adapter |

No plugin the requirements need is labelled experimental, beta, alpha or not production-ready. I searched the 1.7.18 docs, each package's README and source, and the docs' sidebar badges, which mark only Agent Auth, Cloudflare and the 1.7 upgrade guide as "New" (H). The version policy therefore raises no blocker. The packages are young, though:

- `@better-auth/passkey` was split out of the core on 2025-11-05;
- `@better-auth/oauth-provider`, first published on 2025-12-22, replaced the in-core `oidcProvider` plugin, which 1.7 removed;
- `@better-auth/electron` appeared on 2026-02-07.

## 2. The requirements

### 2.1 Summary

| | better-auth 1.7.18 | Shoestring still writes | With oidc-provider and SimpleWebAuthn |
|---|---|---|---|
| **R1** issuer | **Configuration.** `oauthProvider()` takes its issuer from the JWT plugin's `jwt.issuer`, or else from the base URL including its path. A Hono route serves the RFC 8414 document at the root | The official clients' database rows (~50 to 100 lines) | **Native.** The issuer is the constructor argument. Clients come from a static `clients` list, and both discovery routes are built in |
| **R2** passkeys, links, no e-mail | **Configuration and custom code.** The plugin takes `rpID`, `origin` and `userVerification: "required"`. Bootstrap and invites go through `registration.requireSession: false` with `resolveUser`, `context` and `afterVerification`. Every user row needs a unique `email` | Link handling (~150 to 250 lines) and a placeholder address per person | **Custom.** The ceremonies on SimpleWebAuthn (~300 to 500 lines) and the same links |
| **R3** web cookie | **Partly.** HttpOnly, Secure, SameSite=Strict and a 30-day sliding session by configuration. No `__Host-` prefix. The Origin check covers only requests that reach `auth.handler`, and there is no custom-header check | A cookie-name workaround, or waiting for an open pull request; header and Origin middleware (~30 to 60 lines) | **Custom.** The cookie session (~200 to 300 lines) |
| **R4** native tokens | **Configuration.** PKCE S256, native redirect forms, a 900-second access token and a 60-day sliding refresh token. A replay revokes every refresh token of that member for that client, not one device | Both native flows: the Swift `Security` module and the Electron main process | **Native.** `rotateRefreshToken: true` revokes the grant, which is one device's sign-in. The storage adapter must make `consume` atomic |
| **R5** step-up | **Not at all.** A passkey sign-in always creates a new session, `freshAge` only measures a session's age, and reauthentication is an open feature request | Web step-up on SimpleWebAuthn directly (~100 to 150 lines) and the Secure Enclave key (~150 to 250) | **Custom,** the same code |
| **R6** devices, recovery | **Partly.** Web sessions can be listed and revoked. Native sign-ins have no device record, and revoking a web session leaves its refresh tokens alive | Device listing and revocation (~100 to 200 lines); recovery links as in R2 | **Custom,** over oidc-provider's grants (~100 to 200 lines) |

### 2.2 R1: the authorization server and its issuer

- **Issuer** (H). Both the metadata and the tokens take the issuer from the JWT plugin's `jwt.issuer`, or else from the base URL ([`metadata.ts` L45](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/metadata.ts#L45), [`authorize.ts` L210-L228](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/authorize.ts#L210-L228)). Without a custom issuer, "the issuer path is your `basePath`, such as `/api/auth`" ([docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/index.mdx)).
  - So an issuer of `https://app.<domain>` needs the JWT plugin with `jwt.issuer` set.
  - It also needs a Hono route for `/.well-known/oauth-authorization-server`, through `oauthProviderAuthServerMetadata(auth)`, because the auth handler only serves its base path.
- **Endpoints** (H). They move under the base path: `/api/auth/oauth2/authorize`, `/oauth2/token`, `/passkey/*` and so on. They replace `docs/architecture.md`'s `/oauth/*`.
- **Clients** (H).
  - Clients are rows in `oauthClient`, created through `adminCreateOAuthClient`, which needs a signed-in session.
  - `cachedTrustedClients` only keeps listed rows in memory: "To disable one or change its redirect URIs, edit its database row and restart every instance" ([clients docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/clients.mdx#L268-L285)).
  - So the official native clients are seeded by a migration or at startup.
  - `clientPrivileges` must refuse every `create`. By default, "every signed-in user may take every action".
- **oidc-provider** (H). `clients` is "an array of client metadata objects representing statically configured OAuth 2.0 and OpenID Connect clients" ([docs](https://github.com/panva/node-oidc-provider/blob/v9.13.3/docs/README.md#clients)). Both `/.well-known/oauth-authorization-server` and `/.well-known/openid-configuration` are built in ([`initialize_app.js` L180-L184](https://github.com/panva/node-oidc-provider/blob/v9.13.3/lib/helpers/initialize_app.js#L180-L184)).

### 2.3 R2: passkeys, links and no e-mail

- **The relying party** (H).
  - `rpID` defaults to the base URL's hostname.
  - The expected origins come from configuration, "so the request cannot choose its verification target" ([`utils.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/utils.ts)).
  - That has held only since 1.7.16. Before it, "the server took the expected WebAuthn origin from the request", and `userVerification: "required"` was asked for but not checked ([GHSA-gv75-j3v8-3crm](https://github.com/better-auth/better-auth/security/advisories/GHSA-gv75-j3v8-3crm)).
  - The default is `userVerification: "preferred"` ([`routes.ts` L370](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/routes.ts#L370)), so Shoestring sets `"required"`.
- **Bootstrap, invites and recovery** (H for the hooks, M for the effort). With `registration.requireSession: false`:
  - The registration options accept an opaque `context`, which `resolveUser` checks. For Shoestring, that is the bootstrap, invite or recovery token.
  - `afterVerification` can create the person and return its `userId`.
  - `createSession: true` signs the person in, inside one better-auth transaction ([docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/passkey.mdx#L108), [`routes.ts` L118-L160, L838](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/routes.ts#L118-L160)).

  These registration endpoints are then public, so `resolveUser` must refuse anything but a valid, unexpired, unused token (judgment). The tokens themselves, their hashing, single use and expiry, and the host CLI stay Shoestring's code, as in the plan.
- **E-mail** (H). The core `user` table declares `email` as `required` and `unique`, with "TODO(#9124): drop required+unique in v2" ([`get-tables.ts` L208-L216](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/core/src/db/get-tables.ts#L208-L216)). Each person would carry a placeholder address, such as one under the reserved `.invalid` domain (judgment). Password sign-in is off unless enabled, but core still registers 30 routes, about 20 of which Shoestring doesn't use:
  - `/sign-up/email`, `/request-password-reset`, `/change-email`, `/link-social`, `/update-user` and more;
  - each one disabled with `disabledPaths` and checked.

  That option has itself been bypassed with repeated slashes ([GHSA-x732-6j76-qmhm](https://github.com/better-auth/better-auth/security/advisories/GHSA-x732-6j76-qmhm), fixed in 1.4.5).
- **User handle** (H). The plugin draws a random WebAuthn user handle for each registration ([`routes.ts` L349](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/routes.ts#L349)). So passkey managers may treat a member's second passkey as a different account. A stable handle per user is an open request ([#11555](https://github.com/better-auth/better-auth/issues/11555)).
- **Plan** (H). SimpleWebAuthn's four functions, `generateRegistrationOptions`, `verifyRegistrationResponse`, `generateAuthenticationOptions` and `verifyAuthenticationResponse`, are the same ones better-auth calls. Shoestring adds the challenge store, the credential table, counter updates and the expected origin and RP ID.

### 2.4 R3: the web session cookie

- **No `__Host-` in 1.7.18** (H).
  - Every cookie name is built as `` `${secureCookiePrefix}${name}` ``, and that prefix is either `__Secure-` or nothing ([`cookies/index.ts` L76, L96-L112](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/better-auth/src/cookies/index.ts#L76)). A configured `__Host-` name therefore becomes `__Secure-__Host-…`, as [#11061](https://github.com/better-auth/better-auth/issues/11061) reported.
  - Support is open in [#10806](https://github.com/better-auth/better-auth/issues/10806) and [PR #11315](https://github.com/better-auth/better-auth/pull/11315), which adds `cookieSecurity: "host"`.
  - A workaround would turn `useSecureCookies` off, name each cookie `__Host-…` and force `secure: true` through `defaultCookieAttributes`. It was not tested here. The earlier [PR #11171](https://github.com/better-auth/better-auth/pull/11171) notes that `getSessionCookie`, the cookie cache, multi-session and the Expo and Electron packages "only understood `__Secure-` names" (judgment: fragile).
- **Attributes** (H). HttpOnly is the default. SameSite becomes Strict through `advanced.defaultCookieAttributes`. `session.expiresIn` and `updateAge` give the 30-day sliding window ([session management](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/concepts/session-management.mdx)).
- **CSRF** (H).
  - better-auth checks `Origin` against `trustedOrigins` on cookie requests that reach `auth.handler`, and prefers non-simple requests.
  - "A direct `auth.api` call skips them, even when you pass the browser's headers" ([security docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/reference/security.mdx#L74-L85)).
  - So Shoestring's own API routes still need the exact-Origin and custom-header middleware, as in the plan.

### 2.5 R4: native sign-in and tokens

- **The flow** (H).
  - PKCE is "required for every authorization code flow, and only the `S256` method is accepted"; "Public clients cannot opt out" ([tokens docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/tokens.mdx)).
  - Native clients may register a reverse-domain scheme, a custom scheme with a host and path, or an HTTP loopback URI whose port can vary ([clients docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/clients.mdx#L179)).
  - The authorize endpoint sends a browser with no session to `loginPage`. Any endpoint that then sets the session cookie, such as the passkey sign-in, resumes the authorization ([`oauth.ts` L681-L750](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/oauth.ts#L681-L750)).
  - The docs warn against `skip_consent` for a public native client, because "any app on the device can start the flow and receive the redirect" ([L273](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/clients.mdx#L273)). That matches RFC 8252 §8.6 and the plan's passkey on every authorization.
- **Lifetimes** (H). `accessTokenExpiresIn` defaults to 3,600 s and `refreshTokenExpiresIn` to 30 days. "A refresh token's lifetime restarts at each refresh", so 900 and 5,184,000 give the ADR's numbers.
- **Access tokens** (H). They are opaque unless the request names a `resource` and the JWT plugin is on. Opaque tokens are stored hashed and deleted on revocation. A JWT access token "cannot be revoked, because it is never stored".
- **Replay** (H).
  - "Reusing a rotated refresh token is treated as replay: every refresh token the user holds for that client is revoked" ([tokens docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/tokens.mdx#L136)).
  - The code deletes every `oauthRefreshToken` row matching `clientId` and `userId`, and their opaque access tokens ([`token.ts` L537-L568, L1910-L1938](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/token.ts#L537-L568)).
  - That is all of the member's sign-ins in that app, not one device.
  - Rotation has used an atomic compare-and-swap since the fixes for [GHSA-392p-2q2v-4372](https://github.com/better-auth/better-auth/security/advisories/GHSA-392p-2q2v-4372) and [GHSA-7w99-5wm4-3g79](https://github.com/better-auth/better-auth/security/advisories/GHSA-7w99-5wm4-3g79), in 1.6.11.
- **oidc-provider** (H).
  - With `rotateRefreshToken: true`, "when a consumed refresh token is encountered an error shall be returned and the whole token chain (grant) is revoked" ([docs](https://github.com/panva/node-oidc-provider/blob/v9.13.3/docs/README.md#rotaterefreshtoken), [`refresh_token.js` L107-L113](https://github.com/panva/node-oidc-provider/blob/v9.13.3/lib/actions/grants/refresh_token.js#L107-L113)). A grant is one sign-in by one client, so this revokes the device.
  - It checks `consumed`, then calls `consume()`, in two steps ([`grant_source.js` L23-L31](https://github.com/panva/node-oidc-provider/blob/v9.13.3/lib/helpers/grant_source.js#L23-L31)). The adapter's `consume` contract says nothing about concurrency ([`my_adapter.js` L208-L219](https://github.com/panva/node-oidc-provider/blob/v9.13.3/example/my_adapter.js#L208-L219)).
  - So Shoestring's PostgreSQL adapter must consume conditionally (`… WHERE consumed_at IS NULL`) and throw when no row changed. Otherwise the race better-auth fixed in 1.6.11 exists in Shoestring's adapter (judgment).

### 2.6 R5: step-up

- **better-auth** (H).
  - The passkey sign-in endpoint always creates a session and sets its cookie ([`routes.ts` L1020-L1045](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/routes.ts#L1020-L1045)).
  - "Fresh" means the session was created within `freshAge` ([`session.ts` L599-L617](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/better-auth/src/api/routes/session.ts#L599-L617)).
  - Proving identity again without a new session is an open request: "`freshAge` only measures session age. It doesn't ask the user to authenticate again" ([#12184](https://github.com/better-auth/better-auth/issues/12184)).
  - So web step-up would verify a passkey assertion against the current member's credentials with SimpleWebAuthn directly, and mark the session.
- **Apps** (H). Neither library knows the biometry-bound Secure Enclave key. Shoestring registers its public key at sign-in and verifies signatures over server challenges in both options (judgment: ~150 to 250 lines).

### 2.7 R6: devices, revocation and recovery

- **Web sessions** (H). `listSessions`, `revokeSession` and `revokeOtherSessions` are built in.
- **Native sign-ins** (H).
  - Each rotation writes a new `oauthRefreshToken` row carrying `clientId`, `userId`, `sessionId` and the first code's `authorizationCodeId`, with no device field ([`schema.ts` L306-L410](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/schema.ts#L306-L410)).
  - Its session reference is `onDelete: "set null"`, so "session can be deleted but refresh still active".
  - Listing devices and revoking one from another device or by the owner is therefore Shoestring's query over those rows, plus the sync `clients` row (judgment).
- **Recovery** (M). The owner's recovery link for a member and the host CLI's link for the owner are the R2 mechanism with another kind of token, in both options.
- **Plan** (H). oidc-provider's adapter has `revokeByGrantId`, and its `Grant` model is the device record to name and list.

## 3. Hono, Drizzle, PostgreSQL and Effect

- **Hono 5** (H).
  - better-auth: `app.all("/api/auth/*", (c) => auth.handler(c.req.raw))`, since both use web-standard requests and responses ([Hono docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/integrations/hono.mdx)).
  - oidc-provider is a Koa application. On `@hono/node-server`, a route passes `c.env.incoming` and `c.env.outgoing` to `provider.callback()` and returns `RESPONSE_ALREADY_SENT` ([README](https://github.com/honojs/node-server#accessing-nodejs-api), [`response.ts`](https://github.com/honojs/node-server/blob/main/src/utils/response.ts)). That glue is perhaps 30 to 60 lines (judgment).
- **Drizzle 1.0** (H).
  - `@better-auth/drizzle-adapter` accepts `drizzle-orm ^0.45.2 || >=1.0.0-rc.1 <2.0.0`, which covers 1.0.0-rc.5.
  - `auth generate` writes a Drizzle schema, and drizzle-kit drafts SQL from it ([Drizzle adapter docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/adapters/drizzle.mdx)). Shoestring keeps its reviewed SQL migrations (ADR 0007).
  - The generated PostgreSQL dates are `timestamp()`, where Shoestring's instants are `timestamptz`. IDs are random base62 strings unless `advanced.database.generateId` changes them. The generated schema therefore needs edits at each regeneration (judgment).
  - `better-auth` also pulls in Kysely, which it uses internally.
- **PostgreSQL 19** (M). Neither library needs a PostgreSQL feature beyond the core, nor an extension.
- **Effect** (M). Either library sits behind an Effect service, through `Effect.tryPromise` around `auth.handler` and `auth.api.*`, or around the oidc-provider callback.
  - better-auth also runs its own request context and transactions on `AsyncLocalStorage` ([`transaction.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/core/src/context/transaction.ts)). Hooks such as `afterVerification` see better-auth's adapter, not Shoestring's Drizzle transaction.
  - Redeeming an invite writes a passkey, a person, a membership, the personal space, its plan and cash account, and change-log rows under one sync ID.
  - With better-auth, those writes span two transaction owners. Shoestring would write its rows through better-auth's adapter, or order and compensate them (judgment).
  - With SimpleWebAuthn, one Drizzle transaction holds them all. oidc-provider's own writes are only grants and tokens.

## 4. Tables beside Shoestring's schema and the sync engine

better-auth would add 13 tables, plus `rateLimit` when limits are stored in the database (H):

- core: `user`, `session`, `account`, `verification`;
- passkey: `passkey`;
- JWT plugin: `jwks`;
- OAuth provider: `oauthClient`, `oauthResource`, `oauthClientResource`, `oauthRefreshToken`, `oauthAccessToken`, `oauthConsent`, `oauthClientAssertion`.

Consequences (judgment unless sourced):

- **The person.** `user` becomes Shoestring's person, through `modelName` and extra fields, or stays a one-to-one identity table. `account`, which holds passwords and social tokens, stays empty but exists.
- **Isolation.** All of them are instance-wide, with no `household_id` or `space_id`. They sit outside row-level security by household and never sync (ADR 0010), as the plan's auth tables would.
- **Sync clients.** Revoking a device must also retire its sync `clients` row. The plan can store the grant ID on the device record; better-auth would store the `authorizationCodeId`.
- **Upgrades.** Minor releases change these tables. The 1.7 guide starts its identity-provider section with "Migrate OAuth client records": new columns, a backfill, a new compound unique index and dropped columns ([guide](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/guides/1-7-upgrade-guide.mdx)). Under ADR 0018's forward-only migrations, each such upgrade is a reviewed migration.
- **Signing keys.** The JWT plugin stores its keys in `jwks`, private keys encrypted with AES-256-GCM, with optional `rotationInterval` and `gracePeriod` ([JWT docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/jwt.mdx)). They would live in PostgreSQL and in its backups, instead of the files `secrets-generate` writes.

## 5. Expo and Electron packages

- **`@better-auth/expo`** (H).
  - It signs in through `expo-web-browser`'s `openAuthSessionAsync` ([`client.ts` L426](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/expo/src/client.ts#L426)). ADR 0005 rejects that library for sign-in, because it uses the authentication-session initializer that the iOS 27.3 SDK deprecates.
  - It keeps the better-auth session cookie in `expo-secure-store`, read and written from JavaScript. ADR 0005 keeps the refresh token out of JavaScript, in a Swift module that refreshes natively.
  - Its server plugin had a critical advisory on 2026-11-03: "An unauthenticated attacker who knows a user's email address can sign in as that user" ([GHSA-4wq4-2ppv-q3hg](https://github.com/better-auth/better-auth/security/advisories/GHSA-4wq4-2ppv-q3hg)).
  - Not usable. The Swift `Security` module stays, against better-auth's standard OAuth endpoints.
- **`@better-auth/electron`** (H).
  - It opens the system browser with `shell.openExternal` and registers a custom scheme, as [#9](https://github.com/fveauvy/shoestring/issues/9) decided.
  - It then exchanges a one-time code at `/electron/token` for a better-auth session ([`routes.ts` L41-L153](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/electron/src/routes.ts#L41-L153)), not for ADR 0011's access and refresh token pair.
  - It needs `electronProxyClient` and `ensureElectronRedirect()` on the web sign-in page ([docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/integrations/electron.mdx)).
  - It stores the cookie encrypted with the synchronous `safeStorage.encryptString`, in a `conf` file ([`client.ts` L30-L62](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/electron/src/client.ts#L30-L62)).
  - Its server plugin could return "the server's internal auth context", including "the auth secret", in a normal sign-in response, until 1.7.16 ([GHSA-9p5w-h4pr-fcrc](https://github.com/better-auth/better-auth/security/advisories/GHSA-9p5w-h4pr-fcrc), high).
  - Not adopted. The main process runs RFC 8252 against the same endpoints as the iPhone app (judgment).

## 6. Maturity, cadence and advisories

### 6.1 better-auth

- **Cadence** (H).
  - Minor releases: 1.5.0 on 2026-03-01, 1.6.0 on 2026-04-06 and 1.7.0 on 2026-08-18. 1.8.0 has been in beta since 2026-10-15.
  - The 1.7 line had 18 patch releases in 79 days, and the project published 73 stable versions in 2026 across its lines.
  - Its security policy says "We only support the latest version of Better Auth. Older versions are not supported" ([`SECURITY.md`](https://github.com/better-auth/better-auth/blob/v1.7.18/SECURITY.md)), though `release-1.6` still gets patches.
- **Breaking changes in minors** (H). The 1.7 guide has 27 sections for identity providers ([guide](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/guides/1-7-upgrade-guide.mdx)). Among them:
  - "Migrate OAuth client records";
  - "Token target is locked to the login";
  - "`/oauth2/revoke` rejects valid JWT access tokens";
  - "The old `oidcProvider` plugin is removed".
- **Project** (H). 472 contributors, 1,002 open issues and pull requests, and about 30,600 stars. A security-review workstream started in June 2026 ([blog](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/blogs/security-update-june-2026.mdx)), and the maintainers now look after Auth.js too.
- **Advisories** (H for the counts).
  - The repository lists 49 advisories since December 2024: 4 critical, 24 high, 17 medium and 4 low. 39 of them were published in 2026 ([advisories](https://github.com/better-auth/better-auth/security/advisories)).
  - The 2026-11-03 release alone fixed 12 ([blog](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/blogs/security-release-2026-11-03.mdx)).
- **Advisories that a setup like Shoestring's would have met** (M, judgment). I read each advisory's "Am I affected?" section against this setup: passkeys, the OAuth provider, the rate limiter, `disabledPaths` and `clientPrivileges`, with no password, e-mail, social or organization features. Eight would have applied; five of them were published in 2026:

  | Advisory | Severity | Published | Fixed in | What |
  |---|---|---|---|---|
  | [GHSA-7w99-5wm4-3g79](https://github.com/better-auth/better-auth/security/advisories/GHSA-7w99-5wm4-3g79) | High | 2026-05-31 | 1.6.11 | Parallel requests can reuse one authorization code |
  | [GHSA-392p-2q2v-4372](https://github.com/better-auth/better-auth/security/advisories/GHSA-392p-2q2v-4372) | High | 2026-05-31 | 1.6.11 | Concurrent refreshes fork one token family into two |
  | [GHSA-p6v2-xcpg-h6xw](https://github.com/better-auth/better-auth/security/advisories/GHSA-p6v2-xcpg-h6xw) | High | 2026-05-11 | 1.4.17 | Rotating IPv6 addresses bypasses the rate limiter |
  | [GHSA-x732-6j76-qmhm](https://github.com/better-auth/better-auth/security/advisories/GHSA-x732-6j76-qmhm) | High | 2025-12-15 | 1.4.5 | Repeated slashes bypass `disabledPaths` and rate limits |
  | [GHSA-4vcf-q4xf-f48m](https://github.com/better-auth/better-auth/security/advisories/GHSA-4vcf-q4xf-f48m) | High | 2025-11-24 | 1.4.0 | Any signed-in user can delete other users' passkeys |
  | [GHSA-xr8f-h2gw-9xh6](https://github.com/better-auth/better-auth/security/advisories/GHSA-xr8f-h2gw-9xh6) | Medium | 2026-04-16 | 1.6.5 | `clientPrivileges` doesn't limit who creates OAuth clients |
  | [GHSA-9x4v-xfq5-m8x5](https://github.com/better-auth/better-auth/security/advisories/GHSA-9x4v-xfq5-m8x5) | Medium | 2025-02-05 | 1.1.16 | The error page runs attacker script from a crafted link |
  | [GHSA-gv75-j3v8-3crm](https://github.com/better-auth/better-auth/security/advisories/GHSA-gv75-j3v8-3crm) | Low | 2026-11-03 | 1.7.16 | Passkey checks enforce neither origin nor user verification |

  The older in-core `oidcProvider` plugin had three more: plain PKCE ([GHSA-9h47-pqcx-hjr4](https://github.com/better-auth/better-auth/security/advisories/GHSA-9h47-pqcx-hjr4)), `javascript:` redirect URIs ([GHSA-86j7-9j95-vpqj](https://github.com/better-auth/better-auth/security/advisories/GHSA-86j7-9j95-vpqj)) and refresh tokens without the client secret ([GHSA-pw9m-5jxm-xr6h](https://github.com/better-auth/better-auth/security/advisories/GHSA-pw9m-5jxm-xr6h)).

### 6.2 oidc-provider

- **Advisories** (H). None. Its repository has no published advisory. GitHub's advisory database, reviewed and unreviewed, has none for the package, and npm's audit service returns none for any of its 385 versions since 2016.
- **Policy** (H). Its security policy supports v9.x, with an end of life "determined prior to the release of its successor". It also documents a threat model, and puts rate limiting out of scope ([`SECURITY.md`](https://github.com/panva/node-oidc-provider/blob/v9.13.3/SECURITY.md)).
- **Certification** (H). It is OpenID Certified for Basic, Implicit, Hybrid, Config, Form Post, 3rd Party-Init, both logout profiles, FAPI 1.0, FAPI CIBA and FAPI 2.0 ([README](https://github.com/panva/node-oidc-provider/blob/v9.13.3/README.md#certification)).
- **Cadence** (H). Majors come every one to two years: v8.0.0 on 2022-12-03, v9.0.0 on 2025-04-15. It published 26 releases in 2026.
- **Experimental features** (H). The features labelled experimental are attestation-based client authentication, CIMD, external signing, OpenID4VCI and web-message response mode ([docs](https://github.com/panva/node-oidc-provider/blob/v9.13.3/docs/README.md#features)). Shoestring needs none of them.
- **Risk** (judgment). One maintainer wrote nearly all of it: 2,837 contributions out of 67 contributors.

### 6.3 SimpleWebAuthn

- **Advisories** (H). Three, all in `@simplewebauthn/server`:
  - [GHSA-6hxq-p678-4hr2](https://github.com/MasterKale/SimpleWebAuthn/security/advisories/GHSA-6hxq-p678-4hr2), low, fixed in 13.3.2;
  - [GHSA-2g3p-m8c9-hhwh](https://github.com/MasterKale/SimpleWebAuthn/security/advisories/GHSA-2g3p-m8c9-hhwh), medium (CVSS 5.4), fixed in 14.0.2;
  - [GHSA-j3h4-m3m2-7p7j](https://github.com/MasterKale/SimpleWebAuthn/security/advisories/GHSA-j3h4-m3m2-7p7j), medium (CVSS 6.3), fixed in 14.0.2.
- **The 13.x line** (M, judgment from source).
  - The two September advisories list only 14.0.1 as affected.
  - Yet 13.3.3's `validateCertificatePath` still checks the revocation of the submitted `x5c` certificates before it looks for a trust anchor ([L23-L39](https://github.com/MasterKale/SimpleWebAuthn/blob/v13.3.3/packages/server/src/helpers/validateCertificatePath.ts#L23-L39)), the order both advisories describe. 14.0.2 moved that check after the anchor.
  - `@better-auth/passkey` 1.7.18 depends on `^13.3.2`.
  - The impact for Shoestring is small: the API unit's `PrivateNetwork` blocks the outbound CRL fetch, and only a signed-in member or an invite holder can start a registration.
- **Maintenance** (H). One main maintainer: 2,458 contributions out of 22 contributors. 14.0.0 (2026-09-02) moved to Node 22 LTS and later, and the project "will more formally aim to support Node LTS releases through their Active and Maintenance windows" ([release](https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.0.0)).

## 7. Compared with the plan

### 7.1 What Shoestring writes

Line counts are estimates for the server side (judgment).

| Piece | oidc-provider with SimpleWebAuthn | better-auth |
|---|---|---|
| Protocol: authorize, token, PKCE, rotation, revocation, metadata | Library | Library |
| Storage | A PostgreSQL adapter with an atomic `consume` (150 to 250) | Library (Drizzle adapter) |
| HTTP glue | The Koa callback inside Hono (30 to 60) | One route |
| Login and consent | Interaction routes calling `interactionFinished` (150 to 250) | `loginPage` and `consentPage` with `oauthProviderClient` (50 to 100) |
| WebAuthn ceremonies | SimpleWebAuthn, a challenge store and a credential table (300 to 500) | Library |
| Web session | The cookie, sliding expiry and revocation (200 to 300) | Library, with the `__Host-` workaround |
| Clients and keys | Static `clients`; the JWKS from `secrets-generate` | Seeded rows (50 to 100); keys in `jwks` |
| Bootstrap, invites, recovery | 200 to 300 | 150 to 250, plus placeholder e-mail addresses |
| Step-up, web and apps | 250 to 400 | 250 to 400 |
| Devices: listing and revocation | 100 to 200 | 100 to 200 |
| Origin and custom-header middleware | 30 to 60 | 30 to 60 |
| Rate limits | All of them (50 to 100) | Built in for auth routes; invites and step-up by hand (30 to 60) |
| Configuration and the Effect service | 200 to 350 | 250 to 400, including `disabledPaths` |
| **Total** | **about 1,700 to 2,900** | **about 1,100 to 2,000** |

Both app sides are the same in either option.

### 7.2 Test burden

- **Black-box, the same in both options** (judgment). These tests run against the deployed configuration, so they stay whichever library carries it:
  - the full flow, PKCE and a wrong verifier;
  - replay and parallel requests;
  - foreign origins, user verification and the sibling host;
  - no sign-up, and links that work once and expire;
  - the cookie's attributes;
  - step-up and rate limits.
- **The plan adds** unit tests for the adapter's atomic `consume`, the WebAuthn ceremony code and the session cookie.
- **better-auth adds**:
  - tests that its disabled core routes stay unreachable, in path variants too;
  - a check that the `__Host-` workaround holds;
  - a check that the generated schema matches the reviewed migration;
  - a rerun of the suite at every minor release.

Its advisory record argues against trusting the library in place of the black-box suite.

### 7.3 Security track record

See §6. Two of SP-10's properties had high-severity holes in better-auth's OAuth provider until May 2026, and its passkey checks had a hole until five days before this note. oidc-provider has no published advisory. better-auth also ties its passkeys to an older SimpleWebAuthn major (H on the facts, M on the weighing).

### 7.4 Maintenance outlook

- **better-auth** (H). Shoestring would take every minor, every one to five months, with schema and configuration changes. The security policy covers only the latest version.
- **oidc-provider** (H). One major every one to two years, and v9 is the supported line.
- **Both** (judgment). The plan's risk is two single-maintainer libraries. better-auth's risk is churn and a large surface, most of which Shoestring would disable.

### 7.5 What changes in ADR 0011's token model with better-auth

1. **Replay scope.** "Reusing a rotated refresh token revokes the device" becomes "revokes the member's sign-ins in that app".
   - With separate client IDs for the iPhone and desktop apps, it reaches only the app that was replayed.
   - The ADR would say so, or Shoestring would patch the library.
2. **Keys and secrets.** In `docs/architecture.md`'s keys table:
   - the "OIDC token-signing keys" row becomes the JWT plugin's keys, in `jwks` under the better-auth secret, with built-in rotation;
   - the "Session key" row becomes that secret, rotated through its versioned `secrets` option ([security docs](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/reference/security.mdx)).
3. **Routes.** The issuer stays the instance address through `jwt.issuer`, but the endpoints move under the better-auth base path, and the metadata needs its own root route.
4. **The web cookie** is better-auth's database session. `__Host-` waits for PR #11315 or relies on the workaround.
5. **Stray sessions.** Each native sign-in also leaves a better-auth session from the ephemeral browser. The refresh token survives its deletion, so Shoestring would delete it after the code exchange (judgment). oidc-provider keeps a session for the interaction too.
6. **Unchanged.** Opaque access tokens of 15 minutes, 60-day sliding rotation, PKCE S256 and a custom-scheme callback. Both libraries also support DPoP and the device grant, so the hardening ADR and a later device flow for a second Mac stay open either way.

ADR 0007's library line and ADR 0011's last consequence would change, and so would `docs/architecture.md`'s routes, server, keys and spikes sections.

### 7.6 The Expo and Electron apps

They don't change in either option. The Swift `Security` module and the Electron main process run RFC 8252 against standard OAuth endpoints, and neither better-auth client package fits (§5).

### 7.7 SP-10 and SP-11

- **SP-10, with better-auth**, would ask: "Does better-auth's OAuth provider, as Shoestring configures it, give PKCE and rotating refresh tokens with passkeys?" It keeps today's pass criteria and adds:
  - one code redeemed in parallel, and one refresh token sent in parallel, each give exactly one success;
  - registration and sign-in enforce `userVerification: "required"` and the exact origin;
  - the disabled core routes stay unreachable, including `//` and percent-encoded forms;
  - the session cookie is `__Host-`, with Secure, `Path=/`, no `Domain` and SameSite=Strict;
  - registering a passkey without a session fails without a valid bootstrap, invite or recovery token;
  - a replay revokes exactly the scope #25 accepts;
  - the generated schema matches the reviewed migration.
- **SP-10, with the plan**, should add the same two parallel tests against Shoestring's adapter, and the user-verification, origin and sibling-host checks against its ceremony code.
- **SP-11** is the same in both options: browser-sheet passkeys, the `Security` module, refresh with the app suspended and no refresh token in JavaScript, the 5-minute lock, the biometry-bound step-up key, SQLCipher, and a failing ceremony from a sibling host. With better-auth, that last check rests on the passkey plugin's `origin` option.

## Recommendation

| Option | Gains | Costs |
|---|---|---|
| **oidc-provider 9.13.3 with SimpleWebAuthn 14.1.0** (recommended) | Certified server with no advisory history; replay revokes one device; static clients; Shoestring owns its schema, transactions and cookie; no ADR changes | Shoestring writes the adapter with an atomic `consume`, the Koa bridge, the ceremonies and the web session; one maintainer each |
| **better-auth 1.7.18** | About 500 to 900 fewer lines; one-line Hono mount; built-in rate limits, key rotation, DPoP and device grant | The gaps in §2; 13 tables to migrate at each minor; about 20 core routes to disable; breaking minors; the record in §6.1; amends ADRs 0007 and 0011 |
| **A mix** | None found | The passkey plugin needs better-auth's core, sessions and `user` table. better-auth's sessions beside oidc-provider's would mean two session systems for one sign-in page |

**Why:** better-auth is built for apps that sign people in with passwords, e-mail and social providers. Shoestring uses none of those. Its two most demanding needs, step-up and the native token model, sit outside better-auth either way. What better-auth would remove is the code that is easiest to write and test. What it would add is surface, churn and an advisory history in exactly the parts Shoestring would run (judgment).

**What would reopen the question:** a better-auth release that has all of these:

- `__Host-` cookies (PR #11315);
- an optional e-mail (#9124);
- reauthentication without a new session (#12184);
- replay revocation per grant;
- a quiet half-year of advisories for core, passkey and the OAuth provider.

The 2.0 line is the likely place.

## Open questions and spikes

- **For #25.** Confirm the plan, or accept better-auth with these ADR changes: the replay scope, keys in the database, the routes and the `__Host-` workaround.
- **For SP-10, whichever library wins.**
  - Add parallel code redemption and parallel refresh, each expecting exactly one success.
  - Specify the adapter's `consume` as a conditional update that throws when no row changed.
- **For SP-04.** If better-auth wins, check `auth generate` against Drizzle 1.0.0-rc.5 and Shoestring's conventions: `timestamptz`, the ID type and column names.

**Could not verify**

- Runtime behavior: nothing was installed or run.
- That the `__Host-` workaround works across every better-auth route and plugin.
- Whether SimpleWebAuthn 13.3.3 is exploitable in practice: the finding comes from reading its source against the advisories.
- The line counts in §7.1, which are estimates.
- Whether Caddy collapses repeated slashes before they reach the API, which GHSA-x732-6j76-qmhm depends on.

## Sources

**Repo:**

- ADRs [0005](../adr/0005-swift-where-ios-needs-it-in-a-committed-ios-project.md), [0007](../adr/0007-typescript-server-on-node-with-hono-drizzle-and-postgresql.md), [0010](../adr/0010-spaces-for-access-and-sync.md), [0011](../adr/0011-passkeys-on-the-instance-sign-in-page.md) and [0018](../adr/0018-forward-only-upgrades-with-a-client-support-window.md);
- [`docs/architecture.md`](../architecture.md): Security model, Keys and secrets, SP-10, SP-11 and SP-34;
- [`native-apps-auth.md`](native-apps-auth.md);
- tickets [#6](https://github.com/fveauvy/shoestring/issues/6), [#9](https://github.com/fveauvy/shoestring/issues/9), [#21](https://github.com/fveauvy/shoestring/issues/21), [#24](https://github.com/fveauvy/shoestring/issues/24) and [#25](https://github.com/fveauvy/shoestring/issues/25).

**better-auth docs, at `v1.7.18`:**

- OAuth provider: [overview](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/index.mdx), [tokens](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/tokens.mdx), [clients](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/clients.mdx) and [sign-in flow](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/oauth-provider/sign-in-flow.mdx);
- plugins: [passkey](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/passkey.mdx) and [JWT](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/plugins/jwt.mdx);
- integrations: [Expo](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/integrations/expo.mdx), [Electron](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/integrations/electron.mdx) and [Hono](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/integrations/hono.mdx);
- concepts: [session management](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/concepts/session-management.mdx), [rate limit](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/concepts/rate-limit.mdx), [security](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/reference/security.mdx), [options](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/reference/options.mdx) and the [Drizzle adapter](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/adapters/drizzle.mdx);
- the [1.7 upgrade guide](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/docs/guides/1-7-upgrade-guide.mdx);
- security posts: [June 2026](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/blogs/security-update-june-2026.mdx) and [2026-11-03](https://github.com/better-auth/better-auth/blob/v1.7.18/docs/content/blogs/security-release-2026-11-03.mdx);
- [`SECURITY.md`](https://github.com/better-auth/better-auth/blob/v1.7.18/SECURITY.md).

**better-auth source, at `v1.7.18`:**

- passkey: [`routes.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/routes.ts), [`types.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/types.ts), [`utils.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/src/utils.ts) and [`package.json`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/passkey/package.json);
- OAuth provider: [`token.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/token.ts), [`revoke.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/revoke.ts), [`schema.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/schema.ts), [`oauth.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/oauth.ts) and [`metadata.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/oauth-provider/src/metadata.ts);
- core: [`get-tables.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/core/src/db/get-tables.ts), [`transaction.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/core/src/context/transaction.ts), [`cookies/index.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/better-auth/src/cookies/index.ts) and [`session.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/better-auth/src/api/routes/session.ts);
- clients: [Expo `client.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/expo/src/client.ts), [Electron `routes.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/electron/src/routes.ts) and [Electron `client.ts`](https://github.com/better-auth/better-auth/blob/v1.7.18/packages/electron/src/client.ts).

**better-auth issues, pull requests and advisories:**

- issues [#9124](https://github.com/better-auth/better-auth/issues/9124), [#10806](https://github.com/better-auth/better-auth/issues/10806), [#11061](https://github.com/better-auth/better-auth/issues/11061), [#11555](https://github.com/better-auth/better-auth/issues/11555) and [#12184](https://github.com/better-auth/better-auth/issues/12184);
- pull requests [#11171](https://github.com/better-auth/better-auth/pull/11171) and [#11315](https://github.com/better-auth/better-auth/pull/11315);
- the [advisory list](https://github.com/better-auth/better-auth/security/advisories), read through `gh api repos/better-auth/better-auth/security-advisories`.

**oidc-provider, at `v9.13.3`:**

- [README](https://github.com/panva/node-oidc-provider/blob/v9.13.3/README.md), [configuration docs](https://github.com/panva/node-oidc-provider/blob/v9.13.3/docs/README.md) and [`SECURITY.md`](https://github.com/panva/node-oidc-provider/blob/v9.13.3/SECURITY.md);
- [`refresh_token.js`](https://github.com/panva/node-oidc-provider/blob/v9.13.3/lib/actions/grants/refresh_token.js), [`grant_source.js`](https://github.com/panva/node-oidc-provider/blob/v9.13.3/lib/helpers/grant_source.js), [`initialize_app.js`](https://github.com/panva/node-oidc-provider/blob/v9.13.3/lib/helpers/initialize_app.js) and [`example/my_adapter.js`](https://github.com/panva/node-oidc-provider/blob/v9.13.3/example/my_adapter.js);
- [releases](https://github.com/panva/node-oidc-provider/releases) and [advisories](https://github.com/panva/node-oidc-provider/security/advisories).

**SimpleWebAuthn:**

- releases [v14.0.0](https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.0.0), [v14.0.2](https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.0.2) and [v14.1.0](https://github.com/MasterKale/SimpleWebAuthn/releases/tag/v14.1.0);
- advisories [GHSA-2g3p-m8c9-hhwh](https://github.com/MasterKale/SimpleWebAuthn/security/advisories/GHSA-2g3p-m8c9-hhwh), [GHSA-j3h4-m3m2-7p7j](https://github.com/MasterKale/SimpleWebAuthn/security/advisories/GHSA-j3h4-m3m2-7p7j) and [GHSA-6hxq-p678-4hr2](https://github.com/MasterKale/SimpleWebAuthn/security/advisories/GHSA-6hxq-p678-4hr2);
- [`validateCertificatePath.ts` at v13.3.3](https://github.com/MasterKale/SimpleWebAuthn/blob/v13.3.3/packages/server/src/helpers/validateCertificatePath.ts) and [at v14.0.2](https://github.com/MasterKale/SimpleWebAuthn/blob/v14.0.2/packages/server/src/helpers/validateCertificatePath.ts).

**Hono:** [`@hono/node-server` README](https://github.com/honojs/node-server#readme) and [`response.ts`](https://github.com/honojs/node-server/blob/main/src/utils/response.ts).

**npm, read on 2026-11-08:**

- dist-tags and records: [`better-auth`](https://registry.npmjs.org/better-auth), [`@better-auth/passkey`](https://registry.npmjs.org/@better-auth/passkey), [`@better-auth/oauth-provider`](https://registry.npmjs.org/@better-auth/oauth-provider), [`@better-auth/expo`](https://registry.npmjs.org/@better-auth/expo), [`@better-auth/electron`](https://registry.npmjs.org/@better-auth/electron), [`@better-auth/drizzle-adapter`](https://registry.npmjs.org/@better-auth/drizzle-adapter), [`auth`](https://registry.npmjs.org/auth), [`oidc-provider`](https://registry.npmjs.org/oidc-provider), [`@simplewebauthn/server`](https://registry.npmjs.org/@simplewebauthn/server), [`@simplewebauthn/browser`](https://registry.npmjs.org/@simplewebauthn/browser), [`hono`](https://registry.npmjs.org/hono), [`@hono/node-server`](https://registry.npmjs.org/@hono/node-server), [`drizzle-orm`](https://registry.npmjs.org/drizzle-orm) and [`kysely`](https://registry.npmjs.org/kysely);
- the bulk advisory endpoint, `https://registry.npmjs.org/-/npm/v1/security/advisories/bulk`, queried with every published `oidc-provider` version;
- GitHub's [advisory database](https://github.com/advisories?query=ecosystem%3Anpm+oidc-provider), read through `gh api /advisories`.
