# Bank data access for French banks

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: Foreground refresh; device-only parsing; watched folder; §7.2's controller advice, superseded by the household exemption.

Research date: 2026-11-07, revised the same day after a fact-check and to match the current `PRODUCT.md`
(self-hosted model, private-network instances, first supported banks). Prices, limits and signup status are as
of Nov 2026 unless dated otherwise.
"Secondary" marks forums, blogs, GitHub/GitLab issues, law-firm notes and other third-party write-ups.
Confidence: **H** high, **M** medium, **L** low.

## Question

How can `on-budget` (iOS, macOS and web apps at full parity, offline included; one self-hosted API and
database per owner, holding the households the owner creates, possibly reachable only over a private network
such as Tailscale) read accounts at its first supported banks, BoursoBank, Crédit Agricole Aquitaine and
Crédit Mutuel du Sud-Ouest (CMSO, Arkéa group), including joint accounts, securely and with little manual
work, without the owner becoming a licensed AISP? Each person connects their own banks with their own
bank-data provider credentials, or imports files ([PRODUCT.md](../../PRODUCT.md)). Which providers work in
2026, what data and limits they have, what the law and GDPR require, what the manual fallback is, and which
hands-on spikes come first.

## TL;DR

1. **Each person's own accounts and joint accounts: Enable Banking, free "restricted production" mode,
   one EB application per person.** Its terms (last updated 2026-01-09) allow production use "solely for
   evaluation purposes or for the personal use of private individuals", limited to accounts the Control Panel
   user linked, and grant "only a limited, revocable right" ([EB ToS](https://enablebanking.com/terms)). It
   lists all three first supported banks: BoursoBank (as "Boursorama Banque"), Crédit Agricole d'Aquitaine
   and Crédit Mutuel Arkéa, with a `CreditMutuelSudOuest` connector
   ([EB coverage](https://enablebanking.com/open-banking-apis/)). **H** on terms and coverage, **M** on
   per-bank data quality (not tested). CA's PSD2 API may not return every payment account (§2).
2. **The free tier covers only the Control Panel user's own accounts.** The terms grant no right to access
   "account information that does not belong to the Control Panel user who associated the Linked Accounts",
   and forbid making the API "accessible to any third party" ([EB ToS](https://enablebanking.com/terms)).
   So each person needs their own EB app, and whether a member's app may run on an instance someone else
   administers needs EB's written answer (spike 5). If EB says no, the only API route is a contract with a
   licensed AISP: Enable Banking (volume-based, monthly minimum, KYB of a company, per its
   [FAQ](https://enablebanking.com/docs/faq/)) or a French aggregator, Bridge or Powens. That is out of
   proportion for one household, so the member falls back to file import. Bridge and Powens have the best
   French coverage, webhooks and categories. For savings/PEA/assurance-vie, Bridge asks for bank passwords;
   Powens uses password-based sources at CA, CM, CIC and Arkéa but a redirect-based one at BoursoBank (§3.1).
   Prices are quote-only. **H** on the restriction, **L** on prices.
3. **Avoid** GoCardless Bank Account Data: "New signups … are currently disabled"
   ([GoCardless](https://bankaccountdata.gocardless.com/new-signups-disabled)), since July 2025 per
   Actual Budget (secondary). Also avoid Lunch Flow, which resells it for EU banks: "New Platform apps are
   paused", its $34.99/yr individual plan covers one's own data, and its API returns "up to 90 days for
   GoCardless connections" (§1). Tink, Plaid (EU/UK = Custom plans only), Yapily, TrueLayer (data API
   UK-only) and Salt Edge (unlicensed partners by invitation) are sales-led B2B, and none fits a self-hosted
   household app. **H**.
4. **Reject woob/screen scraping** for anyone but, at most, an account holder's own non-payment accounts on
   their own computer. It needs stored bank passwords, breaks several times a year (2026 fixes in the
   BoursoBank, CA and Euro-Information CM modules; CA captcha since Sep 2026; the CMSO module's last
   functional change dates from March 2024), puts liability on whoever holds the password, and still stops at
   every SCA. **H**.
5. **Make file import a first-class feature in every app.** Support OFX/QIF/CSV through browser upload in the
   web app (full parity per `PRODUCT.md`), a macOS watched folder and an iOS share extension. It is the
   fallback for every bank and for any member who can't link through a provider. Put every source behind one
   connector interface so a contracted AISP can be added later. **H**.
6. **Refresh design**: at most 4 unattended accesses per account per day
   ([RTS art. 36(5)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389)), so start
   with 2 background slots until each bank's counting is measured; extra fetches only while the person who
   authorised that consent is present; full-history backfill within about an hour of each SCA (90-day window
   after that); re-consent at least every 180 days, with handling for early expiry. **H** on the rules, **M**
   on the slot count.

## 1. Provider landscape (Nov 2026)

| Provider | Licence | Can an individual / tiny app use it? | Price at household scale | Verdict |
|---|---|---|---|---|
| **Enable Banking** (FI) | AISP, FIN-FSA ([ToS](https://enablebanking.com/terms)) | Yes, free in restricted mode for own linked accounts ([docs](https://enablebanking.com/docs/api/linked-accounts/)); serving others needs a contract + KYB; "applications can't be made available to the public before a contract is signed" ([FAQ](https://enablebanking.com/docs/faq/)) | €0 for personal use, one app per person; contract is "volume based … minimum invoicing per month", figure not public ([FAQ](https://enablebanking.com/docs/faq/)) | **Primary: one app per person** |
| **GoCardless BAD** (ex-Nordigen) | AISP | No: signups disabled ([GC](https://bankaccountdata.gocardless.com/new-signups-disabled)); existing accounts keep working ([Actual](https://actualbudget.org/docs/advanced/bank-sync/gocardless/), secondary) | n/a | Avoid |
| **Powens** (FR, ex-Budget Insight) | ACPR payment institution, CIB 16948 ([Powens](https://www.powens.com/)) | Console signup gives a sandbox and asks for company name, size, country and activity area ([quick start](https://docs.powens.com/documentation/integration-guides/quick-start)); Wealth & Loans "can be activated by your Account Manager only" ([W&L](https://docs.powens.com/documentation/integration-guides/wealth-and-loans)) | Quote only (`/pricing` returns 404) | Only if a contract is ever needed |
| **Bridge** (FR, bridgeapi.io) | ACPR payment institution ([Bridge](https://www.bridgeapi.io/securite-conformite)) | Self-serve sandbox ([dashboard](https://docs.bridgeapi.io/docs/dashboard)); production for data via "the Bridge Team" ([credentials](https://docs.bridgeapi.io/docs/api-credentials)) | Quote only: `/pricing` redirects (301) to the home page and its demo booking | Only if a contract is ever needed |
| **Tink** (Visa) | AISP | Listed prices "applicable exclusively to our existing customers"; new prospects must contact sales ([Tink](https://tink.com/pricing/)) | Quote | No |
| **Salt Edge** | AISP (UK/EU) | The Pending (fake banks, 10 connections), Test (live banks, ≤100 connections, valid 90 days) and Live statuses are for licensed "Clients"; a "Partner" without a licence joins the Partnership Programme on an invitation from sales ([docs](https://docs.saltedge.com/v6/)) | Quote | No: sales-led for unlicensed apps |
| **Yapily** | Regulated subsidiaries: Yapily Connect UAB (Bank of Lithuania, LB002045) for the EU, Yapily Connect Ltd (FCA, 827001) for the UK; the parent Yapily Ltd is unregulated (site footer) | Sales form only ([Yapily](https://www.yapily.com/pricing)) | Quote | No |
| **TrueLayer** | AISP/PISP | Payments-first; "Currently, Data V3 is only supported in the UK" ([TL](https://docs.truelayer.com/docs/enable-your-users-to-connect-their-bank-account)) | Quote | No |
| **Plaid** | AISP (EU) | "For customers based in the EU or UK … only Custom plans are offered" ([Plaid billing](https://plaid.com/docs/account/billing/)) | Custom contract | No |
| **Lunch Flow** (new, 2024+) | Reseller | EU banks come via GoCardless ([docs](https://www.lunchflow.app/docs/guides/get-started/welcome)). The individual plan reads one's own data through a Personal API; the Platform API ("let your end-users pay directly") is closed: "New Platform apps are paused for now" ([Platform API](https://www.lunchflow.app/docs/api/platform-api-overview)). The API returns "up to 90 days for GoCardless connections" ([API](https://www.lunchflow.app/docs/guides/destinations/api)) | Individual plan $34.99/yr, 2 connections, $10 per extra; developer plan "Custom", waitlist ([site](https://www.lunchflow.app/)) | Avoid: no multi-user path today; rests on a closed product |
| **Oxlin / Linxo Connect** (FR, CA group) | ACPR AISP/PISP | B2B; not evaluated beyond press material ([CA press](https://presse.credit-agricole.com/assets/2022-09-28-linxo-connect-wealth-pdf-4716-9ed05.html?dl=1), secondary) | Quote | Not assessed (**L**) |

Market signal (secondary, marketing): Lunch Flow advertises "No $2K/month minimums", which suggests
aggregator contracts usually carry monthly minimums ([Lunch Flow](https://www.lunchflow.app/)).

## 2. Coverage of the three banks

### Crédit Mutuel is several institutions

- `PRODUCT.md`'s first supported CM bank is **Crédit Mutuel du Sud-Ouest, in the Arkéa group**. The other
  groups below matter to self-hosters at other CM banks.
- **Crédit Mutuel Alliance Fédérale**: 14 federations plus CIC. BFCM lists them "as at December 31, 2022",
  Crédit Mutuel Nord Europe included ([BFCM](https://www.bfcm.creditmutuel.fr/en/bfcm/cmaf-presentation.html));
  CMNE joined on 2022-01-01 ([press](https://presse.creditmutuelalliancefederale.fr/?p=5827), via search
  excerpt). Enable Banking notes a single redirect flow "for all regional banks", with Login/Password,
  Electronic Certificate or Safetrans as SCA methods and no app-to-app redirect
  ([EB France](https://enablebanking.com/docs/markets/fr/)).
- **Crédit Mutuel Arkéa** (CM de Bretagne, CM du Sud-Ouest, Arkéa Banque Privée; Fortuneo is in the group).
  It has its own API and is a separate ASPSP everywhere. EB lists it as FR15589 with connectors
  `ArkeaBanquePrivee`, `CreditMutuelBretagne` and `CreditMutuelSudOuest`
  ([EB FR15589](https://enablebanking.com/open-banking-apis/FR15589)); EB's France page documents no Arkéa
  flow.
- **CM Maine-Anjou Basse-Normandie** and **CM Océan** are separate federations outside Alliance Fédérale
  (absent from BFCM's list). They have their own entries at Enable Banking and Bridge but use the same EB
  "CreditMutuel" connector. EB's FR15489 record carries MABN's head-office address, 43 boulevard Volney,
  Laval ([EB](https://enablebanking.com/open-banking-apis/FR15489),
  [MABN](https://www.creditmutuel.fr/telechargements/cmmabn/institutionnel/depliant_identite_2026.pdf)).
- **How to tell which one a member has**: the online-banking path, e.g. `creditmutuel.fr/cmo/…` (Océan),
  `/cmmabn/…` (MABN), `/cmne/…` (Nord Europe, Alliance). Arkéa banks use their own domains (`cmso.com`,
  `cmb.fr`).

### Coverage matrix (provider listings, Nov 2026)

| Bank | Enable Banking | Powens connector (sources → account types) | Bridge |
|---|---|---|---|
| BoursoBank | "Boursorama Banque", MFI FR40618, AIS+PIS, personal+business ([EB](https://enablebanking.com/open-banking-apis/FR40618)) | id 4: PSD2 API → checking, card; "fallback" (webauth: redirect, no stored password) → savings, Livret A, LDDS, PEA, life insurance, market, PER, loans ([connectors](https://integrate.biapi.pro/2.0/connectors?expand=sources)) | "BoursoBank", consumer+business ([providers data](https://providers-data.bridgeapi.io/providers.json)) |
| CA Aquitaine | "Crédit Agricole d'Aquitaine", FR13306, connector CreditAgricoleFR; one ASPSP per regional bank ([EB](https://enablebanking.com/open-banking-apis/FR13306)) | id 6 "Crédit Agricole" (single connector): PSD2 → checking, card; "directaccess" (credentials) → all savings/wealth types | "Crédit Agricole" group: 40 active regional providers ([providers data](https://providers-data.bridgeapi.io/providers.json)) |
| CM Alliance Fédérale | "Crédit Mutuel", FR10278 (Caisse fédérale de crédit mutuel) ([EB](https://enablebanking.com/open-banking-apis/FR10278)) | id 1 "Crédit Mutuel": PSD2 → checking, card; directaccess (credentials) → savings, PEA, life insurance, loans | "Site National", Nord-Europe, Massif Central, Antilles-Guyane, BECM (business) |
| CM Maine-Anjou BN | "Crédit Mutuel", FR15489 (MABN by its Laval address) ([EB](https://enablebanking.com/open-banking-apis/FR15489)) | not separately listed (probably id 1; unverified) | "Crédit Mutuel Maine-Anjou" |
| CM Océan | "Crédit Mutuel", FR15519, legal name "Caisse fédérale du crédit mutuel Océan" ([EB](https://enablebanking.com/open-banking-apis/FR15519)) | not separately listed (unverified) | "Crédit Mutuel Océan" |
| CM Arkéa (CMB/CMSO) | FR15589, connectors ArkeaBanquePrivee / CreditMutuelBretagne / CreditMutuelSudOuest | id 14 / 19: PSD2 → card, checking; "fallback" (credentials) → savings, PEA, life insurance… | "Crédit Mutuel Arkéa" group, 7 active: CMB, CMSO and Arkéa Banque Privée, plus four business-only providers (CMB Pro, CMSO Pro, Arkéa Banking Services, Arkéa BEI) |
| CIC | FR30066 ([EB](https://enablebanking.com/open-banking-apis/FR30066)) | id 10: PSD2 + directaccess | "CIC" (2: CIC, CIC Banque Privée) |

All three first supported banks are listed by EB, Powens and Bridge. **H** (provider listings). A listing
doesn't prove completeness: Bridge uses CA's fallback platform "in particular to retrieve missing payment
accounts under PSD2 (non-compliance)" (incident of 13 Apr–26 May 2026, CA Aquitaine among the affected
regions; [incidents data](https://providers-data.bridgeapi.io/provider-incidents.json)). EB reads only the
PSD2 API, so a CA Aquitaine link may not return every payment account (**M**; spike 2). Whether each
connector returns the expected data was not tested (**M**; see spikes).

## 3. Data you actually get

### 3.1 PSD2 APIs expose payment accounts only

- For every bank above, the Powens PSD2 source ("openapi") lists only `checking` and `card` accounts.
  Livrets, PEL/CEL, PEA, assurance-vie and loans come from "directaccess" or "fallback" sources. Powens
  defines "fallback" as "the emergency gateway provided by banks" for when their PSD2 API is down, and its
  auth mechanism varies by bank ([glossary](https://docs.powens.com/documentation/ressources/glossary),
  [connectors](https://integrate.biapi.pro/2.0/connectors?expand=sources),
  [sources doc](https://docs.powens.com/documentation/integration-guides/transactions/transactions-integration-guide)). **H**.
  - CA, Euro-Information CM and CIC: "directaccess" with `credentials`; Arkéa (CMB, CMSO): "fallback" with
    `credentials`. The user types their bank password into Powens, which stores it encrypted.
  - BoursoBank: "fallback" with `webauth`, a redirect to the bank, for which Powens does "not store end user
    credentials". Its data quality and unattended refresh are untested.
- Bridge defaults to payment accounts. Its "all accounts" mode asks users to enter their bank credentials
  in Bridge Connect ([Bridge](https://docs.bridgeapi.io/docs/connect-user-journey)).
- Bridge permanent issue "CMCIC & Monabanq" (since 2025-01-01): the Euro-Information banks (CIC, CIC Banque
  Privée, BECM, CM Site National, Antilles-Guyane, Maine-Anjou, Massif Central, Nord-Europe, Océan,
  Monabanq) "require a systematic SCA to update savings accounts", so their savings can't refresh
  unattended, even through a licensed aggregator. The Arkéa banks, CMSO included, aren't listed
  ([incidents data](https://providers-data.bridgeapi.io/provider-incidents.json)). **H** (provider-reported).
- Non-payment accounts sit outside PSD2. In the joint-committee report on ratifying the PSD2 transposition
  (ordonnance 2017-1252; registered 19 April 2018), the Senate rapporteur said "aujourd'hui 80 % des comptes
  agrégés ne seraient pas des comptes de paiement" and described a "zone grise"
  ([AN report](https://questions.assemblee-nationale.fr/dyn/15/rapports/368/l15b0894_rapport-fond.pdf)). The
  figure is an eight-year-old estimate. FiDA would have covered these accounts, but the Commission plans to
  withdraw it (§7.4).

### 3.2 Transactions, balances, accounts (Enable Banking model)

- Statuses: `BOOK`, `PDNG`, `HOLD`, `SCHD`, `CNCL`, `RJCT`, `OTHR`
  ([API ref](https://enablebanking.com/docs/api/reference/)).
- Transaction fields: `entry_reference`, `transaction_amount`, `credit_debit_indicator`,
  booking/value/transaction dates, `creditor`/`debtor` (name, account, agent),
  `remittance_information[]`, `bank_transaction_code`, `merchant_category_code` (ISO 18245),
  `balance_after_transaction`, `reference_number`, `exchange_rate`, `note`. All optional except
  amount, indicator and status ([API ref](https://enablebanking.com/docs/api/reference/)).
- IDs: EB provides unique transaction IDs in `entry_reference` "for the majority of ASPSPs", but "some do not
  provide transaction entry references at all, and some provide duplicate values even though they should
  not"; EB can sometimes generate "synthetic" ones. They usually exist only for booked transactions; pending
  ones get them only if the bank keeps the same ID through booking. The value is not globally unique, but
  "for accounts with the same identification hashes, the value is immutable". `transaction_id` is not stable
  ([FAQ](https://enablebanking.com/docs/faq/)). So key on (account matched by `identification_hash`,
  `entry_reference`), with a fingerprint fallback and no hard unique constraint (Architecture below).
  Firefly III hit the cross-account collision
  ([#13115](https://github.com/firefly-iii/firefly-iii/issues/13115), secondary).
- Balances: CLBD, ITAV and XPCD are the most common. There is no historical-balance API; rebuild history
  from `balance_after_transaction` where the bank provides it ([FAQ](https://enablebanking.com/docs/faq/)).
- Accounts: `cash_account_type` is CACC, CARD, SVGS, LOAN or OTHR. EB's account IDs are session-specific
  and change at every re-authorisation. `identification_hash` is "a stable identifier for an account across
  different sessions and even different users", so it carries accounts across re-consents, and the same
  joint account linked by both partners can be merged ([FAQ](https://enablebanking.com/docs/faq/)).
- No categorisation, and EB stores nothing: on-budget is the system of record
  ([FAQ](https://enablebanking.com/docs/faq/)).
- Which of these fields BoursoBank, CA and CMSO actually fill is **not verified** (spike 1–3).

### 3.3 Deferred-debit cards (débit différé)

- Bridge: a separate `card` account holds the month's card transactions. At month end a compensating
  transaction with `operation_type: deferred_debit_card` hits the checking account; Bridge advises hiding it
  or showing it separately to avoid apparent duplicates ([Bridge](https://docs.bridgeapi.io/docs/operation-types)).
- PSD2 sources at all three banks expose `card` accounts (Powens metadata). CA's web banking has an
  "encours carte à débit différé" view ([CA tutorials](https://www.credit-agricole.fr/particulier/informations/tutos-banque-en-ligne.html)).
- Arkéa group, CMSO included: immediate and deferred card month-end bills can't be told apart when a user
  holds both (Bridge permanent issue, since 2026-09-11) ([incidents](https://providers.bridgeapi.io/incidents)).
- Model: card account = liability settled monthly; the settlement line on checking is a transfer, not
  spending. How EB represents these cards per bank: **unverified** (**M**).

### 3.4 History depth

- Amended RTS art. 10a(1)(b): SCA-free AIS access covers "the payment transactions executed in the last
  90 days" ([2022/2360](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32022R2360)).
- EB: full history is usually available only for about an hour after authorisation, then 90 days; some
  banks allow 2–3 years. Use `strategy=longest` at first link ([FAQ](https://enablebanking.com/docs/faq/)).
- Bridge: 0–36 months depending on the bank, at least 7 days right after Connect
  ([Bridge](https://docs.bridgeapi.io/docs/data-fetching)). Powens: at least 3 months
  ([Powens](https://docs.powens.com/documentation/integration-guides/transactions/introduction-to-transactions)).
  GoCardless: up to 24 months ([GC](https://docs.gocardless.com/docs/bank-account-data)).

### 3.5 Counterparty, merchant, categories

- EB: creditor/debtor names and MCC when the bank sends them; no categories.
- Bridge: `clean_description` + `provider_description`, `category_id` (categories endpoint "contact us to
  activate"), `operation_type`, `future`/`deleted` flags
  ([data fetching](https://docs.bridgeapi.io/docs/data-fetching)).
- Powens: original and cleaned wording, type, counterparty; categorisation is a separate product
  ([Powens](https://docs.powens.com/documentation/integration-guides/transactions/introduction-to-transactions)).

## 4. Consent, refresh limits, webhooks, auth

### 4.1 Re-authentication (amended PSD2 RTS)

- Delegated Regulation (EU) 2022/2360, applicable from 2023-07-25, inserted art. 10a. Banks **shall not**
  apply SCA when a PSU reads balances or the last 90 days of transactions via an AISP. SCA is required at
  first access and when more than 180 days have passed since the last SCA via that AISP; banks may still
  require it for "objectively justified and duly evidenced" fraud reasons
  ([EUR-Lex](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32022R2360)). **H**.
- EB: `valid_until` can go up to the ASPSP's `maximum_consent_validity`, which is 180 days for most banks.
  Sessions can still expire early: a new session can cancel the old one, the bank may require a yearly KYC
  survey, or certificates rotate. Handle `EXPIRED_SESSION`
  ([FAQ](https://enablebanking.com/docs/faq/)). Bridge: SCA valid 180 days, exposed as
  `authentication_expires_at` ([Bridge](https://docs.bridgeapi.io/docs/strong-customer-authentication-2025)).
  Powens: new consent every 180 days
  ([Powens](https://docs.powens.com/documentation/integration-guides/transactions/transactions-integration-guide)).
- Conflict: Actual Budget's docs say EB access "lasts for 90 days at most"
  ([Actual](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/), secondary). Read the
  real per-bank value from `GET /aspsps` (spike 1).

### 4.2 Unattended refresh

- RTS art. 36(5): an AISP may access accounts "whenever the payment service user is actively requesting such
  information" (a), and otherwise "no more than four times in a 24-hour period", unless agreed with the bank
  (b) ([EUR-Lex](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389)). Per EBA Q&A
  2018_4210, each of the four requests "may, for example, include both the account balance and associated
  payment transactions", through one or several API calls
  ([EBA](https://eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4210)). Banks don't all count that
  way (GoCardless below).
- EB: scheduled fetches must send no PSU headers. Many banks cap them at 4/day; on
  `ASPSP_RATE_LIMIT_EXCEEDED`, retry after 6 h. Fetches triggered by a user send that user's required PSU
  headers (IP, user agent…): "if person A triggers data retrieval, you should provide the headers specific to
  person A". Background caps don't apply to them. A request must carry all required PSU headers or none
  ([FAQ](https://enablebanking.com/docs/faq/)). Since January 2026, EB checks them at its API gateway and
  returns an error if "A required PSU header is missing" or "A PSU header value is in an invalid format"
  ([changelog](https://enablebanking.com/blog/2026/02/17/enable-banking-changelog-january-2026)).
- GoCardless: "Banks may impose rate limits, down to 4 API calls per day for every account. Each endpoint
  (details, balances, transactions) has it own rate limit" ([GC](https://docs.gocardless.com/docs/bank-account-data)).
  Bridge: the scheduler refreshes items 1–2×/day ([Bridge](https://docs.bridgeapi.io/docs/manage-the-lifecycle)).
  Powens: periodic auto-sync plus on-demand
  ([Powens](https://docs.powens.com/documentation/integration-guides/sca-and-connection-states)).

### 4.3 Webhooks

EB: none found for AIS; `webhook_url` exists only for payments
([API ref](https://enablebanking.com/docs/api/reference/)), so poll. Bridge: `item.refreshed`,
`item.account.updated`… ([Bridge](https://docs.bridgeapi.io/docs/webhooks-events)). Powens:
`CONNECTION_SYNCED`, `ACCOUNT_SYNCED`… ([Powens](https://docs.powens.com/documentation/integration-guides/webhooks)).
`PRODUCT.md` requires every feature to work on an instance reachable only over a private network, where
"nothing requires inbound connections from the internet". Webhooks can't arrive there, so on-budget polls
every provider.

### 4.4 Auth model and where secrets live

- **EB**: per-app RSA key with a self-signed certificate uploaded at registration. Every call carries a
  JWT (RS256, `kid` = app ID, TTL ≤ 24 h). Docs: don't "embed into mobile or other apps"; a token can reach
  every session of the app, so it can't be shared with end users
  ([API ref](https://enablebanking.com/docs/api/reference/), [FAQ](https://enablebanking.com/docs/faq/)).
  → Each person's EB private key in the server's secret store; session IDs encrypted in the DB.
- **Bridge**: `Client-Id` + `Client-Secret` headers plus a per-user bearer token
  ([Bridge](https://docs.bridgeapi.io/docs/strong-customer-authentication-2025)). **Powens**:
  client ID/secret plus a per-user token, and a temporary code to open the webview
  ([Powens](https://docs.powens.com/documentation/integration-guides/sca-and-connection-states)).
- Rule for on-budget: the apps talk only to the on-budget API. Provider credentials never ship in an app.

### 4.5 Linking UX in the native and web apps, SDKs

- Native apps: open the bank redirect in the system browser (`ASWebAuthenticationSession`), not a
  `WKWebView`: WebViews break the switch to the bank's SCA app, and the callback page must be able to bounce
  the user back to the app ([FAQ](https://enablebanking.com/docs/faq/)). Since iOS 17.4 and macOS 14.4 the
  session can match an https callback, but its host "must be a member of a domain associated with the app"
  ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/https(host:path:))),
  and Apple fetches association files through its own CDN, which can't reach a private host
  ([Apple](https://developer.apple.com/documentation/xcode/supporting-associated-domains)). A private-network
  instance (`PRODUCT.md`) with a per-self-hoster domain rules that out, so let the instance's https callback
  page hand off to a custom URL scheme that the session captures
  ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:))).
- Web app: send the tab to the bank's authorisation URL, and bring the user back on an https callback route of
  the instance, which passes the code to the API. That redirect is a navigation in the user's browser, not an
  inbound connection, so it works from any device on the instance's network. EB accepts only `https`
  redirect URLs registered for the app; Actual Budget's desktop app fails on its `app://` URL and tells users
  to link in a browser ([Actual](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/), secondary).
- CA: the user picks the regional bank, SCA happens in the "Ma Banque" app, app switch is supported.
  Euro-Information CM: no app-to-app redirect ([EB France](https://enablebanking.com/docs/markets/fr/)).
  The BoursoBank and CMSO flows aren't documented (spike 4).
- SDKs: EB has code samples (C#, Node.js, PHP, Python, Ruby) and UI widgets, but no native SDK
  ([samples](https://github.com/enablebanking/enablebanking-api-samples)). Powens has a Connect SDK and an
  iOS SDK ([iOS](https://docs.powens.com/documentation/sdk/powens-connect-ios)). Bridge uses its hosted
  Connect.

## 5. Reliability signals (2025–2026)

- **Crédit Agricole is the weak link.** Bridge logged these CA incidents in 2026
  ([incidents](https://providers.bridgeapi.io/incidents)):
  - 13 Apr–26 May: a new login on the CA fallback platform caused missing accounts. Bridge uses that fallback
    "in particular to retrieve missing payment accounts under PSD2 (non-compliance)".
  - 8–23 Jul: a failed bank roll-out caused daily errors for "several thousand users".
  - 20 Aug–9 Sep: persistent error for a minority of users.
  - 23–24 Aug: token renewal failed entirely ("bank totally down").
  - 10 Sep: AISP/PISP disruption, 05:30–11:35.
  - 14–18 Sep: missing accounts.
  - 29 Sep–7 Oct: iOS app-to-app SCA broken in Ma Banque 55.0.0.

  Powens' connector metadata flagged CA as `unstable` (demo-domain snapshot dated 2026-07-29)
  ([connectors](https://integrate.biapi.pro/2.0/connectors?expand=sources)). **H** that CA will have
  multi-day gaps.
- **Crédit Mutuel (Euro-Information banks)**: PSD2 source `stable`, credential source `unstable` (Powens
  snapshot). Daily SCA for savings (Bridge, permanent; not Arkéa). With EB, linking works once the accounts
  are whitelisted in the Control Panel ([Actual #8788](https://github.com/actualbudget/actual/issues/8788),
  secondary).
- **BoursoBank**: Powens `stable`; the only 2026 Bridge incident was on Bridge's side (certificate).
- **Arkéa (CMSO)**: Powens PSD2 and credential sources `stable` (same snapshot). Bridge: IBANs and balances
  missing from the PSD2 API on 2026-11-02 (fixed the same day); deferred-card ambiguity (open since
  2026-09-11).
- EB-specific public reports for these banks are sparse. Actual Budget's EB support is still
  "experimental" ([Actual](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/), secondary).
- Implication: show per-account "last successful sync" and health; never fail silently; keep import as
  the escape hatch.

## 6. Screen scraping: woob / Kresus

- **What**: woob modules `boursorama`, `cragr` (Crédit Agricole), `creditmutuel` (also CIC) and `cmso`
  (Arkéa) ([woob](https://gitlab.com/woob/woob/-/tree/master/modules)). Kresus, a self-hosted French PFM,
  drives woob to log in with the user's password "every night"; in standalone mode "no data will be
  encrypted … not even your banking password" ([Kresus FAQ](https://kresus.org/en/faq.html)).
- **Breakage** (GitLab, secondary):
  - `cragr`: new "CA-Connect login and JSON API" (2026-06-26); `clientId` broke after a site change
    (2026-09-06); "friendly captcha" detection now raises ActionNeeded (2026-09-02)
    ([commits](https://gitlab.com/woob/woob/-/commits/master/modules/cragr)).
  - `creditmutuel`: FormNotFound "since 2026/03/17"; "unable to sync accounts since 2026-06-25"; fixes in
    Oct–Nov 2026 ([commits](https://gitlab.com/woob/woob/-/commits/master/modules/creditmutuel)).
  - `boursorama`: open issues from Apr 2026; fixes in Oct 2026
    ([commits](https://gitlab.com/woob/woob/-/commits/master/modules/boursorama)).
  - `cmso`: last functional commit 2024-03-17, then only a release bump and formatting, so its 2026 state is
    unknown ([commits](https://gitlab.com/woob/woob/-/commits/master/modules/cmso)).

  For the three modules with 2026 activity, that is roughly one breaking change per bank per quarter. **M**.
- **SCA**: direct web access also needs SCA at least every 180 days (amended art. 10, an option for banks)
  and on any risk signal ([2022/2360](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32022R2360)).
  Unattended scraping stops until a human enters the OTP or approves in the app; a 2024 issue on CM mobile
  2FA is still open ([woob issue search](https://gitlab.com/woob/woob/-/issues/?search=creditmutuel)).
- **Legal (France)**:
  - *A user scraping their own accounts* is not a payment service. The exposure is contractual and about
    liability. Bank terms make the customer answer for disclosure or use of their credentials "à / par des
    tiers"; e.g. CM Océan's remote-banking terms (09/2024) exempt only licensed AIS/PIS providers
    ([CM T&Cs](https://www.creditmutuel.fr/telechargements/cmo/informations-reglementaire/conditions-generales/2024_conditions_generales_banque_a_distance.pdf)).
    CMF L133-16 requires the user to take "toute mesure raisonnable pour préserver la sécurité de ses données
    de sécurité personnalisées", and under L133-19 IV the payer bears all losses after failing that
    obligation "intentionnellement ou par négligence grave" (French text from
    [codes.droit.org](https://codes.droit.org/PDF/Code%20mon%C3%A9taire%20et%20financier.pdf), a Légifrance
    export updated 2026-09-01; Légifrance blocks automated fetching; translations:
    [L133-16](https://french-business-law.com/rench-legislation-art/article-l133-16-of-the-french-monetary-and-financial-code),
    [L133-19](https://french-business-law.com/rench-legislation-art/article-l133-19-of-the-french-monetary-and-financial-code),
    secondary).
  - *A third party scraping someone else's accounts* (e.g. an instance holding a member's bank password):
    that person hands their password to an unlicensed party, outside the RTS rules on TPP identification and
    secure communication (arts. 30–34)
    ([RTS](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389)). Done for people outside
    the household, it may also be AIS without authorisation (§7.1).
- **Security**: a stored password gives full online-banking access, and storage must be reversible, so the
  server becomes a high-value target. PSD2 tokens are read-only, scoped, revocable and expire within
  180 days.
- **Verdict: reject** for v1 and for anyone's accounts but the holder's own. At most, a holder could run woob
  locally on their own computer for their own livrets/PEA/assurance-vie. A monthly manual balance snapshot is
  simpler and safer.

## 7. Regulation and GDPR

### 7.1 PSD2/DSP2 position of on-budget

- **Personal use**: EB is the licensed AISP; each person is both the PSU and the owner of their own EB app,
  and EB's terms allow this ([ToS](https://enablebanking.com/terms)). No licence is needed. **H** for the owner
  on their own instance; **M** for a member whose app runs on the owner's instance (spike 5).
- **People outside the household**: `PRODUCT.md` rules out running a service for others. Self-hosters deploy
  their own instance and register their own EB app, so the project itself provides no AIS. The question only
  arises if an owner served people outside their household. Giving others "consolidated information on one
  or more payment accounts" is AIS (PSD2 art. 4(16), Annex I(8),
  [PSD2](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32015L2366)). CMF L521-2: "Il est interdit
  à toute personne autre que celles mentionnées à l'article L. 521-1 de fournir des services de paiement au
  sens du II de l'article L. 314-1 à titre de profession habituelle"; L572-5 punishes a breach with three
  years' imprisonment and a €375,000 fine (French text from
  [codes.droit.org](https://codes.droit.org/PDF/Code%20mon%C3%A9taire%20et%20financier.pdf), a Légifrance
  export; check it on Légifrance by hand, which blocks automated fetching;
  [translation](https://french-business-law.com/rench-legislation-art/article-l521-2-of-the-french-monetary-and-financial-code),
  secondary). A non-commercial tool for a few friends is arguably outside "profession habituelle", but no
  ACPR position was found, so don't rely on it (**L**). Safe structure: a licensed AISP that contracts with
  each end user (its end-user terms plus your app's ToS/privacy policy, which EB requires and monitors in
  unrestricted production; [FAQ](https://enablebanking.com/docs/faq/)).
- **Registration instead of a licence**: PSD2 art. 33 exempts "natural or legal persons providing only" AIS
  from the authorisation procedure; they register instead and must hold professional indemnity insurance
  (art. 5(3)) ([PSD2](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32015L2366)). France
  implements this as the PSIC status (prestataire de services d'information sur les comptes, CMF L522-1 II,
  natural or legal persons), registered with the ACPR under L522-11-2. The ACPR lists no minimum capital and
  no own-funds requirement for a PSIC, but professional liability insurance (ARCP) and internal control
  ([ACPR, Forum Fintech 2021](https://acpr.banque-france.fr/system/files/2024-11/forum_fintech_2021_atelier_services_de_paiement_procedures_reglement.pdf)).
  It is the real alternative to a licence, and still likely too heavy for a household project (**M**).
- **Agent status** (PSD2 art. 19 registration) matters only if on-budget handles bank credentials or
  builds custom flows. Powens: custom flows that transmit credentials "require the Agent Status"
  ([Powens](https://docs.powens.com/documentation/integration-guides/sca-and-connection-states)). Using
  hosted flows avoids it.

### 7.2 GDPR for household members' data

- The household exemption (art. 2(2)(c)) covers "purely personal or household activity". Recital 18 adds
  that the GDPR "applies to controllers or processors which provide the means for processing personal data
  for such personal or household activities"
  ([GDPR](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679)). → An owner who held
  data of people outside their household would be its **controller** (not a `PRODUCT.md` use case). Treat
  household members the same way, since the deployment is shared. **M–H**.
- Special categories by inference: data that indirectly reveals e.g. political opinions, trade-union
  membership or sexual orientation falls under art. 9
  ([CJEU C-184/20](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:62020CJ0184)). The
  prohibition "is independent of … whether the controller is acting with the aim of obtaining information
  that falls within one of the special categories"
  ([CJEU C-252/21, para. 69](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:62021CJ0252)).
  Stored transactions that reveal union dues, donations or pharmacy purchases can therefore be special-category
  data even if on-budget derives no categories. Plan explicit consent (art. 9(2)(a)) when a person links an
  account or imports a file, keep only what the budget needs (e.g. a retention limit on raw payloads), and
  still don't derive such categories.
- Minimum kit:
  - privacy notice (art. 13) and lawful basis (contract or consent);
  - record of processing (the art. 30(5) exemption doesn't apply to regular processing);
  - art. 28 contracts with the host;
  - EU hosting, as a design choice (`PRODUCT.md` puts the maintainer's instance on an EU VPS), not a GDPR
    rule: Chapter V allows transfers under an adequacy decision (art. 45) or appropriate safeguards such as
    standard contractual clauses (art. 46);
  - encryption at rest, per-user access control, audit log (art. 32);
  - deletion and export on request;
  - breach notification to CNIL within 72 h (art. 33);
  - a short DPIA-style risk note (whether one is mandatory was not verified).
- Counterparty names inside transactions are third-party data. Keep them minimal and don't share them.

### 7.3 PSR / PSD3 timeline

- Provisional agreement on 2025-11-27
  ([Hogan Lovells](https://www.hoganlovells.com/en/publications/final-texts-for-psd3-and-psr-awaited-as-european-parliament-and-council-of-eu-announce-provisional),
  secondary). Final compromise texts published in April 2026
  ([Arthur Cox](https://www.arthurcox.com/insights/psd3-and-psr-final-compromise-texts-published/), secondary).
- Since then ([OEIL PSR](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0210(COD)),
  [OEIL PSD3](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0209(COD))):
  - 2026-05-05: ECON committee approved the text.
  - 2026-11-05: Council first-reading positions published (08543/1/26 PSR, 08535/1/26 PSD3).
  - 2026-12-14: indicative Parliament second-reading plenary. **H**.
- Then publication in the Official Journal (likely early 2027), entry into force 20 days later. The PSR
  applies 21 months after that (verification of payee at 27 months), and PSD3 must be transposed within
  21 months ([Arthur Cox](https://www.arthurcox.com/insights/psd3-and-psr-final-compromise-texts-published/)).
  → Around late 2028.
- Open-banking content: a mandatory dedicated interface and user dashboards to monitor or withdraw access
  (ibid.). Nothing changes for on-budget before about 2029, and no personal-use carve-out was found.
  Re-authentication details in the final text were not verified.

### 7.4 FiDA (open finance: savings, investments, insurance)

- OEIL status: "Awaiting Parliament's position in 1st reading"; not law
  ([OEIL](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0205(COD))). **H**.
- The Commission's adopted 2027 work programme (COM(2026) 710 final, 4 Nov 2026) lists FiDA (COM(2023) 360,
  2023/0205(COD)) in Annex III, the "pending legislative proposals, which the Commission intends to withdraw
  within six months": "No trilogues have taken place since June 2025 and the file is unlikely to secure
  support in the Council within a reasonable timeframe"
  ([Annex III](https://commission.europa.eu/document/download/9ad0afe2-f9a8-46e9-bd3b-74c2de496c48_en?filename=COM_2026_710_1_annexe_EN.pdf)).
  **H**.
- → Don't plan on regulated API access to livrets/PEA/assurance-vie.

## 8. Manual import fallback

| Bank | Formats offered | Notes / quirks to test |
|---|---|---|
| BoursoBank | PDF statements; **CSV** from search results ("Exporter en Format CSV"); **OFX and QIF** from account movements ([BoursoBank FAQ](https://www.boursobank.com/aide-en-ligne/banque/releves-et-documents-bancaires/question/comment-et-sous-quels-formats-puis-je-exporter-mes-mouvements-de-compte-et-de-carte-bancaire-26780)) | CSV covers exactly the searched period; card movements are exportable too |
| CA Aquitaine | Official tutorials "Télécharger l'historique de vos opérations" and "Personnaliser le format de téléchargement" (videos; formats not stated) ([CA](https://www.credit-agricole.fr/particulier/informations/tutos-banque-en-ligne.html)) | A 2019 help PDF from another regional bank, CA Charente-Périgord (asset path `cr824`; Aquitaine's is `cr833`), reported at the new portal's launch that OFX downloads were "no longer compatible" with Money/Quicken ("n'est plus compatible") ([PDF](https://www.credit-agricole.fr/content/dam/assetsca/cr824/npc/documents/tutoriels/aide-nouveau-portail-client.PDF)). It only shows that CA offered OFX in 2019. Current formats: **unverified** (spike 7) |
| CMSO (Arkéa) | No official format list found | **Unverified** (spike 7) |
| Crédit Mutuel / CIC (Euro-Information) | OFX, CSV, XLS, QIF + a proprietary "CMI" format, per a forum post of 2004-03-01 ([forum](https://www.generation-nt.com/reponses/money-credit-lyonnais-c-est-fini-entraide-2264791.html), secondary) | Treat as unknown until tested. CFONB-type file transfer (TWEB) is an association service, not personal ([CM](https://www.creditmutuel.fr/fr/associations/comptes/tweb.html)) |

- Generic quirks to handle (check each real file): `;` separator, decimal comma, `dd/mm/yyyy` dates,
  Windows-1252 vs UTF-8, one signed amount vs debit/credit columns, OFX 1.x SGML (not XML) headers,
  `FITID` stability across exports, QIF date ambiguity, overlapping exports causing duplicates. CFONB 120
  is a fixed-width professional statement format, not relevant for personal accounts.
- **Semi-automatic import is realistic**, with a path in every app at parity (`PRODUCT.md`):
  - web app: browser file upload (file picker and drag-and-drop), parsed in the client so import also works
    offline; an installed web app can also register as a share target, in Chromium browsers only, not Safari
    or Firefox ([MDN](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest/Reference/share_target));
  - macOS: a watched folder (FSEvents on `~/Downloads` with filename rules);
  - iOS: a share extension plus "Open in…" document types for `.ofx/.qif/.csv`.

  Dedupe by FITID, else a fingerprint (date, amount, normalised label, running balance). Logging in and
  passing SCA at the bank stays manual, so this is a weekly habit, not automation.

## Provider × bank comparison

| | BoursoBank | CA Aquitaine | CM Alliance / MABN / Océan | CM Arkéa (CMSO) | Savings/PEA/AV | Webhooks | Consent | Individual use 2026 |
|---|---|---|---|---|---|---|---|---|
| **Enable Banking** | ✅ FR40618 | ✅ FR13306 (PSD2 may miss accounts) | ✅ FR10278 / FR15489 / FR15519 | ✅ FR15589 | ❌ PSD2 only | ❌ (poll) | ≤ bank max, mostly 180 d | ✅ free for own accounts, one app per person; contract for others |
| **Powens** | ✅ API + fallback (redirect) | ✅ API + credentials (unstable) | ✅ "Crédit Mutuel" API + credentials | ✅ API + fallback (credentials) | ✅ Wealth & Loans (credentials; BoursoBank by redirect) | ✅ | 180 d | ❌ contract |
| **Bridge** | ✅ | ✅ (40 regional) | ✅ incl. MABN, Océan | ✅ (7 providers) | ✅ "all accounts" (credentials); Euro-Information savings need SCA each refresh | ✅ | 180 d | ❌ contract |
| **GoCardless BAD** | not checked | not checked | not checked | — | ❌ | ❌ | per its docs "up to 90 days" | ❌ signups closed |
| **Tink / Salt Edge / Yapily / TrueLayer / Plaid** | not checked per bank | — | — | — | varies | varies | 180 d (RTS) | ❌ sales-led |
| **Lunch Flow** | via GoCardless | via GoCardless | via GoCardless | — | ❌ | n/a | GC; API history ≤ 90 d | Own data only ($34.99/yr); Platform API paused |
| **woob (scraping)** | ✅ module | ✅ module (captcha) | ✅ module | ✅ `cmso` (no fix since 2024) | ✅ | ❌ | until next SCA | ✅ technically; ❌ recommended |
| **File import** | CSV/OFX/QIF | unverified | unknown (2004 forum: OFX/CSV/QIF) | unverified | manual | — | — | ✅ every app, web upload included |

## Recommendation for on-budget

### Primary and fallback per bank

| Accounts | Primary (v1) | Fallback | Later |
|---|---|---|---|
| BoursoBank | The account holder's own EB restricted app | OFX/QIF/CSV via web upload, share sheet or watched folder | same |
| CA Aquitaine | The holder's own EB restricted app; check that every payment account appears (spike 2) | import (formats unverified); expect outages | same |
| CMSO (Arkéa) | The holder's own EB restricted app | import (formats unverified) | same |
| Joint accounts | Link under either co-holder's EB app (the account also "belongs to" them; **M**: confirm with EB) | import | merge both partners' links via `identification_hash` |
| A member's accounts on an instance someone else administers | The member's own EB app on that instance, once EB confirms in writing (spike 5). Until then: file import | import | contract (EB or Bridge/Powens) only if EB refuses |
| People outside the household | Not a `PRODUCT.md` use case: they self-host with their own EB app | — | — |
| Livrets / PEA / assurance-vie | Manual monthly balance snapshot (or import) | — | Powens Wealth & Loans / Bridge "all accounts" if ever contracted (credentials, except Powens' redirect-based BoursoBank source) |

**Trade-offs**

- EB restricted mode: free, licensed and permitted for personal use, one app per person. EB has no categories
  and no webhooks (which a private-network instance couldn't receive anyway, §4.3), and data quality depends
  on each bank's PSD2 API. The free tier can go away: the terms grant "only a limited, revocable right", EB
  "may modify or discontinue any functionality at any time", "The number of Linked Accounts per Application
  may be limited", and EB's liability is capped at EUR 100 ([EB ToS](https://enablebanking.com/terms)). Keep
  import working, so that losing EB costs latency, not data.
- A contract (EB, Bridge or Powens) is needed only if EB refuses per-person apps on a shared instance, and
  would bring categories at Bridge/Powens (their webhooks need a public endpoint). It likely needs a legal
  entity, published ToS/privacy, KYB and a monthly minimum (unknown). Bridge and Powens also cover wealth
  accounts, mostly through password-based connectors; Powens reaches BoursoBank's through a redirect-based
  fallback (untested).
- File import: zero cost and zero compliance surface, but SCA at the bank every time and latency of days.
- Lunch Flow is not a usable multi-user path today. Its Platform API ("let your end-users pay directly") is
  paused for new apps, and its $34.99/yr individual plan is for one's own data. Each person could buy that
  plan and give on-budget a Personal API key, but EU data then come from GoCardless's closed product, with at
  most 90 days of history through the API, for a fee where EB is free. Avoid it (TL;DR #3).

### Refresh cadence design

- **Background**: start with 2 slots/day per consent (e.g. 07:00 and 19:00 Europe/Paris, ± random jitter).
  Each slot fetches balances, then transactions with `date_from = last_booked_date − 7 d`. That is 2 calls a
  slot, so 2 slots stay within 4 calls even at a bank that counts every call against one 4/day cap. Raise a
  bank to 3–4 slots only after measuring that it counts a slot as one access (EBA Q&A 2018_4210) or each
  endpoint separately (as GoCardless reports). On `ASPSP_RATE_LIMIT_EXCEEDED`, back off at least 6 h.
- **User present**: when the app comes to the foreground and the last sync is older than 15 min, refresh
  through the API with the PSU headers of the person present (client IP and user agent forwarded by the
  backend, in a valid format: EB rejects malformed values), and only for consents that person authorised. On
  an instance reachable only over Tailscale, the backend sees the device's tailnet address (100.64.0.0/10,
  [Tailscale](https://tailscale.com/docs/concepts/tailscale-ip-addresses)), not its public IP; whether banks
  accept that as `Psu-Ip-Address` is unknown (spike 1). A partner opening a shared budget that includes the
  owner's personal accounts is not the PSU of those consents (RTS art. 36(5)(a); EB's "person A" rule), so
  refresh them within the background cap. Never send PSU headers from cron.
- **On every new consent**: run `strategy=longest` immediately (full history is available for about 1 h)
  to backfill anything older than 90 days.
- **Consent lifecycle**: store `valid_until`; notify at T−14 d, T−3 d and T−0; one-tap re-link; treat
  `EXPIRED_SESSION` as "needs re-auth" at any time.
- **Pending transactions**: store PDNG separately, replace them when the booked twin appears (EB pending
  IDs are often unstable), and exclude them from recurring-transaction detection.
- **Health**: per-account state (ok / degraded / needs re-auth / bank down) and last success time. Run
  alerts and forecasts after each successful sync.
- **Architecture**: put EB, file import and a future Bridge/Powens behind one `BankConnector` interface
  (`link`, `refresh`, `accounts`, `transactions(since)`, `consentExpiry`). Map provider accounts to on-budget
  accounts by `identification_hash`, never by EB's account ID, which changes at every re-consent. Key
  transactions on (on-budget account, `entry_reference`) without a hard unique constraint, and fall back to a
  fingerprint (§8) when the reference is missing or duplicated. Keep raw payloads, encrypted and with a
  retention limit (§7.2), for re-processing. Private keys live only on the server.

## Open questions and spikes

**Spikes (in order)**

1. **EB + BoursoBank (½ day)**:
   - Register a production app, then "Activate by linking accounts" with the current account and the card.
   - Authorise through the API (a separate step even for the same account,
     [docs](https://enablebanking.com/docs/api/linked-accounts/)).
   - Record from `GET /aspsps`: `maximum_consent_validity` and `required_psu_headers`; check that a
     user-present fetch with a tailnet `Psu-Ip-Address` succeeds.
   - Record from the data: history depth with `strategy=longest`, balance types, PDNG presence,
     `entry_reference` presence, stability and duplicates, creditor/debtor/MCC fill rate, and how the card
     account looks.
2. **EB + CA Aquitaine**: the same, plus:
   - account completeness: compare the accounts EB returns with those in CA's web banking (Bridge reports
     PSD2 non-compliance on payment accounts, §2);
   - deferred-debit card representation, and the app-switch flow with Ma Banque on iOS;
   - a week at 2 background slots/day, then 3–4, to learn how CA counts balance and transaction calls.
3. **EB + CMSO (Arkéa)**: the same as spike 1, plus the linking flow (EB documents none for Arkéa), IBAN and
   balance presence (missing at Bridge on 2026-11-02) and the deferred-card ambiguity (§3.3). At whichever
   supported bank holds a joint account, link it from both holders' sessions and check that the EB
   `identification_hash` matches across them.
4. **Redirects in every app**, on an instance reachable only over Tailscale: in the native apps,
   `ASWebAuthenticationSession` with the instance's https callback page handing off to a custom URL scheme; in
   the web app, a full-page redirect to an https callback route. Test BoursoBank, CA and CMSO, including the
   switch to each bank's SCA app and back. Confirm EB accepts a tailnet https redirect URL.
5. **Email EB support** (support.api@enablebanking.com):
   - Can each household member register their own restricted app and run it on the owner's instance, as
     `PRODUCT.md` plans? Can a co-holder link a joint account?
   - What is the linked-accounts cap?
   - If not: what is the minimum monthly invoice for a household-sized contract, and do they contract with an
     individual or a micro-entrepreneur?
6. **Quotes from Bridge and Powens**, only if spike 5 fails (Powens has a Startup Program): minimum fee for a
   household, whether a legal entity is required, agent-status requirements when only hosted Connect/Webview
   is used, and whether Powens' BoursoBank "fallback" (webauth) refreshes unattended.
7. **Import corpus**: export 3 months from each first supported bank in every format it offers, and record
   which formats CA Aquitaine and CMSO actually provide; write golden-file tests for encoding, separators,
   FITID stability and overlap dedupe; run them through the web upload and the native paths.

**Not verified (treat as unknowns)**

- EB contract prices and KYB rules for individuals; per-bank consent maximum and data fields (needs an
  authenticated `/aspsps` call or a live link); whether a member's app may run on an instance someone else
  administers (spike 5); whether EB accepts a tailnet redirect URL and banks a tailnet `Psu-Ip-Address`
  (spikes 1, 4).
- Bridge/Powens/Tink/Salt Edge prices; whether Bridge/Powens contract with individuals; whether the Powens
  stability snapshot (2026-07-29, demo domain) is current; data quality and unattended refresh of Powens'
  BoursoBank webauth fallback.
- Whether CA Aquitaine's PSD2 API returns every payment account through EB (Bridge reports non-compliance).
- Exact export formats for CA Aquitaine and CMSO personal accounts (no official text found); the
  Euro-Information CM/CIC formats rest on a 2004 forum post.
- GoCardless per-bank coverage (not re-checked; product closed). TrueLayer's French coverage outside Data v3.
  Lunch Flow's French bank list, the continuity of its GoCardless supply, and whether its terms let a
  third-party app use a Personal API key.
- PSR re-authentication details; whether serving people outside one's household without a contracted AISP
  counts as "profession habituelle" under L521-2 (no ACPR position found; take legal advice first). The CMF
  articles (L133-16, L133-19, L521-2, L522-1, L522-11-2, L572-5) were read on codes.droit.org's Légifrance
  export, not on Légifrance, which blocks automated fetching: check them there by hand.
- The state of woob's `cmso` module in 2026.

## Sources

**Product**: [PRODUCT.md](../../PRODUCT.md)

**Enable Banking (primary)**: [Terms of Service](https://enablebanking.com/terms) ·
[Linked accounts](https://enablebanking.com/docs/api/linked-accounts/) · [FAQ](https://enablebanking.com/docs/faq/) ·
[API reference](https://enablebanking.com/docs/api/reference/) · [France specifics](https://enablebanking.com/docs/markets/fr/) ·
[January 2026 changelog](https://enablebanking.com/blog/2026/02/17/enable-banking-changelog-january-2026) ·
[Coverage list](https://enablebanking.com/open-banking-apis/) ([FR40618](https://enablebanking.com/open-banking-apis/FR40618),
[FR13306](https://enablebanking.com/open-banking-apis/FR13306), [FR10278](https://enablebanking.com/open-banking-apis/FR10278),
[FR15489](https://enablebanking.com/open-banking-apis/FR15489), [FR15519](https://enablebanking.com/open-banking-apis/FR15519),
[FR15589](https://enablebanking.com/open-banking-apis/FR15589), [FR30066](https://enablebanking.com/open-banking-apis/FR30066)) ·
[Code samples](https://github.com/enablebanking/enablebanking-api-samples)

**Other providers (primary)**: [GoCardless signups disabled](https://bankaccountdata.gocardless.com/new-signups-disabled) ·
[GoCardless BAD overview](https://docs.gocardless.com/docs/bank-account-data) ·
[Powens licence](https://www.powens.com/) ·
[Powens quick start](https://docs.powens.com/documentation/integration-guides/quick-start) ·
[Powens glossary](https://docs.powens.com/documentation/ressources/glossary) ·
[Powens transactions intro](https://docs.powens.com/documentation/integration-guides/transactions/introduction-to-transactions) ·
[Powens integration guide](https://docs.powens.com/documentation/integration-guides/transactions/transactions-integration-guide) ·
[Powens Wealth & Loans](https://docs.powens.com/documentation/integration-guides/wealth-and-loans) ·
[Powens SCA & states](https://docs.powens.com/documentation/integration-guides/sca-and-connection-states) ·
[Powens webhooks](https://docs.powens.com/documentation/integration-guides/webhooks) ·
[Powens iOS SDK](https://docs.powens.com/documentation/sdk/powens-connect-ios) ·
[Powens connectors (demo domain)](https://integrate.biapi.pro/2.0/connectors?expand=sources) ·
[Bridge licence](https://www.bridgeapi.io/securite-conformite) · [Bridge dashboard](https://docs.bridgeapi.io/docs/dashboard) ·
[Bridge credentials](https://docs.bridgeapi.io/docs/api-credentials) · [Bridge data fetching](https://docs.bridgeapi.io/docs/data-fetching) ·
[Bridge lifecycle](https://docs.bridgeapi.io/docs/manage-the-lifecycle) · [Bridge SCA](https://docs.bridgeapi.io/docs/strong-customer-authentication-2025) ·
[Bridge deferred debit](https://docs.bridgeapi.io/docs/operation-types) · [Bridge account types](https://docs.bridgeapi.io/docs/connect-user-journey) ·
[Bridge webhooks](https://docs.bridgeapi.io/docs/webhooks-events) · [Bridge providers](https://providers.bridgeapi.io/)
([data](https://providers-data.bridgeapi.io/providers.json)) ·
[Bridge incidents](https://providers.bridgeapi.io/incidents) ([data](https://providers-data.bridgeapi.io/provider-incidents.json)) ·
[Tink pricing](https://tink.com/pricing/) ·
[Salt Edge docs v6](https://docs.saltedge.com/v6/) · [Yapily pricing](https://www.yapily.com/pricing) ·
[TrueLayer data docs](https://docs.truelayer.com/docs/enable-your-users-to-connect-their-bank-account) ·
[Plaid billing](https://plaid.com/docs/account/billing/) · [Plaid EU coverage](https://plaid.com/docs/institutions/europe/) ·
[Lunch Flow](https://www.lunchflow.app/) · [Lunch Flow docs](https://www.lunchflow.app/docs/guides/get-started/welcome) ·
[Lunch Flow Platform API](https://www.lunchflow.app/docs/api/platform-api-overview) ·
[Lunch Flow API destination](https://www.lunchflow.app/docs/guides/destinations/api)

**Banks (primary)**: [BoursoBank export FAQ](https://www.boursobank.com/aide-en-ligne/banque/releves-et-documents-bancaires/question/comment-et-sous-quels-formats-puis-je-exporter-mes-mouvements-de-compte-et-de-carte-bancaire-26780) ·
[CA online-banking tutorials](https://www.credit-agricole.fr/particulier/informations/tutos-banque-en-ligne.html) ·
[CA Charente-Périgord portal help PDF (2019)](https://www.credit-agricole.fr/content/dam/assetsca/cr824/npc/documents/tutoriels/aide-nouveau-portail-client.PDF) ·
[CM Océan remote-banking T&Cs 09/2024](https://www.creditmutuel.fr/telechargements/cmo/informations-reglementaire/conditions-generales/2024_conditions_generales_banque_a_distance.pdf) ·
[CM TWEB](https://www.creditmutuel.fr/fr/associations/comptes/tweb.html) ·
[CM Alliance organisation](https://www.creditmutuelalliancefederale.fr/en/who-are-we/our-organization.html) ·
[CMNE joins Alliance](https://presse.creditmutuelalliancefederale.fr/?p=5827) ·
[BFCM: CMAF presentation](https://www.bfcm.creditmutuel.fr/en/bfcm/cmaf-presentation.html) ·
[CM MABN identity 2026](https://www.creditmutuel.fr/telechargements/cmmabn/institutionnel/depliant_identite_2026.pdf)

**Regulation (primary)**: [RTS 2018/389](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389) ·
[RTS amendment 2022/2360](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32022R2360) ·
[PSD2](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32015L2366) ·
[EBA Q&A 2018_4210](https://eba.europa.eu/single-rule-book-qa/qna/view/publicId/2018_4210) ·
[GDPR](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679) ·
[CJEU C-184/20](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:62020CJ0184) ·
[CJEU C-252/21](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:62021CJ0252) ·
[OEIL PSR 2023/0210(COD)](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0210(COD)) ·
[OEIL PSD3 2023/0209(COD)](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0209(COD)) ·
[OEIL FiDA 2023/0205(COD)](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0205(COD)) ·
[Commission work programme 2027, annexes (COM(2026) 710)](https://commission.europa.eu/document/download/9ad0afe2-f9a8-46e9-bd3b-74c2de496c48_en?filename=COM_2026_710_1_annexe_EN.pdf) ·
[ACPR Forum Fintech 2021: payment-service statuses](https://acpr.banque-france.fr/system/files/2024-11/forum_fintech_2021_atelier_services_de_paiement_procedures_reglement.pdf) ·
[Assemblée nationale report l15b0894 (2018-04-19)](https://questions.assemblee-nationale.fr/dyn/15/rapports/368/l15b0894_rapport-fond.pdf)

**Platforms (primary)**: [Apple: ASWebAuthenticationSession https callback](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/https(host:path:)) ·
[Apple: custom-scheme callback](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:)) ·
[Apple: supporting associated domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains) ·
[MDN: share_target](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest/Reference/share_target) ·
[Tailscale IP addresses](https://tailscale.com/docs/concepts/tailscale-ip-addresses)

**Secondary**: [Actual: GoCardless](https://actualbudget.org/docs/advanced/bank-sync/gocardless/) ·
[Actual: Enable Banking](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/) ·
[Actual #8788](https://github.com/actualbudget/actual/issues/8788) · [Firefly III #13115](https://github.com/firefly-iii/firefly-iii/issues/13115) ·
[woob modules](https://gitlab.com/woob/woob/-/tree/master/modules) ·
[woob cragr commits](https://gitlab.com/woob/woob/-/commits/master/modules/cragr) ·
[woob creditmutuel commits](https://gitlab.com/woob/woob/-/commits/master/modules/creditmutuel) ·
[woob boursorama commits](https://gitlab.com/woob/woob/-/commits/master/modules/boursorama) ·
[woob cmso commits](https://gitlab.com/woob/woob/-/commits/master/modules/cmso) ·
[woob issues](https://gitlab.com/woob/woob/-/issues/?search=creditmutuel) · [Kresus FAQ](https://kresus.org/en/faq.html) ·
[CMF, Légifrance export (codes.droit.org, update 2026-09-01)](https://codes.droit.org/PDF/Code%20mon%C3%A9taire%20et%20financier.pdf) ·
[CMF L521-2 (translation)](https://french-business-law.com/rench-legislation-art/article-l521-2-of-the-french-monetary-and-financial-code) ·
[CMF L133-16](https://french-business-law.com/rench-legislation-art/article-l133-16-of-the-french-monetary-and-financial-code) ·
[CMF L133-19](https://french-business-law.com/rench-legislation-art/article-l133-19-of-the-french-monetary-and-financial-code) ·
[Arthur Cox: PSD3/PSR final texts](https://www.arthurcox.com/insights/psd3-and-psr-final-compromise-texts-published/) ·
[Hogan Lovells: provisional agreement](https://www.hoganlovells.com/en/publications/final-texts-for-psd3-and-psr-awaited-as-european-parliament-and-council-of-eu-announce-provisional) ·
[CM/CIC formats forum (2004)](https://www.generation-nt.com/reponses/money-credit-lyonnais-c-est-fini-entraide-2264791.html) ·
[CA/Linxo Connect press](https://presse.credit-agricole.com/assets/2022-09-28-linxo-connect-wealth-pdf-4716-9ed05.html?dl=1)
