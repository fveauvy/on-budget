# Bank data: alternatives for transaction history

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: Friends-and-family rows; device-held keys.

Research date: 2026-11-07, revised the same day after a fact-check (iOS 27 Shortcuts, ADR 0001, the public-repo rule).
Prices, terms and availability are as of Nov 2026 unless dated otherwise.
Builds on [bank-aggregation.md](bank-aggregation.md) (providers, coverage, consent rules, import formats) and does not
repeat it. "Secondary" marks forums, blogs, vendor claims and search excerpts. Confidence: **H** high, **M** medium,
**L** low.

## Question

The owner asked: "Is Enable Banking the only solution? Are there other ways to sync accounts? I only need the
transaction history." Which routes give transaction history to an owner, to a household member such as a partner, and
to friends and family in other households, at the first supported banks (BoursoBank, Crédit Agricole Aquitaine, Crédit
Mutuel du Sud-Ouest of the Arkéa group) and at any French bank? At what cost and reliability, and within which terms
and laws? Under [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md), "anyone who wants on-budget hosts their
own instance", so each friend's household owns its own instance.

## TL;DR

**Enable Banking (EB) is the only free, licensed, self-serve API route found, and it covers only the person who links
the accounts.** Every other automated route is paid (a reseller or a contract), reserved for organisations, or partial.
Cheap complements are real but partial: Apple Pay taps through the Shortcuts Transaction trigger (seconds when it works,
Apple Pay only), bank-app notifications through the iOS 27 notification automation (untested), and bank alerts or
exports (manual or partial). FinanceKit is not available in France.

| Case | 1st choice | 2nd | 3rd | Always |
|---|---|---|---|---|
| **Owner**, on their own instance | Own EB restricted app (**H**) | Provisional entries: Apple Pay tap capture → App Intent (**M**); on iOS 27, bank-app notifications → notification automation, where the bank sends one per card payment (**L**, spike) | The bank's own SMS or e-mail alerts, as a backstop during provider outages (**L**, spike) | File import (**H**) |
| **Household member** (e.g. a partner) on the owner's instance | Own EB restricted app, once EB confirms in writing who may hold its key (**M**) | Member-paid open-banking.io feed (EB underneath), once the vendor confirms server use (**L**) | Provisional entries, as for the owner (**M**/**L**) | File import (**H**) |
| **Friends and family** (other households) | Their own instance and EB app (ADR 0001): the owner row applies, at their bank (**H**) | Where EB lacks their bank: Bankin' Plus e-mailed export, Bridge or Powens underneath (**M**) | open-banking.io (€3/month, EB underneath), to skip EB's Control Panel steps (**L**) | File import (**H**) |

1. **Per-person EB apps** satisfy the ownership rule (each person links their own accounts), and each owner's own app on
   their own instance is allowed. Because friends self-host (ADR 0001), no server holds another household's key. The
   clash with "grant any right to access or use … the API to any third party" and "don't … share [the private key] with
   anyone" narrows to a household member's app run by the owner's instance. A key kept on the member's own device is
   more defensible but less reliable. Ask EB (§1). **H** for owners, **M** for members.
2. **FinanceKit / Wallet**: FinanceKit serves the US (Apple Card, Apple Cash, Savings) and the UK (open banking) only.
   It requires App Store distribution there and an organisation developer account. Wallet "connected accounts" cover
   UK issuers and US PayPal only. Nothing for France as of iOS 26–27 (§2). **H**
3. **Shortcuts "Transaction" trigger**: documented, can run without confirmation. Reported fields: card, merchant,
   amount, name. Reported failures: timeouts when the issuer is slow, triggers on declined payments, and an "Always
   Allow" prompt that returns on every payment when the App Intent comes from a pre-configured shortcut file. On iOS 27,
   automations live inside shortcuts, can ship in a shared shortcut and sync to the person's other devices turned off,
   so each device still needs one manual step. Good for instant provisional entries matched later to bank data, never
   as the source of truth (§3). **M**
4. **Banks**: none of the three offers scheduled exports. Alerts: BoursoBank can notify "chaque retrait ou paiement
   effectué avec votre carte"; CA Aquitaine's 2026 conditions list free e-mailed mini-statements and real-time alerts,
   but its 2027 brochure lists free alerts by SMS only and weekly mini-statements only in a paid module; CMSO alerts
   cover the balance and deferred movements (direct debits, transfers, cheques), not card payments as far as stated.
   No API lets on-budget's apps read another app's notifications; on iOS 27 a user-installed notification automation
   can hand them to an App Intent. Nothing found forbids parsing your own alerts, but formats are undocumented (§4).
   **M**
5. **Consumer apps**: Bankin' Plus (€39.99/yr) e-mails CSV/Excel exports on demand; Linxo's legacy app exports
   CSV/QIF/OFX from its website (help text last edited in 2023); no Finary bank export found; none has an official API.
   open-banking.io (Danish, built on EB, €3/month + €1 per extra account, API, polling only) was commercialised in
   May 2026 and onboards no partners. Lunch Flow still relies on GoCardless; BankSync has no EU coverage (§5). **M**
6. **Licensed small-scale offers**: the Powens Startup Program and the Salt Edge Partner Program are sales-led and
   meant for companies; Ponto prices per account but targets businesses; EB's Control Panel says unrestricted production
   "is available only to organisations". Becoming an AISP's agent, or registering as an AISP, is not realistic for a
   hobby project (§6). **H**
7. **"Transaction history only"** is still AIS: same consent, SCA at least every 180 days, 4 unattended fetches a day,
   under PSD2 and its RTS until the PSR applies (about late 2028). It does mean PSD2 APIs suffice (no password-based
   connectors) and that user-supplied data (files, e-mails, taps, notifications) covers much of the need outside AIS
   (§7). **M**

## 1. Per-person Enable Banking apps

### 1.1 What the terms allow and forbid

