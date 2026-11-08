# Enable Banking before the instance exists

> Research note, 2026-11-08, for [#14](https://github.com/fveauvy/shoestring/issues/14). Versions are the newest stable releases that day, from the npm registry, nodejs.org and GitHub releases. Confidence: **H** high, **M** medium, **L** low. "Secondary" marks a non-primary source, "judgment" my own assessment, "bundle" a fact read in the script that enablebanking.com serves for its Control Panel (read, never run), "measured" a request made for this note. EB is Enable Banking. "Sandbox" is EB's environment, never the test household. Builds on [connectors-and-license.md](connectors-and-license.md) and [bank-aggregation.md](bank-aggregation.md), and edits neither.

## Question

The engine step comes right after step 0 and the visual direction. It covers the schema, the API and sync protocol, the owner's bank connections through EB, and the three clients ([#7](https://github.com/fveauvy/shoestring/issues/7), decision 9). How can it run refreshes end to end on a laptop, before the tailnet instance exists? The note checks:

- EB's sandbox and mock banks;
- restricted mode's limits;
- which redirect URLs EB allows, localhost included;
- the key and certificate setup for development.

It then says what moves to the tailnet host: SP-12, a callback on a tailnet-only host, and SP-13, each supported bank's data and how it counts accesses.

## TL;DR

1. **Run the engine step on EB's sandbox, with the Mock ASPSP and the contract kit's fake provider. Register no production application before the tailnet host exists.**
   - Sandbox applications are free and "activated automatically", and they call the same base URL as production.
   - The Mock ASPSP serves every country and needs no bank credentials. It returns the accounts and transactions loaded into it through the Control Panel, in the API's own shapes, with transactions in pages of 10.

   **H**
2. **The callback is `http://localhost:<port>/connect/enable-banking/callback`, listed in the sandbox application.**
   - EB's own samples list `http://localhost:8080/auth_redirect`.
   - `POST /auth` refuses a URL the application doesn't list (`REDIRECT_URI_NOT_ALLOWED`).
   - EB documents no scheme rule. Peers report that production needs `https` and that the sandbox accepts `http` (secondary).

   **H** for the sandbox, **M** for production.
3. **The development server generates the RSA key and the self-signed certificate on production's code path, and the developer pastes the certificate into a sandbox application.**
   - Requests carry RS256 JWTs whose `kid` is the application ID, valid 24 hours at most.
   - An application never moves between sandbox and production, so a development key can't reach real accounts.
   - Development mode should accept only `SANDBOX` applications, and the Nix package only `PRODUCTION` ones.

   **H** on EB's rules; the split between modes is judgment.
4. **The fake provider covers what the sandbox can't show.**
   - It covers rate limits, expired consents, PSU-header errors and repeated continuation keys, and serves a consent page for the iPhone sheet.
   - The Mock ASPSP's consent page needs a signed-in Control Panel session, and sign-in is an emailed link. So a person gives sandbox consents in a desktop browser, and the nightly smoke test reuses a consent given by hand.

   **M** (bundle)
5. **Restricted mode waits for the host.**
   - It returns only accounts linked in the Control Panel through the bank's SCA, and needs a separate API consent after that.
   - It is for personal use, under an unpublished cap on linked accounts.
   - The certificate appears fixed at registration.

   **H**, **M** on the certificate.
6. **These move to the tailnet host:**
   - the production application, its key generated there and its callback on `https://app.<domain>`;
   - SP-12, except the state checks;
   - all of SP-13.

   Before then, the owner can preview SP-13's per-bank facts without a consent: consent validity and required PSU headers from EB's public bank list, field fill rates and the oldest transaction from the Control Panel's Data Insights. **M**, judgment on the split.

## 1. Sandbox and Mock ASPSP

- **Two environments, never crossed.** "An application can be registered to either PRODUCTION (aka "live") or SANDBOX (aka "simulation") environment. Applications can not be transferred from the sandbox to the production environment and vice versa." Also: "Applications registered into the sandbox environment are activated automatically" ([API reference](https://enablebanking.com/docs/api/reference/)). **H**
- **One base URL.** The reference lists only `https://api.enablebanking.com`. The quick start keeps "the Sandbox environment" and calls that URL, and `GET /application` returns the application's `environment` ([API reference](https://enablebanking.com/docs/api/reference/), [Quick start](https://enablebanking.com/docs/api/quick-start/)). The connector needs no environment switch. **H**
- **Free, and meant for development.** "Use of the API in the Sandbox Environment is available to all users for evaluation and development purposes", with "mock data and a limited number of ASPSPs' sandboxes" ([Terms](https://enablebanking.com/terms), updated 2026-01-09). "Personal usage, Sandbox and evaluation usage continue to be free of charge" ([changelog, December 2025](https://enablebanking.com/blog/2026/01/15/enable-banking-changelog-december-2025)). **H**
- **Bank sandboxes are a poor proxy.** EB offers "a limited number of ASPSPs' sandboxes" ([Sandbox](https://enablebanking.com/docs/api/sandbox/)). Its warnings:
  - they often don't simulate live behaviour;
  - they break without notice and may be disabled;
  - "filtering doesn't work in all sandboxes";
  - "In the sandbox environment, some ASPSPs do not support session validity" ([FAQ](https://enablebanking.com/docs/faq/)).

  None of the supported banks has one. **H**
- **Mock ASPSP.** "Countries: All countries. No credentials needed." Its data comes from the Control Panel, by form or from a JSON file. Its documented limits: no payment initiation, and transactions "returned in batches of 10 with more recent transaction occuring first" ([Sandbox](https://enablebanking.com/docs/api/sandbox/)).
  - The file is `{"accounts": [{"info": …, "balances": […], "transactions": […]}]}`, in the API's account, balance and `Transaction` shapes.
  - EB's synthetic sample holds one account with 611 booked transactions ([sample](https://enablebanking.com/sample-data/DK-Danske_Bank-synthetic-1.json), measured).

  **H**
- **Its consent page needs the Control Panel.** The Mock ASPSP's consent page (`/cp/mock-aspsp/auth`) is a signed-in route. It lists the signed-in user's mock accounts ("Please select the account(s) you want to grant access to"), then posts the choice with that user's ID token and redirects back (bundle).
  - Sign-in is a one-time link sent by email ([Control Panel guide](https://enablebanking.com/docs/api/control-panel/)).
  - An ephemeral `ASWebAuthenticationSession` shares no "cookies or other browsing data" with the normal browser ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/prefersephemeralwebbrowsersession)), so the iPhone sheet starts signed out.

  **M**
- **EB keeps the mock data.**
  - Terms: "If you upload data to the Mock ASPSP …, you must ensure that such data does not contain any personal data that you are not authorized to use" ([Terms](https://enablebanking.com/terms)).
  - Uploads stay "until it is removed by you" (bundle).
  - The sandbox page suggests exporting a real account through EB's demo app. Don't: fixtures stay synthetic ([factory.md](../factory.md), Agent setup).

  **H**
- **No account-information webhooks** in either environment. EB's only webhook reports payment status ([Webhooks](https://enablebanking.com/docs/api/webhooks/)). **H**

Use in the engine step (judgment):

- **The whole path runs against the Mock ASPSP,** from `POST /auth` to normalization:
  - the stored state, the callback and `POST /sessions`;
  - account choice and the backfill through continuation keys;
  - dedupe and `DELETE /sessions/{id}`;
  - renewal reattaching by `identification_hash` across two mock consents.
- **The test-household generator writes the Mock ASPSP file,** so the sandbox data matches the seed.
- **A regular household holds the sandbox connection.** `SPEC.md` keeps test households free of bank connections, so a regular household in the development seed takes it.

## 2. Restricted mode

- **Linking, then consent.** A production application starts "Inactive". "Activate by linking accounts" sends the owner through EB's page to the bank's authorization. Then "Access needs to be authorized through API … This applies even when using the same account" ([Linked accounts](https://enablebanking.com/docs/api/linked-accounts/)). **H**
- **Only linked accounts.** Others "will be stripped from the response", so an unlinked account comes back as an empty list ([FAQ](https://enablebanking.com/docs/faq/)). `NO_ACCOUNTS_ADDED` means "No allowed accounts added to the application" ([OpenAPI](https://enablebanking.com/docs/api/reference/enablebanking-api.yaml)). **H**
- **Scope.**
  - Terms: "Use of the API in the Production Environment is limited to Linked Accounts and is available solely for evaluation purposes or for the personal use of private individuals. The number of Linked Accounts per Application may be limited." No figure is published. The terms also exclude business accounts, accounts that don't belong to the user who linked them, and any commercial use ([Terms](https://enablebanking.com/terms)).
  - Applications can't go public before a contract ([FAQ](https://enablebanking.com/docs/faq/)).
  - Control Panel: "unrestricted access in the production environment is available only to organisations. Individuals … may continue using them in restricted mode, adding more linked accounts if needed" (bundle).

  **H**
- **Registration fields.** Production asks for a description, a data-protection email, and privacy and terms URLs ([API reference](https://enablebanking.com/docs/api/reference/)). Restricted activation doesn't check them ([FAQ](https://enablebanking.com/docs/faq/)). **H**
- **Edits.** Only "an active, unrestricted application registered in the production environment" waits for approval (bundle), so a new redirect URL on a restricted application applies at once. The edit form covers the name, description, URLs, redirect URLs, and widget and webhook origins, but not the certificate (bundle). A new key probably means a new application, whose accounts are linked again. **M**, until SP-13 confirms.
- **Limits that follow the bank.** "Many ASPSPs have the limit of 4 times a day" for background fetches, which EB tells apart by the absence of PSU headers ([FAQ](https://enablebanking.com/docs/faq/)). EB's own quotas "per second, per hour and per day may apply" ([Terms](https://enablebanking.com/terms)). **H**

Why not a restricted application on the laptop (judgment):

- **Data.** Real bank data would land in a development database.
- **Key and accounts.** Its key would live outside the instance ([ADR 0012](../adr/0012-bank-data-through-the-owners-provider-application.md)). The host needs its own application anyway: applications don't move, and linked accounts belong to one application.
- **Budget.** Its refreshes would count at the same banks as the instance's. At some banks, "authorising a new session will expire a previous one" ([FAQ](https://enablebanking.com/docs/faq/)).
- **No SP-13 answer.** The laptop's server sees the browser on loopback, never on a tailnet address.
- **Scheme.** Production callbacks need `https` (secondary, §3), which development mode doesn't serve.

## 3. Redirect URLs

- **A list per application.** The Control Panel asks for "Allowed redirect URLs (one per line)", and the list can be edited later (bundle). `redirect_urls` is "always required" at registration ([API reference](https://enablebanking.com/docs/api/reference/)). A `redirect_url` outside the list fails with `REDIRECT_URI_NOT_ALLOWED` ([OpenAPI](https://enablebanking.com/docs/api/reference/enablebanking-api.yaml)). **H**
- **No published scheme rule.** The reference's examples use both `http://example.com` and `https://example.com/redirect`. The Control Panel only trims each line (bundle), so any check happens on EB's servers, under an unpublished rule.
- **Localhost works in the sandbox.**
  - EB's samples set `"redirectUrl": "http://localhost:8080/auth_redirect"`, a value unchanged since 2021 in a repository updated on 2026-03-30 ([config.json](https://github.com/enablebanking/enablebanking-api-samples/blob/master/config.json)). They ask the developer to paste the URL the browser lands on, so nothing has to listen there ([JS sample](https://github.com/enablebanking/enablebanking-api-samples/blob/master/js_example/accountInformation.js)).
  - Firefly III's tutorial: "Production environment requires HTTPS, but the Sandbox environment works with HTTP … The URL does not have to be public" (secondary; [tutorial](https://github.com/firefly-iii/docs/blob/main/docs/docs/tutorials/data-importer/eb.md), updated 2026-10-26; Data Importer v2.3.5).
  - Actual's docs: "Enable Banking only sends you back to addresses that start with https://", and EB refuses its `app://` URL (secondary; [docs](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/); Actual v26.11.0).

  **H** for the sandbox; **M** that production needs `https`. Untested: whether production accepts `https://localhost`.
- **The return is a browser navigation.** The browser lands on the callback with `code` and `state`, then the server calls `POST /sessions` (the flow in the [API reference](https://enablebanking.com/docs/api/reference/)). EB never connects to the instance, so a laptop and a tailnet-only host look alike to it. **H**

For Shoestring (judgment):

- **The callback.** Development mode's instance address is `http://localhost:<port>` ([architecture.md](../architecture.md), Security model), so the callback is `http://localhost:<port>/connect/enable-banking/callback`.
- **Worktrees.** Each worktree has its own ports ([factory.md](../factory.md)), so the sandbox application lists one callback per worktree. The `GET /application` check already reports a missing one. Register exact URLs, since EB doesn't say whether it matches by prefix.
- **Production.** The production application lists `https://app.<domain>/connect/enable-banking/callback`, which SP-12 tests.

## 4. Keys and certificates

- **EB's recipe.** The steps: "Generate a private RSA key and a self-signed certificate; Upload the certificate …; Construct JWT …; Send the JWT in the Authorization header."
  - The reference's example uses `openssl genrsa -out private.key 4096` and a certificate valid 365 days.
  - The alternative is a key generated in the browser and saved to Downloads as `<application-id>.pem`.
  - The registration API's `certificate` takes "Content of the certificate or public key".

  Sources: [API reference](https://enablebanking.com/docs/api/reference/), [Control Panel guide](https://enablebanking.com/docs/api/control-panel/). **H**
- **The JWT.** The header carries `typ: JWT`, `alg: RS256` ("only RS256 is supported") and `kid: <application_id>`. The claims are `iss: enablebanking.com`, `aud: api.enablebanking.com`, `iat` and `exp`. "Maximum allowed time-to-live for token is 86400 seconds" ([API reference](https://enablebanking.com/docs/api/reference/)). **H**
- **Secrecy.** "Keep the private key in secret (e.g. don't expose it to client-side, share with anyone nor embed into mobile or other apps …)" (Control Panel, bundle). **H**
- **Libraries, checked 2026-11-08.**
  - Node 26.13.0 LTS generates the key and signs (`crypto.generateKeyPair`, `crypto.sign`). Its `X509Certificate` "provides read-only access" ([Node crypto](https://github.com/nodejs/node/blob/v26.13.0/doc/api/crypto.md)), so it can't make the certificate.
  - `@peculiar/x509` 3.1.0 makes it ([README](https://github.com/PeculiarVentures/x509)): MIT, ESM-only, `X509CertificateGenerator.createSelfSigned` on WebCrypto, and a `Reflect` polyfill required.
  - `jose` 6.2.15 signs the JWT: MIT, no dependencies. `oidc-provider` 9.13.3, already in ADR 0007's stack, depends on it (`^6.2.13`).
  - OpenSSL 4.1.0 is EB's manual route.

  **H**

For Shoestring (judgment):

- **One path.** The instance generates the key and certificate in every mode, as ADR 0012 says. Don't use the browser-generated key, which leaves a key file in Downloads.
- **One sandbox application per developer.** Each worktree has its own database, so a key generated per database would need an application per worktree.
  - Instead, a command that only development mode accepts imports the developer's sandbox key and application ID into a worktree's database, from an untracked file.
  - CI's `nightly` environment holds its own sandbox key ([factory.md](../factory.md)).
  - Sandbox keys reach only mock data.
- **Two free checks for SP-13.**
  - Register a sandbox application with a certificate valid one day, then check whether its JWTs still pass after expiry.
  - Try to replace an application's certificate.

  Production may behave differently, so SP-13 still records both.

## 5. What the sandbox doesn't show

EB documents no way to make the Mock ASPSP return rate limits, PSU-header errors or expired consents, and no consent validity for it. The contract kit's fake provider ([architecture.md](../architecture.md), Connectors) scripts these cases with EB's error codes ([OpenAPI](https://enablebanking.com/docs/api/reference/enablebanking-api.yaml), [FAQ](https://enablebanking.com/docs/faq/)):

- `ASPSP_RATE_LIMIT_EXCEEDED` (429), then 6 hours of back-off;
- `PSU_HEADER_NOT_PROVIDED` (422) and `PSU_HEADER_INVALID`;
- `EXPIRED_SESSION` in the middle of a refresh, at one consent or at every consent of one bank;
- `REDIRECT_URI_NOT_ALLOWED`, an empty account list and a repeated `continuation_key`;
- a consent page that redirects to the callback. The web redirect and the iPhone sheet's custom-scheme bounce then run in CI and on the simulator without a Control Panel sign-in.

Development mode lets the connector's base URL point at the fake provider; the Nix package keeps `https://api.enablebanking.com` (judgment).

- **Nightly smoke test.** No one can give a Mock ASPSP consent unattended. So the test checks `GET /application`, `GET /aspsps` and `POST /auth`, then refreshes on a consent given by hand, which a person renews when it expires (judgment, **M**).
- **PSU headers.** On a laptop, the browser reaches the server over loopback, so an explicit refresh sends `127.0.0.1` or `::1`. That tests the plumbing, not the bank; EB's JS sample sends `10.10.10.10` in the sandbox. Tailnet addresses are SP-13's.

## 6. What moves to the tailnet host

| | Laptop, engine step | Tailnet host |
|---|---|---|
| Provider application | Sandbox application, Mock ASPSP | Restricted production application registered from the instance; accounts linked in the Control Panel |
| Callback | `http://localhost:<port>/connect/enable-banking/callback` | `https://app.<domain>/connect/enable-banking/callback` |
| Key | Sandbox key, under a development KEK | Generated on the instance, under its KEK |
| SP-12 | Single-use, expiring state with replays refused; the web redirect and the sheet bounce against the fake provider | EB accepts the tailnet-only callback; consents complete from the iPhone and the web at each supported bank; the app regains control after the bank's app |
| SP-13 | A preview without consent (below); sandbox checks of certificate expiry and replacement | A fact sheet per bank; how each bank counts accesses; an explicit refresh and the backfill with a tailnet `Psu-Ip-Address`; a week of refresh slots; no duplicates across overlaps or renewals; the production certificate's validity and replacement |
| Fetch budget | Logic tested against the fake provider, with a fake clock | Refresh slots set per bank |

What the owner can preview before the host:

- **The public bank list.** `https://enablebanking.com/api/aspsps`, the list EB's website uses, answers without sign-in. On 2026-11-08 it held 2,760 entries (measured):
  - every entry has `maximum_consent_validity`, 180 days for 2,737 of them;
  - 381 entries list `required_psu_headers`.

  The list is undocumented: `GET /aspsps` with the production application stays the reference. **M**
- **Data Insights.** The Control Panel shows "the available data for each ASPSP, as well as what data is missing", with each field's presence ([Control Panel guide](https://enablebanking.com/docs/api/control-panel/)). For each ASPSP, PSU type and endpoint, it also returns the oldest transaction and the time to status (bundle). That previews the field mapping and the backfill depth. It needs a Control Panel account; whether it needs more is unverified. **M**
- Per-bank values from either source go to the private ops repository with SP-13 ([AGENTS.md](../../AGENTS.md)).

## Recommendation

1. **Build the engine step's bank connections against EB's sandbox:**
   - one sandbox application per developer;
   - the Mock ASPSP, loaded from the test-household generator;
   - the fake provider for everything else.

   No production application and no real bank data until the tailnet host exists.
2. **Split the modes.** Development mode accepts only `SANDBOX` applications. The Nix package accepts only `PRODUCTION` applications, with an `https` callback.
3. **Keep production's key path.** A development-only import shares a developer's sandbox key across worktrees, and CI's `nightly` environment holds its own.
4. **Keep test households unconnected.** The development seed's regular household holds the sandbox connection.
5. **Give sandbox consents by hand.** A person gives them in a desktop browser, and the nightly test reuses one.
6. **Preview SP-13 before the host.** The owner reads the public list and Data Insights, and records the values privately.
7. **On the host, register and run the spikes.** The owner registers the restricted production application from the instance, then runs SP-12 and SP-13.

Consequences for the docs PR (judgment):

- `SPEC.md`, Bank connections: the instance checks that the application is in production, or in the sandbox in development mode.
- `docs/architecture.md`, Keys and secrets: a row for sandbox keys, held in the developer's untracked file and CI's `nightly` environment.
- ADR 0012 and the provider-key rotation row: a new key probably means a new application, whose accounts are linked again, until SP-13 shows otherwise.

## Not verified

- Whether production accepts `https://localhost`, and whether it accepts the tailnet-only callback (SP-12).
- Whether EB matches redirect URLs exactly.
- The Mock ASPSP's `maximum_consent_validity` and `required_psu_headers`, and whether the sandbox enforces PSU headers or rate limits. One `GET /aspsps` call and one refresh in the engine step settle them.
- Whether EB checks certificate expiry, and whether an application's certificate can be replaced (the §4 checks).
- The cap on linked accounts per application.
- Whether every `Transaction` field, pending status included, survives a Mock ASPSP file.
- What Data Insights needs beyond a Control Panel account.

## Sources

- **Enable Banking (primary):**
  - docs: [API reference](https://enablebanking.com/docs/api/reference/), [OpenAPI](https://enablebanking.com/docs/api/reference/enablebanking-api.yaml), [Sandbox](https://enablebanking.com/docs/api/sandbox/), [Control Panel guide](https://enablebanking.com/docs/api/control-panel/), [Linked accounts](https://enablebanking.com/docs/api/linked-accounts/), [FAQ](https://enablebanking.com/docs/faq/), [Quick start](https://enablebanking.com/docs/api/quick-start/), [Webhooks](https://enablebanking.com/docs/api/webhooks/);
  - [Terms](https://enablebanking.com/terms) (2026-01-09) and the [changelog, December 2025](https://enablebanking.com/blog/2026/01/15/enable-banking-changelog-december-2025);
  - data: the [Mock ASPSP sample](https://enablebanking.com/sample-data/DK-Danske_Bank-synthetic-1.json) and the [public bank list](https://enablebanking.com/api/aspsps);
  - the Control Panel's script bundle, `app.83127064.js`, read on 2026-11-08 behind the [Control Panel](https://enablebanking.com/cp/applications);
  - [API samples](https://github.com/enablebanking/enablebanking-api-samples): [config.json](https://github.com/enablebanking/enablebanking-api-samples/blob/master/config.json), [JS](https://github.com/enablebanking/enablebanking-api-samples/blob/master/js_example/accountInformation.js), [Python](https://github.com/enablebanking/enablebanking-api-samples/blob/master/python_example/account_information.py).
- **Platforms and libraries (primary):** [Node releases](https://nodejs.org/dist/index.json), [Node 26.13.0 crypto](https://github.com/nodejs/node/blob/v26.13.0/doc/api/crypto.md), [Apple, ephemeral sessions](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/prefersephemeralwebbrowsersession), npm records for [jose](https://registry.npmjs.org/jose), [@peculiar/x509](https://registry.npmjs.org/@peculiar/x509) and [oidc-provider](https://registry.npmjs.org/oidc-provider), [@peculiar/x509 README](https://github.com/PeculiarVentures/x509), [OpenSSL 4.1.0](https://github.com/openssl/openssl/releases/tag/openssl-4.1.0).
- **Peers (secondary):** the [Firefly III EB tutorial](https://github.com/firefly-iii/docs/blob/main/docs/docs/tutorials/data-importer/eb.md) and [Data Importer v2.3.5](https://github.com/firefly-iii/data-importer/releases/tag/v2.3.5); [Actual's EB docs](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/) and [Actual v26.11.0](https://github.com/actualbudget/actual/releases/tag/v26.11.0).
- **Repository:**
  - ADRs [0003](../adr/0003-nixos-just-tailscale-infrastructure.md) and [0012](../adr/0012-bank-data-through-the-owners-provider-application.md);
  - [architecture.md](../architecture.md): Connectors, Bank ingestion, Security model, Keys and secrets, Spikes;
  - [`SPEC.md`](../../SPEC.md): Bank connections, Households;
  - [factory.md](../factory.md): Environments, Local development, CI, Build order;
  - [GLOSSARY.md](../../GLOSSARY.md), [connectors-and-license.md](connectors-and-license.md), [bank-aggregation.md](bank-aggregation.md).