EB Terms of Service, last updated 2026-01-09 ([ToS](https://enablebanking.com/terms)):

- Production use "is limited to Linked Accounts and is available solely for evaluation purposes or for the personal use
  of private individuals. The number of Linked Accounts per Application may be limited."
- No right for "(ii) accessing account information that does not belong to the Control Panel user who associated the
  Linked Accounts with the application accessing them, or (iii) using the API for any commercial purpose whatsoever."
- "You shall not license, sublicense, … transfer, … disclose, or make accessible to any third party … or grant any
  right to access or use the Control Panel or the API to any third party." The licence is "non-transferable".
- "You must … keep credentials secure, and protect any private keys or tokens used for authentication with the API."
- On breach: termination, "no limitation of liability shall apply to you", and afterwards no new account or
  "Application to access the same Accounts without first seeking clarification from Enable Banking".

EB documentation:

- Restricted mode serves evaluation "as well as individual non-commercial use"; the app "will stay in this state until
  an agreement has been signed" ([linked accounts](https://enablebanking.com/docs/api/linked-accounts/)).
- "applications can't be made available to the public before a contract is signed"
  ([FAQ](https://enablebanking.com/docs/faq/#can-i-test-the-api-before-signing-a-contract-with-enable-banking)).
- A restricted app lets "you (and any user of your application)" authorise, but the API strips every account not linked
  in the Control Panel ([FAQ](https://enablebanking.com/docs/faq/#why-does-my-restricted-application-receive-an-empty-list-of-accounts-after-successful-authorisation)).
- "keep the private key in secret (e.g. don't expose it to client, share with anyone nor embed into mobile or other apps
  intalled to user devices)"; JWTs are signed with RS256 only ("only RS256 is supported";
  [API ref](https://enablebanking.com/docs/api/reference/#private-key-and-certificate-generation)).
- The Control Panel's activation dialog: "unrestricted access in the production environment is available only to
  organisations. Individuals can activate applications by linking their accounts and may continue using them in
  restricted mode, adding more linked accounts if needed" (UI string in the
  [Control Panel bundle](https://enablebanking.com/js/app.83127064.js) loaded by enablebanking.com/cp on 2026-11-07).

| Model | Ownership rule (ii) | Third-party and key rules | Verdict |
|---|---|---|---|
| Owner's own app on their own instance (each friend's household included, ADR 0001) | ✅ | ✅ | Allowed **H** |
| Household member's own app, key held by the owner's instance | ✅ | ⚠️ the member lets an instance they don't administer use their app and key | Grey; ask EB **M** |
| Other households' apps, keys held by one person's server | ✅ | ❌ in substance a hosted service for third parties without a contract | Ruled out by ADR 0001; likely outside the terms anyway **M** |
| Each person's key on their own device; the app calls EB directly | ✅ | ⚠️ no third party gets access, but contradicts "nor embed into mobile … apps" | Most defensible variant for a member; ask EB **M** |
| Other people's accounts linked into the owner's app | ❌ | — | Forbidden **H** |

Practice (secondary): Actual Budget "stores a single set of Enable Banking credentials for the whole server, shared by
all budget files" ([Actual](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/)). That describes
self-hosting habits, not EB's view of shared servers.

### 1.2 Operational implications

- **Onboarding per person** (two SCAs): create an EB Control Panel account; register a production app with on-budget's
  `https` redirect URL; "Activate by linking accounts" through the bank's SCA, linking *every* account to be used; then
  authorise a session through the API, which "applies even when using the same account"
  ([linked accounts](https://enablebanking.com/docs/api/linked-accounts/)). Renew that session at least every 180 days
  (bank-dependent; bank-aggregation.md §4.1). on-budget can guide but not automate the Control Panel steps.
- **Key custody**, three options:
  1. The person generates the key in the Control Panel ("Generate in the browser (using SubtleCrypto) and export
     private key"; same API ref section) and uploads the `.pem` to on-budget: a plain "share with anyone" breach.
  2. on-budget generates the key pair; the person pastes only the certificate into the Control Panel. The key never
     leaves the instance, but the owner still operates the member's app.
  3. The on-budget iOS/macOS app generates the key on the person's device and keeps it in the Keychain. The Secure
     Enclave cannot hold it: EB needs an RS256 (RSA) key, and the Enclave holds P-256 keys and, since the 26 releases,
     ML-KEM and ML-DSA keys through CryptoKit, but no RSA keys
     ([CryptoKit](https://developer.apple.com/documentation/cryptokit/secureenclave)). Apple's older Security article
     still says it "Works only with NIST P-256 elliptic curve keys"
     ([Apple](https://developer.apple.com/documentation/security/protecting-keys-with-the-secure-enclave)). At most an
     Enclave key can encrypt the RSA key at rest; the RSA key is still decrypted in memory to sign. The device fetches,
     then syncs transactions through on-budget's normal sync. iOS gives no schedule guarantee ("the system doesn't
     guarantee launching the task at the specified date";
     [BGTaskRequest](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate)), so
     fetch on app open (user present, so no 4/day cap) and from a Mac login item. Web-only members can't use it.
- **Instance-held keys** (options 1–2): one JWT signer per member app (`kid` = app ID, TTL ≤ 24 h, same API ref), keys
  encrypted at rest, consent and rate limits tracked per app. A finding of breach could end the household's apps at
  once, and the shared redirect domain makes the pattern visible to EB (hiding it would breach the "falsify the origin"
  clause).
- **Cost** €0; **effort** mostly onboarding support.

**Verdict**: fine for the owner, and so for each friend who runs their own instance (ADR 0001) (**H**). For a household
member such as a partner, worth a written question to EB (spike 1) (**M**). Holding other households' apps on one
server is ruled out by ADR 0001, and was likely outside EB's terms anyway.

## 2. Apple Wallet and FinanceKit in France

- FinanceKit ([Get started](https://developer.apple.com/financekit/), checked 2026-11-07):
  - US, iOS 17.4+: "Apple Card …, Apple Cash …, Savings".
  - UK, iOS 18.4+: "uses open banking to access various financial data in the U.K. from institutions, including
    Barclays, … Monzo, Nationwide, NatWest, … Santander."
  - Apps must "Be listed in the Finance category … and distributed through the App Store for iPhone in the United
    States or United Kingdom"; the Account Holder requests the entitlement, per bundle ID.
  - Apps must "hold an organization-level Apple Developer account" ([docs](https://developer.apple.com/documentation/financekit)).
- Wallet "connected accounts" (balance and up to two years of history) are "Available for eligible card issuers only":
  a UK brand list, plus PayPal debit cards in the US. The French-language page lists the same and no French issuer
  ([EN](https://support.apple.com/en-us/111117), [FR](https://support.apple.com/fr-fr/111117)).
- Wallet lists recent transactions for cards used with Apple Pay ("You might see transactions for Apple Pay and your
  physical cards"; [EN](https://support.apple.com/en-us/104954), same text on [FR](https://support.apple.com/fr-fr/104954)),
  but no API exposes them outside FinanceKit.
- iOS 26–27: the Get started page still names only the US and the UK. Apple's FinanceKit change log is no evidence
  either way: it lists only June 2024 ([updates](https://developer.apple.com/documentation/updates/financekit)) and
  omits later additions such as `BackgroundDeliveryExtension` (iOS 26.0+;
  [docs](https://developer.apple.com/documentation/financekit/backgrounddeliveryextension)). No country expansion
  found. **M** (absence)

**Verdict**: unusable for on-budget (French banks, no public App Store listing, and an organisation account needed).
Re-check after each WWDC. **H**

## 3. Shortcuts "Transaction" automation → App Intent

**Documented by Apple**

- Trigger, iOS 26 guide: "When I tap: Select a card to trigger an automation whenever it's tapped"
  ([guide, iOS 26](https://support.apple.com/en-gb/guide/shortcuts/apd65c67538a/ios)). The current iOS 27 guide (10.0)
  has no trigger-by-trigger pages, so the trigger's iOS 27 behaviour is undocumented (spike 2).
- "Transaction", "Message" and "Email" are among the automations that "can be run automatically": on iOS 26 once "Ask
  Before Running" is off ([guide, iOS 26](https://support.apple.com/en-gb/guide/shortcuts/apd602971e63/ios)), on iOS 27
  through "Privacy" then "Allow Running When Locked"
  ([guide, iOS 27](https://support.apple.com/guide/shortcuts/add-automations-apdfbdbd7123/10.0/ios/27)).
- Set-up changed with iOS 27 (released 2026-09-14; 27.2 current, [Apple](https://support.apple.com/en-us/100100)):
  "Automations are now built directly into shortcuts, including shared shortcuts" and "Automations now sync between
  devices" ([What's new](https://support.apple.com/en-us/149045)). They remain "specific to a device" and "sync to other
  devices with the automation disabled"
  ([guide, iOS 27](https://support.apple.com/guide/shortcuts/intro-to-automation-shortcuts-apd690170742/10.0/ios/27));
  "Personal automation shortcuts are disabled by default during sync"
  ([sync](https://support.apple.com/guide/shortcuts/sync-shortcuts-apdb3a4240b0/10.0/ios/27)). So on-budget can publish
  one shared shortcut that carries the Transaction automation and the `RecordWalletTap` action; each person still adds
  it and turns it on, device by device. Whether a shared shortcut's automation arrives on or off is not documented. No
  API for an app to create or enable an automation was found (**M**). On iOS 26 an automation "will not sync to other
  devices" ([guide, iOS 26](https://support.apple.com/en-gb/guide/shortcuts/apd690170742/ios); the en-gb guide still
  shows this text), so it is built by hand on each iPhone.
- An App Intent exposes a Shortcuts action with typed parameters and can write to the app-group database without
  opening the app ([App Intents](https://developer.apple.com/documentation/appintents); app group in
  apple-client-stack.md).

**Reported (secondary)**

- Fields: "only Transaction, Card, Merchant, Amount and Name"
  ([Apple Community, Feb 2026](https://discussions.apple.com/thread/256252606)); filters by card, category and merchant
  ([iDropNews, Jul 2023](https://www.idropnews.com/ios-17/this-new-ios-17-shortcut-will-help-you-keep-track-of-your-spending/197589/)).
- Reliability: "frequently times out" while waiting for the issuer's Wallet notification ("Hit timeout waiting for
  transaction"); "activated even when the Transaction is declined"; bug reports FB14035016 and FB16379100; reports run
  from Oct 2024 to Feb 2026, one user failing on a Mastercard only
  ([forum](https://developer.apple.com/forums/thread/765516)).
- Permission prompts: with a Wallet "After Payment" trigger and an App Intent, "Always Allow" does not persist and the
  prompt returns on every payment when the action comes from "a pre-configured .shortcut file that has variable
  descriptors embedded for the parameters"; it persists when the user "adds the App Intent action inline and maps
  fields manually" (iOS 18.x, posted about Oct 2026, no reply; [forum](https://developer.apple.com/forums/thread/850369)).
  A shared on-budget shortcut has exactly that shape (spike 2).
- Practice: ActualTap posts each tap to Actual Budget, maps the Wallet card name to an account, and uses "Run
  Immediately" ([ActualTap](https://github.com/MattFaz/actualtap)).
- All three banks support Apple Pay ([BoursoBank](https://www.boursobank.com/aide-en-ligne/carte-bancaire/paiement-mobile),
  [CA Aquitaine](https://www.credit-agricole.fr/ca-aquitaine/particulier/compte/paiement-mobile/apple-pay.html),
  [CMSO](https://www.cmso.com/reseau-bancaire-cooperatif/web/comptes-et-cartes/payer-autrement/apple-pay)). How fast
  each issuer's Wallet notification arrives is untested.

**Limits**: Apple Pay taps only (no physical card, direct debit, transfer or salary; bank-app notifications can cover
physical-card payments, §4); an authorisation, not a booking (amounts change, holds, declines); no bank ID to dedupe on;
iPhone only (whether Watch taps trigger it is unverified, **L**); nothing on Android or the web.

**Design**: the intent writes a *provisional* transaction (source `wallet_tap`) that updates safe-to-spend and limits at
once. The bank feed later confirms it by account (card mapping), amount, date window (0–5 days) and merchant
similarity, keeping the user's category. Unmatched taps go to the review inbox after ~7 days. Entries parsed from bank
notifications (§4) take the same path. **M**

## 4. Bank-side exports and notifications

| | BoursoBank | CA Aquitaine | CMSO (Arkéa) |
|---|---|---|---|
| Manual export | CSV, OFX, QIF (bank-aggregation.md §8) | Web download; formats unverified (ibid.) | Not documented on cmso.com (spike) |
| Scheduled export | None found | None; a free daily e-statement ("Relevé journalier – format électronique uniquement") ([CGB 2026](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/BROCHURES_INTEGRALES_CGB_DETAILLEES_2026_Web_45-06-2026.pdf)) | None; electronic statements more often than monthly, up to daily ([2026 extract](https://www.moneyvox.fr/tarif-bancaire/credit-mutuel-du-sud-ouest/pdf/tarifs-2026.pdf), archived copy, layout ambiguous) |
| E-mail alerts | None found | "Mini relevés : Par e-mail", free (CGB 2026, the only CGB published as of 2026-11-07; frequency not stated). Both brochures list weekly "mini relevés (hebdomadaire)" only inside the paid "Module Suivi @ssuré": 2,40 €/month in 2026, 2,25 €/month from 1 Jan 2027 ([2026](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/PlaquetteConditionsTarifairesParticuliers_2026.pdf), [2027](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/0275_76125_PlaquetteConditionsTarifairesParticuliers_2027_Web.pdf)) | Alerts and "Info Solde" by "push notification ou … e-mail" ([Bud'jet](https://www.cmso.com/reseau-bancaire-cooperatif/web/comptes-et-cartes/piloter-vos-comptes/gestion-de-budget-en-ligne)) |
| SMS alerts | Card authorisations above 200 €: push to the trusted device; "Si vous n'en avez pas, vous recevrez un SMS" ([FAQ](https://www.boursobank.com/aide-en-ligne/carte-bancaire/utilisation-quotidienne-de-ma-carte/notifications-et-suivi-des-operations/question/comment-mettre-en-place-des-notifications-sur-les-paiements-et-retraits-effectues-par-carte-219)) | Free, incl. "Alerte Temps réel de gestion (solde créditeur, opération en débit ou en crédit, libellé, encours débit fin de mois)" (CGB 2026); "uniquement par SMS" from 1 Jan 2027, when the brochure's free alerts are all SMS ([2027 brochure](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/0275_76125_PlaquetteConditionsTarifairesParticuliers_2027_Web.pdf)) | Not found |
| Push | Optional notifications "pour chaque retrait ou paiement effectué avec votre carte", so physical-card payments too (FAQ; channel not stated); custom alerts: low balance, transfer done ([services](https://www.boursobank.com/banque/services-bancaires)) | Not checked | Same alerts as e-mail |
| Content | Card authorisations, incl. holds; card refunds | Balance; operations matching rules; labels | Balance; "mouvements sur votre compte (ceux que vous avez initiés mais dont le délai d'exécution peut être différé tel que les prélèvements, virements, chèques...)" and "des mouvements à venir le lendemain"; card payments not stated; amounts can be hidden |

**Can on-budget read them?**

- Other apps' push notifications, through an API: no. iOS notification APIs manage notifications "for your app or app
  extension" ([UNUserNotificationCenter](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)).
  **H** for on-budget's apps. Two routes exist outside that API:
  - **iOS 27 notification automation**, installed by the user: a new "Notifications ('When I receive a notification
    from News')" automation ([What's new](https://support.apple.com/en-us/149045)) "runs when a notification is
    received from a specific app", with keyword filters, and Apple wants notifications "easy to parse and interpret
    within a shortcut" ([WWDC26 session 310](https://developer.apple.com/videos/play/wwdc2026/310/)). A shortcut could
    pass a bank app's notification (BoursoBank's per-payment card notifications, CMSO's alerts) to an on-budget App
    Intent as a provisional entry, physical-card payments included. Limits: only on the iPhone or iPad that receives the
    bank app's notifications (Mac not checked); undocumented formats; a developer reports runs silently skipped on
    iOS 27.0.1, 7 of 10 notifications handled (FB25255777; [forum](https://developer.apple.com/forums/thread/852334),
    secondary); Apple's list of automations that can run without asking does not include it
    ([guide, iOS 27](https://support.apple.com/guide/shortcuts/add-automations-apdfbdbd7123/10.0/ios/27)). **L**,
    spike 8.
  - **Bluetooth accessories**: ANCS gives BLE accessories "a simple and convenient way to access many kinds of
    notifications that are generated on iOS devices"
    ([ANCS](https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Introduction/Introduction.html)).
    Since iOS 26.5, on iPhones in the EU only, the Accessory Notifications framework gives a paired accessory's
    companion-app extension "the complete notification data" for the apps the person chooses
    ([Accessory Notifications](https://developer.apple.com/documentation/accessorynotifications)). Both need an
    accessory that on-budget doesn't make: out of scope. **H**
- SMS: on the iPhone only, with a Message automation filtered on sender and "Message Contains"
  ([guide, iOS 26](https://support.apple.com/en-gb/guide/shortcuts/communication-triggers-apdd711f9dff/9.0/ios/26)) that
  passes the text to an App Intent. That the text arrives as Shortcut input is reported by users, not documented.
  Message is still in iOS 27's list of automations that can run without asking. **M**
- E-mail: one address per member on a dedicated mailbox that the instance polls over IMAP (outbound only, so it works
  on a private network as `PRODUCT.md` requires), or the on-device Mail trigger (sender, subject). Accept a message only
  when a DKIM signature of the bank's (or Bankin's) domain verifies, or anyone who learns the address could inject
  transactions. That works for a provider-side alias or redirect that keeps the message intact ("DKIM signatures may
  remain valid", [RFC 7960 §3.2.1](https://www.rfc-editor.org/rfc/rfc7960#section-3.2.1)) and for a forward as an
  attachment, checked on the attached original. An inline forward from a mail client is a "ReSender"
  ([RFC 7960 §3.2.2](https://www.rfc-editor.org/rfc/rfc7960#section-3.2.2)): a new message, which the original
  signature over the header fields and body hash doesn't cover ([RFC 6376 §3.5, §3.7](https://www.rfc-editor.org/rfc/rfc6376)).
  Verify on receipt, since a signer "MAY remove or revoke a public key at any time" (RFC 6376 §6). **M**
- PDF statements need a login each time: the same effort as a CSV export.

**Is it permitted?**

- It is the customer's own data, and no clause forbidding forwarding was found in the documents read (not a full T&C
  review). **L**
- Probably not AIS: PSD2 recital 28 describes AIS data as "accessed via online interfaces of the account servicing
  payment service provider" ([PSD2](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32015L2366)). No
  ACPR position found. **M**
- GDPR still applies: each owner is the controller for their household members' data (bank-aggregation.md §7.2); under
  ADR 0001 no instance holds other households' data.

**Verdict**: a complement, not a feed. Two spikes are worth it: the iOS 27 notification automation on card-payment
notifications (spike 8), and CA's real-time SMS alerts as a backstop during CA outages (spike 3). CMSO alerts flag the
balance and deferred movements (direct debits, transfers, cheques), not card payments as far as stated. **M**

## 5. Consumer aggregator apps and resellers

| Product | Data route | Export / API | Price | Terms that matter | Fit |
|---|---|---|---|---|---|
| **Bankin'** | AIS by Bankin' as agent of Bridge (Perspecteev), REGAFI 381188, "et également par notre partenaire (ii) Powens", whose own CGU then apply ([CGU, applicable from 13 Mar 2026](https://bankin.com/fr/cgu.html)) | Plus/Pro only: chosen accounts and dates, "export de données aux formats Excel et CSV par email" ([help, 2026-10-14](https://support.bankin.com/hc/fr/articles/360007523551-Exporter-votre-historique-bancaire)); no API | Plus €39.99/yr or €4.99/mo ([help](https://support.bankin.com/hc/fr/articles/360006559578-Pr%C3%A9sentation-de-Bankin-Plus)) | "utilisation strictement personnelle et à des fins non lucratives" | The account holder triggers the export and forwards it as an attachment, or through a provider redirect, to on-budget (§4) |
| **Linxo** (Oxlin, ACPR-licensed payment institution) | Own AIS ([site](https://linxo.com/)) | Legacy app ("L'application historique") website only: "uniquement depuis le site web", "Télécharger ces opérations", CSV, QIF, OFX ([help](https://support.linxo.com/hc/fr/articles/115003171125-Comment-exporter-mes-donn%C3%A9es-de-Linxo), text last edited 2023-01-11 per the [Zendesk API](https://support.linxo.com/api/v2/help_center/fr/articles/115003171125.json)); nothing found for the newer Linxo Lab app; no API | Premium 4 €/month on linxo.com ([legacy app](https://linxo.com/linxo-app-historique/)); App Store €5.99/mo or €47.99/yr (budgeting-products.md) | Personal finance use ([CGU](https://linxo.com/cgu-linxo/)) | Manual, like a bank export |
| **Finary** | Aggregators incl. Powens and Flanks, plus "web scraping" for some accounts ([help](https://help.finary.com/fr/articles/13462818-comprendre-les-problemes-de-synchronisation)) | No bank-transaction export or API found (help search) | — | — | No |
| **open-banking.io** (Tatic ApS, DK) | "Bank connectivity is provided through Enable Banking Oy" ([terms, 2026-07-17](https://open-banking.io/en/terms)) | Web app, CSV, REST API, 8 SDKs, server-to-server keys ([developers](https://open-banking.io/en/developers)); no webhooks | €3/month incl. 1 account, €1/month per extra (terms) | Private users allowed; no reselling or white-labelling without agreement; keep "API keys and credentials confidential"; "We do not support onboarding new partners" ([FAQ](https://open-banking.io/en/faq)) | Best paid feed for an owner's own server; very young |
| **Lunch Flow** | EU banks "via GoCardless" ([site](https://www.lunchflow.app/)) | API, CSV/OFX | $34.99/yr per user (bank-aggregation.md) | — | Rests on a product closed to new signups |
| **BankSync** | Plaid (US/CA), Salt Edge (UK), Fiskil (AU); no EU provider disclosed ([compliance](https://banksync.io/compliance)) | API | from $4/mo ([pricing](https://banksync.io/pricing)) | Personal or internal use ([terms](https://banksync.io/terms)) | No French coverage disclosed |
| **YNAB** | Its own EU import provider | API; personal tokens "should not be shared" ([API](https://api.ynab.com/)) | — | — | Pointless detour |

open-banking.io details (vendor pages): "Tatic ApS, the company behind open-banking.io, is five years old and has used
AISP access for several years. The product was commercialised in May 2026" ([FAQ](https://open-banking.io/en/faq)).

- Data is sealed under the user's key; "a sync can only start where that key is available: your browser while you are
  signed in, your own API client, or the CLI" (FAQ). An unattended pull by on-budget needs the user's `credentials.json`
  (API key + private key) on the server, which the docs say to keep "server-side only"
  ([developers](https://open-banking.io/en/developers)). The FAQ expects that for one's own server: "your own server
  can fetch new transactions once a day or more often". Fine for an owner; for a member on the owner's instance, the
  same custody question as EB options 1–2.
- "Is there a webhook or notification when new transactions appear? No, only polling" (FAQ).
- "Can we use open-banking.io in an app for our own customers? Unfortunately not. We do not support onboarding new
  partners" (FAQ). That rules out offering it to other households, as ADR 0001 already does.
- It lists "Boursorama Banque", "Crédit Agricole d'Aquitaine" (Beta) and "Crédit Mutuel du Sud-Ouest" (Beta)
  ([banks](https://open-banking.io/en/banks)).
- No EB app to set up: "Nothing to buy or register, and no application to set up first", "no certificates"
  ([self-hosters](https://open-banking.io/en/for-self-hosters)).
- Its "No four-requests-per-day cap" claim (self-hosters) is marketing;
  RTS art. 36(5) still caps unattended access, so ask how scheduled syncs are flagged. Its FAQ says "Most consents last
  90 days", against EB's "180 days for most banks" (bank-aggregation.md §4.1). **L**

## 6. Licensed providers: small-scale offers and agent status

- **Powens Startup Program** ("Launchpad"): for fintech startups "From MVP validation to Series A and beyond"; no
  published criteria or prices; "Schedule a meeting with us" ([Powens](https://www.powens.com/startup-program/)). **H**
- **Salt Edge Partner Program**: for "the legal entity, not possessing an Open Banking license, operating through the
  Partner Program, and having an agreement with Salt Edge"; joined on invitation ("Request an invitation … through the
  'Contact Us' section or reach out to our Sales Team") ([docs v6](https://docs.saltedge.com/v6/),
  [page](https://www.saltedge.com/products/account_information/partner_program)). **H**. The docs' Test status
  ("limited to 100 Connections", "valid for 90 days") is written for "the Client's application", a Client being "the
  legal entity holding an Open Banking license"; whether it applies to Partners is undocumented (unverified).
- **Ponto** (Isabel NV, AISP/PISP authorised by the National Bank of Belgium): "simple monthly subscription per linked
  account", 14-day trial, aimed at businesses ([pricing](https://myponto.com/en/pricing/customer-paying-model/),
  [company](https://myponto.com/en/company/)). €4 per account per month according to a search excerpt (secondary).
  Consumer eligibility and French coverage unverified. **L**
- **EB contract**: "volume based … minimum invoicing per month" plus KYB of a company (bank-aggregation.md). The Control
  Panel adds that unrestricted production "is available only to organisations" (§1.1); whether a micro-entrepreneur
  counts as one is open (spike 1).
- **Agent of an AISP** (the Bankin' model): the principal notifies the regulator of the agent's AML controls and
  "evidence that they are fit and proper persons"; registration takes up to 2 months (PSD2 art. 19); the principal
  remains "fully liable for any acts of … any agent" (art. 20) ([PSD2](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32015L2366)).
  Principals take on commercial partners, not hobby projects. **H**
- **Own registration as an AISP**: lighter than a licence, but still a supervised payment institution that must "hold a
  professional indemnity insurance" (PSD2 arts. 5(3), 33), plus eIDAS certificates and bank integrations. Not
  proportionate. **H**

## 7. Does "transaction history only" change anything?

Baseline: PSD2 and its RTS, as of 2026-11-07. PSD3 repeals PSD2
([Council 8535/26 ADD 1](https://data.consilium.europa.eu/doc/document/ST-8535-2026-ADD-1/en/pdf), 2026-10-21) and the
AIS rules move to the PSR: Council first-reading position 08543/1/2026 published 2026-11-05, Parliament second reading
indicatively on 2026-12-14 ([OEIL](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0210(COD))).
The PSR applies 21 months after entry into force, about late 2028, and adds "dashboards for monitoring, withdrawing or
re-establishing data access" ([Arthur Cox](https://www.arthurcox.com/insights/psd3-and-psr-final-compromise-texts-published/),
secondary; timeline in bank-aggregation.md §7.3). The article numbers below (and in §4, §6) are PSD2's and will change.

- **Regulatory category: no.** AIS is "an online service to provide consolidated information on one or more payment
  accounts" (PSD2 art. 4(16)); the AISP must "access only the information from designated payment accounts and
  associated payment transactions" (art. 67(2)(d)). There is no lighter regime for transactions alone. **H**
- **Consent and refresh: unchanged, until the PSR applies.** SCA at first access and after 180 days; 90 days of history
  SCA-free ([RTS 2022/2360](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32022R2360)); at most 4
  unattended accesses a day ([RTS 2018/389 art. 36(5)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389)).
  **H** for today; the PSR's re-authentication rules were not verified.
- **Scope: smaller, and that helps.** Current-account and card transactions are exactly what PSD2 APIs expose, so no
  password-based "wealth" connector is needed (bank-aggregation.md §3.1). EB's consent object has separate `balances`
  and `transactions` flags ([API ref](https://enablebanking.com/docs/api/reference/#access)); whether French banks
  honour a transactions-only consent is untested. Keep balances anyway: they reveal missing transactions. **M**
- **History depth**: the initial backfill decides it (full history for about an hour after SCA at EB, 90 days after;
  bank-aggregation.md §3.4). Older history comes from a one-off file import.
- **Outside AIS**: files, forwarded e-mails, taps and notifications supplied by the user are, on the recital 28 reading,
  not AIS (§4). With "history only" as the need, these cover any member without a provider app, at the cost of latency.
  **M**

## Comparison

Friends and family run their own instances (ADR 0001), so the Owner column applies to them, at their own banks.

| Option | Owner | Household member | Latency | Cost per person | Terms / law risk | Confidence |
|---|---|---|---|---|---|---|
| EB restricted app, own key on the instance | ✅ | ⚠️ ask EB | ≤ 6 h (4/day) | €0 | Low (owner) / medium (member) | H / M |
| EB restricted app, key on own device | n/a | ⚠️ ask EB | On app open | €0 | Medium | M |
| Contracted AISP (EB, Bridge, Powens) | ✅ | ✅ | ≤ 6 h; webhooks at Bridge/Powens | Monthly minimum, unknown | Low; needs an organisation | H / L on price |
| open-banking.io (paid by the user) | ✅ | ⚠️ ask vendor | When the key is present; polling only | €3–5/month | Medium; young vendor | L |
| Bankin' Plus e-mailed export | ✅ | ✅ | Days (manual trigger) | €39.99/yr | Low | M |
| Bank or Linxo file import | ✅ | ✅ | Days (manual) | €0 | None | H |
| Apple Pay tap capture (Shortcuts) | ✅ (iPhone) | ✅ (iPhone) | Seconds, Apple Pay only | €0 | None | M |
| Bank-app notifications → iOS 27 automation | By bank¹ | By bank¹ | Seconds, when it runs | €0 | Low | L |
| Bank SMS/e-mail alerts | By bank¹ | By bank¹ | Minutes to a week | €0; CA's e-mailed mini-statements may need a paid module in 2027 | Low (GDPR) | L |
| FinanceKit / Wallet | ❌ | ❌ | — | — | Not available in France | H |
| Agent status / own AISP registration | ❌ | ❌ | — | High | Disproportionate | H |

¹ BoursoBank: a notification per card payment or withdrawal (SMS fallback above 200 €). CA Aquitaine: real-time SMS;
e-mailed mini-statements free in the 2026 CGB. CMSO: balance and deferred movements, card payments not stated (§4).

## Recommendation for on-budget

1. **One transaction model, three levels of evidence**: provisional (taps, notifications, SMS), booked (EB, an
   aggregator, imports) and reference (statements). One matcher promotes provisional entries to booked ones; every
   source sits behind the `BankConnector` interface from bank-aggregation.md.
2. **v1 connectors**: EB for each owner (key held by the instance); file import for everyone; a `RecordWalletTap` App
   Intent for every iPhone user, installed from one shared shortcut on iOS 27, or added inline if spike 2 reproduces
   the "Always Allow" report. **Later**: a notification intent fed by the iOS 27 notification automation (after
   spike 8); e-mail ingest for Bankin' exports and, if CA's 2027 conditions keep it free, its mini-statements; an
   open-banking.io connector behind a flag, for owners who prefer it.
3. **Household member (partner)**: send EB the questions in spike 1. Instance-held key (option 2) if EB agrees;
   device-held key (option 3) if EB prefers it; otherwise file import plus provisional entries, or open-banking.io once
   the vendor answers.
4. **Friends and family**: they self-host (ADR 0001), so each owner registers their own EB app for their own instance.
   That is the owner case: no other household's key on anyone's server, and no contract. on-budget's part is a clear
   guide through EB's Control Panel; file import, Bankin' exports and provisional entries cover banks EB lacks. Hosting
   friends' instances or keys on the maintainer's server would reopen the third-party question (§1.1).
5. **Don't build**: FinanceKit, accessory-based notification capture (ANCS, Accessory Notifications), password
   scraping, agent or AISP registration.

## Open questions and spikes

1. **E-mail EB** (support.api@enablebanking.com) and keep the answer in writing:
   - May a household member's restricted app be run by the owner's instance (key generated there, certificate uploaded
     by the member)?
   - Is a key kept in the member's own device Keychain acceptable?
   - What is the linked-accounts cap, and the minimum invoice for one household? The Control Panel reserves
     unrestricted production for "organisations": does a micro-entrepreneur count?
2. **Tap capture, one week**: BoursoBank, CA and CMSO cards on iPhone and Watch, on iOS 26 and 27. Measure trigger rate,
   latency, declines, amount and merchant formats (French locale), repeated permission prompts. On iOS 27, also check
   that the Transaction trigger still behaves as documented for iOS 26, whether a shared shortcut's automation arrives
   on or off, and whether "Always Allow" persists for the App Intent in a shared shortcut, compared with the action
   added inline (forum 850369).
3. **CA SMS spike**: enable "Alerte Temps réel de gestion"; record the SMS format; Message automation → App Intent.
   Check e-mailed mini-statements in the CGB 2027 once CA publishes it.
4. **CMSO**: export formats in web banking; a Bud'jet e-mail and push sample with and without amounts; whether card
   payments trigger any alert.
5. **open-banking.io**: may a household instance hold a member's `credentials.json`? How are scheduled syncs flagged
   under RTS art. 36(5)? Consent length per French bank? What happens on shutdown? (The FAQ already answers: no partner
   onboarding, polling only, commercialised in May 2026.)
6. **Bankin' export sample**: CSV columns and encoding, to write an importer; which domain DKIM-signs the e-mail, and
   whether the signature verifies after a forward as attachment.
7. **Ponto**: does it accept private individuals and cover the three banks?
8. **iOS 27 notification automation, one week**: BoursoBank per-payment card notifications and CMSO alerts → App Intent.
   Count runs against delivered notifications (forum 852334 reports skips), check whether it runs without asking and
   while locked, and record text formats (amount, merchant, card).

**Not verified**: EB's view of a household member's app run by the owner's instance; contract prices (EB, Bridge,
Powens, Salt Edge, Ponto); whether Salt Edge's Test limits apply to Partners; whether Watch taps trigger the iPhone
automation; the Transaction trigger's behaviour on iOS 27; whether a shared shortcut's automation arrives turned on;
whether Notification automations can run without asking; that Message automations pass the SMS text (user reports
only); CA and CMSO export formats; CMSO SMS alerts, and whether card payments trigger CMSO alerts; CA's 2027 terms for
e-mailed alerts (no CGB 2027 yet); bank T&C clauses on forwarding alerts (only tariff brochures and help pages read);
FinanceKit plans beyond Apple's public pages; open-banking.io's claims about rate limits and consent length; the PSR's
re-authentication rules.

## Sources

**Project**: [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md) · [PRODUCT.md](../../PRODUCT.md)

**Enable Banking (primary)**: [Terms of Service](https://enablebanking.com/terms) ·
[Linked accounts](https://enablebanking.com/docs/api/linked-accounts/) · [FAQ](https://enablebanking.com/docs/faq/) ·
[API reference](https://enablebanking.com/docs/api/reference/) ·
[Control Panel bundle (activation dialog, 2026-11-07)](https://enablebanking.com/js/app.83127064.js)

**Apple (primary)**: [FinanceKit](https://developer.apple.com/financekit/) ·
[FinanceKit docs](https://developer.apple.com/documentation/financekit) ·
[FinanceKit updates (stale)](https://developer.apple.com/documentation/updates/financekit) ·
[BackgroundDeliveryExtension](https://developer.apple.com/documentation/financekit/backgrounddeliveryextension) ·
[Wallet connected accounts EN](https://support.apple.com/en-us/111117) / [FR](https://support.apple.com/fr-fr/111117) ·
[Apple Pay transaction history EN](https://support.apple.com/en-us/104954) / [FR](https://support.apple.com/fr-fr/104954) ·
Shortcuts guide, iOS 26 (9.0): [Transaction trigger](https://support.apple.com/en-gb/guide/shortcuts/apd65c67538a/ios) ·
[Run without asking](https://support.apple.com/en-gb/guide/shortcuts/apd602971e63/ios) ·
[Intro to personal automation](https://support.apple.com/en-gb/guide/shortcuts/apd690170742/ios) ·
[Communication triggers](https://support.apple.com/en-gb/guide/shortcuts/communication-triggers-apdd711f9dff/9.0/ios/26) ·
Shortcuts guide, iOS 27 (10.0): [Intro to automation](https://support.apple.com/guide/shortcuts/intro-to-automation-shortcuts-apd690170742/10.0/ios/27) ·
[Add automations](https://support.apple.com/guide/shortcuts/add-automations-apdfbdbd7123/10.0/ios/27) ·
[Sync shortcuts](https://support.apple.com/guide/shortcuts/sync-shortcuts-apdb3a4240b0/10.0/ios/27) ·
[What's new in Shortcuts 27](https://support.apple.com/en-us/149045) ·
[About iOS 27 updates](https://support.apple.com/en-us/149076) ·
[Security releases (dates)](https://support.apple.com/en-us/100100) ·
[WWDC26 session 310](https://developer.apple.com/videos/play/wwdc2026/310/) ·
[App Intents](https://developer.apple.com/documentation/appintents) ·
[CryptoKit SecureEnclave](https://developer.apple.com/documentation/cryptokit/secureenclave) ·
[Secure Enclave keys](https://developer.apple.com/documentation/security/protecting-keys-with-the-secure-enclave) ·
[BGTaskRequest.earliestBeginDate](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate) ·
[UNUserNotificationCenter](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter) ·
[Accessory Notifications](https://developer.apple.com/documentation/accessorynotifications) ·
[ANCS specification](https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Introduction/Introduction.html)

**Banks (primary)**: [BoursoBank card notifications](https://www.boursobank.com/aide-en-ligne/carte-bancaire/utilisation-quotidienne-de-ma-carte/notifications-et-suivi-des-operations/question/comment-mettre-en-place-des-notifications-sur-les-paiements-et-retraits-effectues-par-carte-219) ·
[BoursoBank services](https://www.boursobank.com/banque/services-bancaires) ·
[BoursoBank Apple Pay help](https://www.boursobank.com/aide-en-ligne/carte-bancaire/paiement-mobile) ·
[CA Aquitaine tariffs page](https://www.credit-agricole.fr/ca-aquitaine/particulier/informations/tarifs.html) ·
[CA Aquitaine CGB 2026 (upd. 2026-06-15)](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/BROCHURES_INTEGRALES_CGB_DETAILLEES_2026_Web_45-06-2026.pdf) ·
[CA Aquitaine brochure 2026](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/PlaquetteConditionsTarifairesParticuliers_2026.pdf) ·
[CA Aquitaine brochure 2027](https://www.credit-agricole.fr/content/dam/assetsca/cr833/npc/documents/pdf/0275_76125_PlaquetteConditionsTarifairesParticuliers_2027_Web.pdf) ·
[CA Aquitaine Apple Pay](https://www.credit-agricole.fr/ca-aquitaine/particulier/compte/paiement-mobile/apple-pay.html) ·
[CMSO Bud'jet alerts](https://www.cmso.com/reseau-bancaire-cooperatif/web/comptes-et-cartes/piloter-vos-comptes/gestion-de-budget-en-ligne) ·
[CMSO app](https://www.cmso.com/reseau-bancaire-cooperatif/web/comptes-et-cartes/piloter-vos-comptes/application-mobile) ·
[CMSO Apple Pay](https://www.cmso.com/reseau-bancaire-cooperatif/web/comptes-et-cartes/payer-autrement/apple-pay) ·
[CMSO 2026 tariff extract (MoneyVox archive)](https://www.moneyvox.fr/tarif-bancaire/credit-mutuel-du-sud-ouest/pdf/tarifs-2026.pdf)

**Apps and resellers (primary, vendor pages)**: [Bankin' CGU](https://bankin.com/fr/cgu.html) ·
[Bankin' export](https://support.bankin.com/hc/fr/articles/360007523551-Exporter-votre-historique-bancaire) ·
[Bankin' Plus](https://support.bankin.com/hc/fr/articles/360006559578-Pr%C3%A9sentation-de-Bankin-Plus) ·
[Linxo export](https://support.linxo.com/hc/fr/articles/115003171125-Comment-exporter-mes-donn%C3%A9es-de-Linxo) ·
[Linxo export metadata (Zendesk API)](https://support.linxo.com/api/v2/help_center/fr/articles/115003171125.json) ·
[Linxo legacy app](https://linxo.com/linxo-app-historique/) · [Linxo CGU](https://linxo.com/cgu-linxo/) ·
[Finary sync](https://help.finary.com/fr/articles/13462818-comprendre-les-problemes-de-synchronisation) ·
[open-banking.io](https://open-banking.io/en) · [terms](https://open-banking.io/en/terms) ·
[developers](https://open-banking.io/en/developers) · [FAQ](https://open-banking.io/en/faq) ·
[banks](https://open-banking.io/en/banks) · [self-hosters](https://open-banking.io/en/for-self-hosters) ·
[Lunch Flow](https://www.lunchflow.app/) · [BankSync compliance](https://banksync.io/compliance) ·
[BankSync pricing](https://banksync.io/pricing) · [BankSync terms](https://banksync.io/terms) ·
[YNAB API](https://api.ynab.com/)

**Licensed providers (primary)**: [Powens Startup Program](https://www.powens.com/startup-program/) ·
[Salt Edge docs v6](https://docs.saltedge.com/v6/) ·
[Salt Edge Partner Program](https://www.saltedge.com/products/account_information/partner_program) ·
[Ponto pricing](https://myponto.com/en/pricing/customer-paying-model/) · [Ponto company](https://myponto.com/en/company/)

**Regulation (primary)**: [PSD2 (recital 28, arts. 4, 5, 19, 20, 33, 67)](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32015L2366) ·
[RTS 2018/389](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32018R0389) ·
[RTS amendment 2022/2360](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32022R2360) ·
[OEIL PSR 2023/0210(COD)](https://oeil.secure.europarl.europa.eu/oeil/en/procedure-file?reference=2023/0210(COD)) ·
[Council 8535/26 ADD 1 (PSD3)](https://data.consilium.europa.eu/doc/document/ST-8535-2026-ADD-1/en/pdf)

**Standards (primary)**: [RFC 7960 (DMARC and indirect mail flows)](https://www.rfc-editor.org/rfc/rfc7960) ·
[RFC 6376 (DKIM)](https://www.rfc-editor.org/rfc/rfc6376)

**Secondary**: [Actual: Enable Banking](https://actualbudget.org/docs/advanced/bank-sync/enable-banking/) ·
[Apple Developer Forums 765516](https://developer.apple.com/forums/thread/765516) ·
[Apple Developer Forums 850369](https://developer.apple.com/forums/thread/850369) ·
[Apple Developer Forums 852334](https://developer.apple.com/forums/thread/852334) ·
[Apple Community 256252606](https://discussions.apple.com/thread/256252606) ·
[iDropNews on the Transaction trigger](https://www.idropnews.com/ios-17/this-new-ios-17-shortcut-will-help-you-keep-track-of-your-spending/197589/) ·
[ActualTap](https://github.com/MattFaz/actualtap) ·
[Arthur Cox: PSD3/PSR final texts](https://www.arthurcox.com/insights/psd3-and-psr-final-compromise-texts-published/)
