# Native apps and authentication for a self-hosted product

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: DPoP; §9 RP ID options and `"storage"`; watch enrollment.

Research note, 2026-11-07, revised the same day after independent fact-checks. Versions, prices and availability are as of November 2026. "Secondary" marks third-party sources; "judgment" marks my own assessment. Confidence: **high**, **medium**, **low**.

It builds on [`apple-client-stack.md`](apple-client-stack.md), [`watchos-app.md`](watchos-app.md), [`web-client-sync.md`](web-client-sync.md), [`bank-aggregation.md`](bank-aggregation.md), [`sync-engine.md`](sync-engine.md), [`factory-ci-quality.md`](factory-ci-quality.md), [`cross-platform-ui-stack.md`](cross-platform-ui-stack.md), [`self-hosting.md`](self-hosting.md) and [`connectors-and-license.md`](connectors-and-license.md), and edits none of them. The older notes assumed one hosted service on the maintainer's domain. Here every owner runs an instance on a hostname no app build can know in advance, and [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md) makes the reference instance reachable only over Tailscale.

## Question

How do the iOS, macOS and watch apps reach a self-hoster's own server, how are they distributed, and which authentication works for any self-hosted domain? This note covers distribution channels and their costs, passkeys and the Associated Domains entitlement, OIDC and the other sign-in options, push, Face ID / Touch ID, the watch, and web sessions.

## TL;DR

1. **Every client signs in on the instance's own web page with a passkey.**
   - The on-budget server embeds a small OAuth 2.0 authorization server.
   - Native apps open its authorize page in `ASWebAuthenticationSession` and use the authorization code flow with PKCE and a private-use URI scheme callback ([RFC 8252](https://www.rfc-editor.org/rfc/rfc8252)), through the `.customScheme` Callback API; the 27.3 SDKs deprecate the older initializer (§2).
   - The page runs WebAuthn with the instance hostname as RP ID by default (§9).
   - Tokens are a short access token and a rotating refresh token, both DPoP-bound to a Secure Enclave key ([RFC 9449](https://www.rfc-editor.org/rfc/rfc9449), [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)).
   - This needs no Apple entitlement, so one build works with any instance and any distribution channel, including Tailscale-only instances. It needs a UI stack that exposes `ASWebAuthenticationSession` on iOS and macOS: Expo's WebBrowser has no macOS support (§10, spike S7).
   - **High** on the platform facts, **medium-high** on the design.
2. **Native passkey sheets don't work for arbitrary domains.**
   - Apps need `webcredentials:<RP ID>` in their signed entitlement to create or use passkeys, in native UI and in `WKWebView` alike ([Apple](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys)). A build covers only the domains it was signed for.
   - Apple's CDN must also fetch the association file from the RP ID host: "You must host the file using `https://` with a valid certificate and with no redirects" ([Apple](https://developer.apple.com/documentation/xcode/supporting-associated-domains)). Only that file must be public, not the API. [`apple-client-stack.md`](apple-client-stack.md) §3.1, §3.3 and [`web-client-sync.md`](web-client-sync.md) §5.2–5.3 keep it on a public static host while the API stays private.
   - So a Tailscale-only instance can use the sheet with its own build if that file is public, which the RP ID choice decides (§9, open question 6). Otherwise only development builds on devices in developer mode work.
   - Apple staff point other relying parties to `ASWebAuthenticationSession` or `SFSafariViewController` ([forum](https://developer.apple.com/forums/thread/723273)).
   - Nextcloud's iOS app has shipped passkey login this way, in an ephemeral session, since its 33.0.2 release, published on GitHub on 2026-02-24 ([PR 3996](https://github.com/nextcloud/ios/pull/3996), [release](https://github.com/nextcloud/ios/releases/tag/33.0.2)).
   - **High** on the platform facts.
3. **Instance prerequisite: HTTPS on a hostname, with passkeys as the only credential in v1.**
   - WebAuthn rejects IP-address origins and needs a secure context ([WebAuthn L3](https://www.w3.org/TR/webauthn-3/)). Service workers need a secure context too ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)).
   - The owner is bootstrapped from the CLI, members come in through invites, and recovery goes through the owner (as in [`self-hosting.md`](self-hosting.md) §5).
   - External OIDC (Pocket ID, Authelia, authentik, Kanidm, Keycloak) comes later, brokered by the server.
   - **Medium.**
4. **Watch: no passkey API on watchOS** ([Apple](https://developer.apple.com/documentation/authenticationservices/asauthorizationplatformpublickeycredentialprovider)). The iPhone bootstraps the watch's enrollment ([`watchos-app.md`](watchos-app.md) §2.7), which doesn't depend on the domain. **High.**
5. **Push decides who can use the apps the maintainer signs.**
   - APNs signing keys belong to the app publisher's team and "must remain private" ([Apple](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns)).
   - So an instance can push only to builds signed by its own owner, or through a relay run by the publisher. Home Assistant, Bitwarden and ntfy each run such a relay.
   - A relay is a service run for other people, which [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md) rules out.
   - Web Push needs no Apple account ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). But every push must show a notification, so it carries visible alerts and can't silently wake the web app to sync ([WebKit](https://webkit.org/blog/12945/meet-web-push/)).
   - **High.**
6. **Distribution now: the maintainer's TestFlight internal group.**
   - It costs 99 USD/year and needs no App Review.
   - Up to 100 testers, each holding an App Store Connect role, with 30 devices per tester. Builds expire after 90 days ([TestFlight](https://developer.apple.com/testflight/), [overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview)).
   - It covers iPhone, watch and Mac.
   - **High.**
7. **Later, for other self-hosters, in this order:**
   - the web app each instance serves;
   - a build-from-source guide using the self-hoster's own paid account, which brings their own APNs key;
   - a notarized Developer ID Mac app, since Mac notarization "is not App Review" ([Apple](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)).

   Free Apple accounts get neither Push nor Associated Domains ([Apple](https://developer.apple.com/help/account/reference/supported-capabilities-ios)). **Medium.**
8. **An App Store or external TestFlight app needs a deliberate decision first.**
   - Guideline 5.1.1(ix) wants "banking and financial services" apps "submitted by a legal entity … not by an individual developer" ([guidelines](https://developer.apple.com/app-store/review/guidelines/)).
   - Review applies it unevenly. A free, open-source, read-only finance app by an individual was rejected under it in 2023 (secondary, [forum](https://developer.apple.com/forums/thread/740042)). Yet individuals sell self-hosted budgeting clients for Firefly III and Actual Budget in the Finance category today (§8).
   - Review also needs demo access: a demo account on a reachable instance, which a Tailscale-only one isn't, or a built-in demo mode, which needs Apple's prior approval (§8).
   - Add a DSA self-assessment, where a non-commercial hobbyist may not be a trader, and the push decision.
   - **Medium.**
9. **Web sessions on shared domains.** `SameSite` treats sibling subdomains as the same site ([MDN](https://developer.mozilla.org/en-US/docs/Glossary/Site)), so `Origin` and custom-header CSRF checks are mandatory. `Clear-Site-Data: "cookies"` clears the whole registered domain ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data)), so don't send it. Both apply to `*.ts.net` names, since `ts.net` is a public suffix ([PSL](https://publicsuffix.org/list/public_suffix_list.dat)). **High** on the facts.

## 1. What changes from the earlier notes

| Note | Assumed | Now |
|---|---|---|
| `apple-client-stack.md` §3.1, §3.3, §4.4, §5 | Native passkeys on one maintainer RP ID; the association file on a public VPS, or on a static public site if the API goes private (§3.3); external TestFlight group for friends | Sign-in through the browser sheet on each instance's own host. ADR 0003 keeps the reference instance off the internet, so its association file needs a public static host, which the RP ID choice allows or not (§9). The external group is gone: friends host their own instances |
| `bank-aggregation.md` §4.5, `connectors-and-license.md` | Already revised: the instance's `https` callback hands off to a custom scheme | Agreed. One private-use scheme serves that hand-off and the OAuth callback (§6) |
| `web-client-sync.md` §4.3, §4.6 | The product has a registrable domain to itself | Self-hosters often share a parent domain, which changes CSRF and logout (§9) |
| `web-client-sync.md` §5.2–5.3 | The association file public and static on `<rp-id>`; the app on `app.<rp-id>` over Tailscale | Still valid for the native sheet. It makes the RP ID a parent of the instance host, which §9 weighs against the exact host |
| `watchos-app.md` §2.2 | W2 starts once the maintainer's API is public | Per instance. Tailscale-only instances, the reference one included, stay on the W1 relay through the iPhone |
| `factory-ci-quality.md` | "Friends & family" external group with a public link | Internal group only. External review raises 5.1.1(ix) (§8) |

## 2. What a signed build fixes in advance

| Feature | Bound at signing to | Works for any instance from one build? |
|---|---|---|
| Native passkey sheet (`ASAuthorizationPlatformPublicKeyCredentialProvider`) | `webcredentials:<RP ID>`; "otherwise, the request returns an error" ([Apple](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys)) | No |
| Passkeys in `WKWebView` | The same associated domain (same page) | No |
| Passkeys in `ASWebAuthenticationSession` or `SFSafariViewController` | Nothing: the page runs as the top-level site ([Apple staff](https://developer.apple.com/forums/thread/723273), [passkeys.dev](https://passkeys.dev/docs/reference/ios/)) | **Yes** |
| `ASWebAuthenticationSession` with an `https` callback | "The host must be a member of a domain associated with the app" ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/https(host:path:))) | Only on the publisher's own domain, as Element X does ([FORKING.md](https://github.com/element-hq/element-x-ios/blob/main/docs/FORKING.md)) |
| `ASWebAuthenticationSession` with a custom-scheme callback (API versions below) | Nothing. The session "ensures that only the calling app's session receives the authentication callback, even when more than one app registers the same callback URL scheme" ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession), [API](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:))) | **Yes** |
| Universal links for invites and bank returns | `applinks:<host>` ([Apple](https://developer.apple.com/documentation/xcode/supporting-associated-domains)) | No |
| Password AutoFill for a domain, iPhone and watch | Associated domain ([Apple](https://developer.apple.com/documentation/watchos-apps/authenticating-users-on-apple-watch)) | No |
| APNs | Team key plus the bundle ID as topic. "You can't map a connection to APNs to multiple teams" ([Apple](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns), [topics](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns)) | No, not without a relay |
| Sign in with Apple on the web | Website URLs registered in the publisher's account: up to 10 for an individual, 100 for an organization ([Apple](https://developer.apple.com/help/account/capabilities/configure-sign-in-with-apple-for-the-web/)) | No |
| Passkeys for "any relying party identifier" | `com.apple.developer.web-browser.public-key-credential`: apps that act as a web browser, macOS 13.3+, an organization's Account Holder, Apple review ([Apple](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.web-browser.public-key-credential)) | Not available to on-budget |

- **Custom-scheme API.**
  - Custom schemes have worked since iOS 12, macOS 10.15 and watchOS 6.2 through `init(url:callbackURLScheme:completionHandler:)`. The 27.3 SDKs, in beta since 2026-11-03, deprecate it: "Use initWithURL:callback:completionHandler: instead" ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/init(url:callbackurlscheme:completionhandler:)-7fpox), [releases](https://developer.apple.com/news/releases/)).
  - The replacement, `.customScheme` ([API](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:))), needs iOS 17.4, macOS 14.4 and watchOS 10.4, below the planned 26 floors (`apple-client-stack.md` §2.1, `watchos-app.md` §3.2).
  - Expo's `expo-web-browser` and Nextcloud still call the deprecated initializer for custom schemes ([Expo source](https://github.com/expo/expo/blob/main/packages/expo-web-browser/ios/WebAuthSession.swift), [Nextcloud source](https://github.com/nextcloud/ios/blob/master/iOSClient/Login/NCLoginProvider.swift)).
  - On macOS, a session that uses the Callback API runs in the default browser only if that browser declares `CallbackURLMatchingIsSupported`: "If the system doesn't find this key in the default browser app, it sends the request to Safari instead" ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback)).
- **Association file mechanics** ([Apple](https://developer.apple.com/documentation/xcode/supporting-associated-domains), [entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.associated-domains)):
  - Since iOS 14 and macOS 11, devices fetch the file through Apple's CDN, which fetches it within 24 h. Devices re-check about weekly.
  - The file must be served over `https` with a valid certificate and no redirects, and each subdomain needs its own entry and its own file. `*.` wildcards work for every service except `appclips`.
  - Only the file must be reachable from the internet, at the exact host the entitlement names; the API and the web app can stay private (§9).
  - The `?mode=developer` alternate mode bypasses the CDN, but only for development-signed apps on devices in developer mode; `managed` needs MDM.
- **Free accounts** ("Apple Developer") can't use Associated Domains, Push, Time Sensitive notifications or Sign in with Apple ([capabilities](https://developer.apple.com/help/account/reference/supported-capabilities-ios)). Their profiles "expire 7 days from issuance", on at most 3 devices and 3 apps per device ([Apple](https://developer.apple.com/support/compare-memberships/)).

## 3. Getting passkeys to work for an arbitrary domain

| Approach | Any domain | Private (Tailscale-only) instance | Cost or risk | Verdict |
|---|---|---|---|---|
| **Browser sheet**: the passkey ceremony runs on the instance page inside `ASWebAuthenticationSession` | Yes | Yes, if the device reaches the host and trusts its certificate | A system sheet. A consent alert "asking whether to proceed" ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession)), skipped in ephemeral sessions (secondary, [Auth0 FAQ](https://github.com/auth0/Auth0.swift/blob/master/FAQ.md)); a Mac's non-Safari default browser may not honor the ephemeral request (§10). Passkey AutoFill has been reported broken in the session since iOS 17.4 (secondary, [forum](https://developer.apple.com/forums/thread/763067)), so the page starts the passkey request from a button | **Pick** |
| **Per-build entitlement**: `webcredentials:<RP ID>` from untracked build config, with the association file at the RP ID host | That build's domain | Only if the file is public at the RP ID host: a static host, plus split-horizon DNS when the RP ID is the instance host (§9). Whether that fits ADR 0003 is open question 6. Otherwise only a development build on a device in developer mode | A paid Apple account for each person who builds | Optional add-on that reuses the same passkeys |
| Wildcard `webcredentials:*.<project domain>`, with instances as subdomains of it | Names under the project domain | No | The maintainer hands out names, which is a service for others | Reject |
| One RP ID on a project domain, which every instance verifies | Yes | Yes | The project domain becomes a relay point for phishing every instance. Web logins on instance origins can't use those passkeys, and clients need only support 5 related-origin labels ([WebAuthn L3 §5.11](https://www.w3.org/TR/webauthn-3/)) | Reject |
| The browser passkey entitlement | Yes | Yes | Web browsers only, macOS only, with Apple's approval | Not available |
| List known instances in one build | The listed ones only | No | A rebuild for each instance, and the domains are public in the binary | Reject |

**Confidence: high.** Passkeys also defeat fake in-app browsers ([RFC 8252 §8.7, §8.12](https://www.rfc-editor.org/rfc/rfc8252)). A `WKWebView` dressed up as a sheet can't run the instance's passkeys, because that needs the instance's associated domain (judgment).

## 4. How reference self-hosted apps do it

App Store sellers below were read on 2026-11-07.

| Project and seller | Native sign-in against a self-hosted server | Push for self-hosters |
|---|---|---|
| Home Assistant (Nabu Casa, Inc) | `/auth/authorize` and `/auth/token` on the instance. `client_id` is the app's website, which declares custom-scheme redirects with `<link rel="redirect_uri">`. PKCE S256 "always in a native app". Refresh tokens expire after 90 days without use ([auth API](https://contributors.home-assistant.io/core/internals/auth-api/)) | The project's push service, 500 per device per day. A local WebSocket channel goes first, with cloud fallback ([push](https://contributors.home-assistant.io/companion/mobile-app-api/notifications/), [limits](https://companion.home-assistant.io/docs/notifications/notification-details)) |
| Nextcloud | Login Flow v2: the app opens the login URL "in the default browser", polls for up to 20 min and gets an app password ([docs](https://docs.nextcloud.com/server/latest/developer_manual/client_apis/LoginFlow/index.html)). Passkeys through an ephemeral `ASWebAuthenticationSession` with a custom-scheme callback: merged 2026-02-21, first released in 33.0.2, published on GitHub on 2026-02-24 ([PR 3996](https://github.com/nextcloud/ios/pull/3996), [release](https://github.com/nextcloud/ios/releases/tag/33.0.2), [source](https://github.com/nextcloud/ios/blob/master/iOSClient/Login/NCLoginProvider.swift)) | Not checked |
| Immich (FUTO Holdings, Inc.) | Password, or the server's OIDC with redirect `app.immich:///oauth-callback`. An `https` "mobile redirect URI override" through `/api/oauth/mobile-redirect` serves identity providers that reject the scheme ([docs](https://docs.immich.app/administration/oauth/)). A public demo server works in the app ([README](https://github.com/immich-app/immich#demo)) | Not checked |
| Jellyfin (Swiftfin is sold under an individual's name) | Password. Quick Connect: a 6-character code on the new device, approved from a signed-in one ([docs](https://jellyfin.org/docs/general/server/quick-connect/)). Public demo at [demo.jellyfin.org](https://demo.jellyfin.org/stable) | — |
| Bitwarden and Vaultwarden (Bitwarden Inc) | The official apps take a server URL and a master password | A relay authenticated by an installation ID and key from bitwarden.com ([FAQ](https://bitwarden.com/help/hosting-faqs/)) |
| Matrix and Element X | The OAuth 2.0 API (Matrix spec v1.15+): RFC 7591 dynamic registration, with native redirects that are reverse-DNS private-use schemes, loopback or claimed `https` ([spec](https://spec.matrix.org/latest/client-server-api/)). Element X runs it in `ASWebAuthenticationSession` with an `https` callback on its own domain ([FORKING.md](https://github.com/element-hq/element-x-ios/blob/main/docs/FORKING.md)) | Not checked |
| Actual Budget (web only) | Server password, or OpenID with redirect `https://<server>/openid/callback`. Tested with Pocket ID, authentik and Keycloak, among others. The first OpenID user becomes owner ([docs](https://actualbudget.org/docs/config/oauth-auth/)) | — |
| ntfy | — | "iOS makes it impossible to implement instant push notifications without a central server", so servers forward message IDs to ntfy.sh. Without that, delivery "can take hours" ([docs](https://docs.ntfy.sh/config/)) |

**Takeaways** (judgment, **high**):
- All of them ship one generic app that takes a server URL, and none uses the native passkey APIs for domains the user supplies. A self-hosted mail server's open design record leans the same way: the browser sheet sits under "Leaning", and "only Decided items get built" (2026-10-30, secondary, [REST Mail](https://github.com/rest-mail/rest-mail-server/issues/1331)).
- With browser flows (Home Assistant, Nextcloud, Element X, Immich's OIDC), the server owns the sign-in method.
- Every project with dependable iOS push runs a central relay.
- Individuals publish self-hosted clients too: Swiftfin outside finance, and Firefly III and Actual Budget clients in the Finance category (§8).

## 5. Sign-in options compared

| Option | Any domain from one build | Private instance | Phishing-resistant | Self-hoster burden | Verdict |
|---|---|---|---|---|---|
| **A. Built-in authorization server plus passkeys, through the browser sheet** | Yes | Yes | Yes: the passkey is bound to the host | An HTTPS hostname | **Pick** |
| B. Native passkey sheet | One build per domain | Only with a public association file at the RP ID host (§9) | Yes | A paid account and a public association file | Optional add-on (§3) |
| C. External identity provider, called by the app directly | Only if it accepts custom schemes (Immich needed an override) | Yes | If the provider uses passkeys | Run the provider and register clients | Reject for v1 |
| D. External identity provider brokered by the server, which is the OIDC client with an `https` redirect on the instance | Yes | Yes | If the provider uses passkeys | An optional provider | **Later** |
| E. Passwords (+ TOTP) with device-bound tokens | Yes, even IP or HTTP with ATS exceptions | Yes | No | Password storage and rate limits | Only if the owner wants a fallback |
| F. Magic links or email codes | Without universal links, links open Safari, not the app | Yes | Partly | An SMTP server | Reject. Codes at most for recovery |
| G. Sign in with Apple | Native only, since the web needs registered URLs (§2) | — | Yes | An Apple ID, and the publisher's key for revocation | Reject |
| H. Device approval: Quick Connect style, or the [RFC 8628](https://www.rfc-editor.org/rfc/rfc8628) device grant | Yes | Yes | Partly: codes can be phished | None | Later, for a second Mac |

- **Identity providers as of November 2026:**

  | Provider | Version | License | Passkeys |
  |---|---|---|---|
  | [Pocket ID](https://github.com/pocket-id/pocket-id) | 2.19.2 | BSD-2-Clause | Passkey-only, OpenID Certified; an admin login code adds the first passkey ([docs](https://pocket-id.org/docs/)) |
  | [Authelia](https://github.com/authelia/authelia) | 4.39.28 | Apache-2.0 | Passkey login via `enable_passkey_login`, default `false`; counts as one factor ([docs](https://www.authelia.com/configuration/second-factor/webauthn/)) |
  | [authentik](https://github.com/goauthentik/authentik) | 2026.8.5 | MIT core | WebAuthn passwordless flows and passkey autofill ([docs](https://docs.goauthentik.io/add-secure-apps/flows-stages/stages/authenticator_validate/)) |
  | [Kanidm](https://github.com/kanidm/kanidm) | 1.12.0 | MPL-2.0 | Passkeys; PKCE "can not be disabled for public clients" ([docs](https://kanidm.github.io/kanidm/stable/integrations/oauth2.html)) |
  | [Keycloak](https://github.com/keycloak/keycloak/releases) | 26.8.2 | Apache-2.0 | Not checked |

- **Client impersonation.** A custom scheme doesn't prove which app is asking, so "the authorization server SHOULD NOT process authorization requests automatically without user consent or interaction" ([RFC 8252 §8.6](https://www.rfc-editor.org/rfc/rfc8252)). Require a passkey and show the device name on every authorization. Binding the code with `dpop_jkt` ([RFC 9449 §10](https://www.rfc-editor.org/rfc/rfc9449)) makes a leaked code useless with any other key.

## 6. Reaching the server

- **Finding the instance.** The member types the URL or scans the QR code on the invite page (judgment).
  - The app reads `/.well-known/oauth-authorization-server` ([RFC 8414](https://www.rfc-editor.org/rfc/rfc8414)) plus the instance's version and name, and stores the issuer per connection.
  - Several connections per app, such as a test instance, cost little (judgment).
- **TLS.**
  - ATS requires TLS 1.2+ and a chain to an anchor that ships with the OS or that the user or an administrator installed ([Apple](https://developer.apple.com/documentation/security/preventing-insecure-network-connections)). Since iOS 17 and macOS 14, ATS also blocks IP addresses by default ([Apple](https://developer.apple.com/documentation/bundleresources/information-property-list/nsapptransportsecurity/nsallowslocalnetworking)).
  - Public instances use Let's Encrypt behind a reverse proxy ([Vaultwarden wiki](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS)).
  - Tailscale-only instances can use Tailscale HTTPS for `*.ts.net` names, which puts them in public certificate-transparency logs ([Tailscale](https://tailscale.com/kb/1153/enabling-https)).
  - Such a name is tied to its tailnet. Tailscale doesn't permit changing the tailnet name "other than between your default tailnet DNS name and your randomly generated tailnet name", and renaming the machine changes the name too ([HTTPS](https://tailscale.com/kb/1153/enabling-https), [tailnet name](https://tailscale.com/kb/1217/tailnet-name)). It also shares a site with every machine in the tailnet (§9).
  - A name under the owner's own domain keeps the RP ID if the network setup changes later. Passkeys can't move to a new RP ID (judgment; `apple-client-stack.md` §3.3).
- **Private networks.**
  - LAN addresses trigger the local network alert on iOS 14+ and macOS 15+. VPN interfaces don't count as local ([TN3179](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy)).
  - Tailscale takes iOS's single VPN slot and has no watchOS app (`apple-client-stack.md` §4.4, `watchos-app.md`).
- **Bank returns.**
  - The provider redirects to the instance's `https` callback, which the member's browser reaches.
  - The server completes the consent, then bounces to the app's scheme with no secret in the URL (`bank-aggregation.md` §4.5, `connectors-and-license.md`). The `.customScheme` session catches it.
  - When the bank's own app handles SCA, the return lands in Safari, outside the session. So the app also registers the scheme in its Info.plist (judgment; spike S4).

## 7. Push

- **Only the build's own team can push to it** ([token auth](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns), [requests](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns)).
  - Team-scoped keys, and topic-specific keys (up to 200 per environment, each covering up to 400 topics), "must remain private".
  - Alert pushes use the bundle ID as topic.
  - Each request targets `/3/device/<device_token>`, so a self-hoster holding a key would reach only the devices whose tokens they have, normally those registered with their own instance. But no key can be limited to one instance, and a key "must remain private", so the publisher can't hand keys out (judgment).
- **What follows:**
  - Builds the maintainer signs get push from the maintainer's instance.
  - A self-hoster's own build gets push from their instance, with their key.
  - A maintainer build used with someone else's instance needs a relay: Home Assistant (500 per device per day), Bitwarden (an installation key), or ntfy (forwards only message IDs). A relay that carries encrypted, content-free pokes is still a service run for others (judgment).
- **Without push**:
  - On iOS, `BGAppRefreshTask` runs when the system decides: "the system doesn't guarantee launching the task at the specified date, but only that it won't begin sooner" ([Apple](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate)). So limit alerts become best-effort.
  - BackgroundTasks covers iOS, iPadOS, Mac Catalyst, tvOS and visionOS, not native macOS or watchOS (same page). The watch has its own budgeted refresh ([WKApplication](https://developer.apple.com/documentation/watchkit/wkapplication/schedulebackgroundrefresh(withpreferreddate:userinfo:scheduledcompletion:))). A native Mac app can schedule work with `NSBackgroundActivityScheduler` ([Apple](https://developer.apple.com/documentation/foundation/nsbackgroundactivityscheduler)), or hold a connection and post local notifications while it runs (judgment).
- **Web Push**: "You don't need to join the Apple Developer Program" ([Safari 16](https://webkit.org/blog/12945/meet-web-push/)). On iOS and iPadOS 16.4+ it reaches only Home Screen web apps ([WebKit](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/)). From a private instance it is outbound only, like APNs.
  - Safari requires `userVisibleOnly`: "Violations of the userVisibleOnly promise will result in a push subscription being revoked", and Web Push "is not an invitation for silent background runtime" ([Safari 16](https://webkit.org/blog/12945/meet-web-push/)). So it carries visible alerts but can't silently wake the offline-first web app to sync.

| Transport | Reaches | Needs | Use |
|---|---|---|---|
| APNs with the instance owner's key | Builds that owner signed | A paid account | The reference instance; build-from-source |
| Web Push (VAPID) | Browsers and installed web apps, with visible notifications only | Nothing from Apple | Every instance |
| Publisher relay | The publisher's builds anywhere | A public service with abuse limits | Only if the owner decides to run one |
| None: background refresh plus local notifications | Any build, best-effort: `BGAppRefreshTask` on iOS, budgeted refresh on the watch, a running app on the Mac | Nothing | Default for the publisher's builds on other instances |

**Confidence: high** on the facts, **medium** on the defaults. This matches `self-hosting.md` §7.

## 8. Distribution

| Channel | Who installs | Cost | Review | Expiry | Push from any instance | Fit |
|---|---|---|---|---|---|---|
| **The maintainer's TestFlight, internal group** | Up to 100 App Store Connect users with the Account Holder, Admin, App Manager, Developer or Marketing role; 30 devices each ([TestFlight](https://developer.apple.com/testflight/)) | 99 USD/year, "or in local currency where available" ([Apple](https://developer.apple.com/programs/whats-included/)) | None | 90 days per build ([overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview)) | Only from the maintainer's instance | **Now** |
| The maintainer's TestFlight, external group or public link | Up to 10,000 | Same | The first build of a version, against the guidelines | 90 days | Needs a relay | No |
| App Store, public or unlisted | Anyone, or anyone with the link; unlisted apps can't be "in a beta or prerelease state" ([Apple](https://developer.apple.com/support/unlisted-app-distribution/)) | The same membership, likely an organization one (below) | Every release | — | Needs a relay | Later, if the owner opts in |
| Build from source, paid account | The self-hoster's household | 99 USD/year, plus a Mac with Xcode 27 or Expo's EAS cloud builds (free tier: 15 iOS builds, low priority; [Expo](https://expo.dev/pricing)) | None, through their own internal TestFlight | 90 days | Yes, with their own key | Later; document it |
| Build from source, free account | 3 devices, 3 apps per device | Free | None | 7 days | No: no push, no associated domains | Tinkering only |
| Ad hoc | 100 devices per product family per year ([Apple](https://developer.apple.com/help/account/devices/devices-overview)) | The membership | None | — | With the signer's key | No: UDID chores |
| Mac: Developer ID with notarization | Anyone | The membership | Notarization "is not App Review" | Profiles last 18 years (`apple-client-stack.md` §5) | Needs a relay | Later, for the Mac |
| Web app served by the instance | Members the owner invites | 0 | None | — | Yes, through Web Push | **Always** |

**App Review points that matter for a self-hosted budgeting client** ([guidelines](https://developer.apple.com/app-store/review/guidelines/), updated 2026-10-16):
- **2.1(a): demo access.**
  - "Include demo account info (and turn on your back-end service!) if your app includes a login."
  - The "Before You Submit" checklist accepts "either an active demo account or fully-featured demo mode". A built-in demo mode needs "prior approval" and is meant for cases where legal or security obligations prevent a demo account.
  - Immich and Jellyfin keep public demos. A Tailscale-only instance can't serve reviewers, so on-budget needs either a disposable public instance with a test household or a built-in demo mode on generated data, like the test household in `PRODUCT.md` (judgment). Whether Apple approves a demo mode for a self-hosted client is unverified.
- **5.1.1(ix) and 3.2.1(viii): who may submit.**
  - 5.1.1(ix): apps in "banking and financial services" should be "submitted by a legal entity that provides the services, and not by an individual developer".
  - 3.2.1(viii): "money management" apps "should be submitted by the financial institution".
  - An individual's free, open-source, read-only finance app was rejected on a one-line update after a year on the store (Oct 2023, secondary, [forum](https://developer.apple.com/forums/thread/740042)).
  - Yet self-hosted budgeting clients sold under a person's name are in the Finance category today ([Abacus](https://apps.apple.com/us/app/abacus-for-firefly-iii/id1627093491), [FireFlow](https://apps.apple.com/us/app/fireflow-for-firefly-iii/id6806560719), [Actuali](https://apps.apple.com/us/app/actuali/id6764063765)):
    - Abacus for Firefly III: open source, OAuth2 to the user's own instance, listed since 2022-06.
    - FireFlow for Firefly III: iPhone, iPad and Watch, released 2026-09.
    - Actuali: an Actual Budget client with SimpleFIN bank sync, released 2026-06, updated 2026-10-19.
  - So review applies (ix) unevenly (judgment).
  - Organization accounts need a legal entity, a D-U-N-S number and a public website. Individual accounts show the person's legal name as seller ([enroll](https://developer.apple.com/programs/enroll/)). An officially recognized nonprofit can have the fee waived ([waivers](https://developer.apple.com/help/account/membership/fee-waivers/)).
- **TestFlight external builds** are reviewed against the guidelines, and 2.2 says they "should be intended for public distribution".
- **5.1.1(v): account deletion.** Account creation implies initiating account deletion in the app. A link to a web page that completes it is allowed ([Apple](https://developer.apple.com/support/offering-account-deletion-in-your-app/)).
- **4.8: login services.** It doesn't apply when the app "exclusively uses your company's own account setup and sign-in systems". A self-hoster's own identity provider isn't a "third-party or social login service" (judgment).
- **Privacy label.** "Collect" means data that "you and/or your third-party partners" can access ([Apple](https://developer.apple.com/app-store/app-privacy-details/)). Data sent to the member's own server can be declared "Data Not Collected", as Immich and Swiftfin declare (judgment, **medium**).
- **DSA** ([Apple](https://developer.apple.com/help/app-store-connect/manage-compliance-information/manage-european-union-digital-services-act-trader-requirements)).
  - EU App Store distribution needs a trader self-assessment. An individual trader publishes an address or P.O. box, a phone number and an email. TestFlight-only distribution isn't acting as a trader.
  - Apple: "if you're a hobbyist and you developed your app with no intention of commercializing it, you may not be considered a trader." A non-trader account publishes no contact information, and EU consumers are told that consumer-protection rights won't apply to contracts with the developer.
  - Apple adds that "a legal status associated with a business activity" suggests trader status, which matters if an entity is formed for 5.1.1(ix). The owner self-assesses.
- **EU alternative distribution.** The page's legend says guidelines carrying its key icon "apply to Notarization for iOS and iPadOS apps". Its "Highlight Notarization Review Guidelines Only" toggle dims every item without `data-nr` ([script](https://developer.apple.com/app-store/review/guidelines/scripts/filterNotarized.js)). 2.1(a) and the 5.1.1 heading carry the icon, and 5.1.1(ix) carries `data-nr`, so they apply to notarized iOS apps too (**high**).

**Confidence: high** on the channel facts, **medium** on how review would treat on-budget.

## 9. The web app

- **RP ID: the exact instance host by default** (judgment).
  - WebAuthn accepts the host or a registrable suffix of it ([WebAuthn L3](https://www.w3.org/TR/webauthn-3/)). With a suffix such as `example.org`, sibling apps on `*.example.org` can start ceremonies with on-budget's passkeys.
  - The server still rejects their assertions: the RP "MUST NOT accept unexpected values of origin", and by default "SHOULD NOT allow a subdomain origin" ([WebAuthn L3 §13.4.9, §13.4.8](https://www.w3.org/TR/webauthn-3/)). With an exact origin check, a shared parent exposes passkey prompts to sibling sites, not usable assertions.
  - The exact host has a cost for the native sheet (§3): the association file must be public at that very host. On a Tailscale-only instance, that takes split-horizon DNS: public DNS sends the name to a static host with the file, and the tailnet resolves it to the instance.
  - A parent name used only by on-budget avoids that. The RP ID is `<rp-id>`, a static host serves the file there, and the app lives on `app.<rp-id>` (`web-client-sync.md` §5.2). The server then needs an RP ID setting apart from `public_url`.
  - A `*.ts.net` name allows neither: the owner doesn't control `ts.net` DNS, so no static host can answer for the name. The documented public route is Tailscale Funnel, which routes internet traffic to the node ([Funnel](https://tailscale.com/kb/1223/funnel)), against ADR 0003 (judgment).
  - Choose once (`self-hosting.md`: `public_url` is "also the passkey RP ID"); see open question 6.
- **Sessions.** Keep the BFF from `web-client-sync.md` §4.3, recommended by [RFC 10017](https://www.rfc-editor.org/rfc/rfc10017): an HttpOnly `__Host-` cookie, `Secure` and `SameSite=Strict`, on the same origin.
- **Shared parent domains.**
  - A "site" is the registrable domain, so `budget.example.org` and `photos.example.org` are the same site ([MDN](https://developer.mozilla.org/en-US/docs/Glossary/Site)). `SameSite` therefore doesn't stop a compromised sibling app. The exact `Origin` allow-list and the required custom header are mandatory, not defense in depth.
  - `Clear-Site-Data: "cookies"` "affects the entire registered domain, including subdomains" ([MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data)). On logout, send `"cache", "storage"` and expire on-budget's own cookie instead. This amends `web-client-sync.md` §4.6 (judgment).
  - Tailscale names share a site too. `ts.net` is on the Public Suffix List ([PSL](https://publicsuffix.org/list/public_suffix_list.dat)), so every machine in a tailnet shares the registrable domain `<tailnet>.ts.net`. `SameSite` and `Clear-Site-Data: "cookies"` then reach every HTTPS service in the tailnet, the reference instance included if it takes a `*.ts.net` name.
- **One login page** serves both the web app and the native authorize endpoint, so the WebAuthn ceremony is implemented once (judgment). A button starts the modal `navigator.credentials.get()`; AutoFill is only an extra, since it has been reported broken inside the session (§3).

**Confidence: high** on the facts.

## 10. Watch, Face ID / Touch ID, UI stack

- **Watch.**
  - Apple's watch sign-in options are Sign in with Apple, CloudKit, and custom forms whose AutoFill depends on associated domains ([Apple](https://developer.apple.com/documentation/watchos-apps/authenticating-users-on-apple-watch)). `ASWebAuthenticationSession` exists on watchOS 6.2+, but the passkey provider doesn't.
  - Keep the enrollment from `watchos-app.md` §2.7 as a custom grant on the same token endpoint: scopes `glance:read review:write`, with DPoP from the watch's Secure Enclave key (judgment).
  - Direct watch-to-server traffic (W2) needs an instance the watch can reach. Tailscale-only instances stay on W1. **High.**
- **Face ID / Touch ID.**
  - The app lock follows `apple-client-stack.md` §3.3: a UI gate, with Keychain classes protecting the data.
  - For step-up without the browser sheet, register a second Secure Enclave key with the `biometryCurrentSet` and `privateKeyUsage` flags ([flags](https://developer.apple.com/documentation/security/secaccesscontrolcreateflags)), and require its signature for sensitive actions. Fall back to a fresh passkey in the sheet (judgment, **medium**).
- **UI stack** ([ADR 0002](../adr/0002-one-ui-codebase-native-watch.md) wants one UI codebase, framework open):

  | Stack | Browser sheet | Device key, Keychain, app lock |
  |---|---|---|
  | SwiftUI (reference) | `ASWebAuthenticationSession` | CryptoKit Secure Enclave, Keychain, LocalAuthentication |
  | Expo / React Native, SDK 58 | `openAuthSessionAsync` uses `ASWebAuthenticationSession` on iOS, through the deprecated initializer for custom schemes ([source](https://github.com/expo/expo/blob/main/packages/expo-web-browser/ios/WebAuthSession.swift)); the module lists Android, iOS and Web, no macOS ([Expo](https://docs.expo.dev/versions/latest/sdk/webbrowser/)) | A native module for Secure Enclave and DPoP (judgment) |
  | Web UI in a `WKWebView` shell | Sign-in must leave the WebView, since passkeys there need the associated domain | A native plugin; the refresh token stays out of JavaScript (RFC 10017) |
  | Desktop shells without the API | The system browser with a custom scheme or loopback redirect ([RFC 8252 §7](https://www.rfc-editor.org/rfc/rfc8252)) | The OS keychain |

  On macOS, the session opens the default browser if it supports authentication sessions, and Safari otherwise ([Apple](https://developer.apple.com/documentation/authenticationservices/authenticating-a-user-through-a-web-service)). With the Callback API the browser must also declare `CallbackURLMatchingIsSupported` (§2).
  - Chromium's current `Info.plist` declares session, ephemeral and callback-matching support ([source](https://chromium.googlesource.com/chromium/src/+/main/chrome/app/app-Info.plist)).
  - Firefox implemented it in Nightly 157 ([bug 1921535](https://bugzilla.mozilla.org/show_bug.cgi?id=1921535)). Release builds got it only in Firefox 159, tagged 2026-10-22: the 157 and 158 release tags keep the keys under `#ifdef NIGHTLY_BUILD` ([commit](https://github.com/mozilla-firefox/firefox/commit/ae8ebfd437a09b6387608059057f6bb603c7c8d4), [158](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_158_0_RELEASE/browser/app/macbuild/Contents/Info.plist.in), [159](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_159_0_RELEASE/browser/app/macbuild/Contents/Info.plist.in)).
  - Ephemeral sessions: "Safari always respects the request. In macOS, someone can choose a different default browser that might or might not respect the request" ([Apple](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/prefersephemeralwebbrowsersession)). A browser that doesn't declare ephemeral support makes the system warn the user ([Apple](https://developer.apple.com/documentation/bundleresources/information-property-list/aswebauthenticationsessionwebbrowsersupportcapabilities/ephemeralbrowsersessionissupported)). Chrome and Firefox declare it, but whether they honor it is untested (S1).

## Recommendation for on-budget

### Authentication design (any instance, any build)

1. **Authorization server.**
   - The server is its own issuer (`https://<host>`) and publishes RFC 8414 metadata.
   - It accepts a static list of native clients: the official apps' client IDs and redirect schemes, shipped in the server's public default config, plus any the owner adds for their own builds. Dynamic registration isn't needed (judgment).
   - Each scheme must be "based on a domain name under their control, expressed in reverse order" ([RFC 8252 §7.1](https://www.rfc-editor.org/rfc/rfc8252)). So the official scheme, and the bundle ID that usually mirrors it, need a domain the project controls (open question 5).
2. **Owner and members.**
   - `on-budget admin bootstrap` prints a single-use, one-hour enrollment link, and the owner registers a passkey ([`self-hosting.md`](self-hosting.md) §5.1).
   - Members redeem single-use 72-hour invites in a browser on `https://<host>/join/…` and register a passkey there.
   - Invite secrets never travel in custom-scheme URLs (judgment).
3. **Native sign-in.**
   - Create a Secure Enclave P-256 key (`privateKeyUsage`, `AfterFirstUnlockThisDeviceOnly`).
   - Open an ephemeral `ASWebAuthenticationSession` on `/oauth/authorize` with PKCE S256 ([RFC 7636](https://www.rfc-editor.org/rfc/rfc7636)), `state` and `dpop_jkt`, created with `init(url:callback:completionHandler:)` and `.customScheme`, not the initializer deprecated in 27.3 (§2).
   - On a Mac whose default browser isn't Safari, the ephemeral request may be ignored (§10). So the authorize page never reuses a browser session and asks for a passkey every time (§5); ephemerality only spares cookies and the consent alert (judgment).
   - The page requires a passkey, started from a button rather than AutoFill (§3), and names the device. The `.customScheme` callback returns the code.
   - `/oauth/token`, called with a DPoP proof, issues an access token of about 15 minutes and a rotating refresh token on a 60-day sliding window, both bound to the key.
   - Reusing a rotated refresh token revokes the device (`apple-client-stack.md` §3.3).
   - When offline, local data stays readable behind the app lock, and the outbox waits for the next sign-in.
4. **Web sign-in.** The same page and the same passkeys, then the `__Host-` cookie session, with the §9 amendments.
5. **Step-up** (bank links, members, sharing, export): the biometric Secure Enclave key in the apps, a fresh passkey on the web.
6. **Watch.** iPhone-bootstrapped enrollment as a custom grant, with narrow scopes.
7. **Recovery.**
   - The owner issues a single-use re-enrollment link for a member.
   - For the owner, `on-budget admin recover-owner` on the host: shell access is the root of trust, as with `immich-admin reset-admin-password` ([Immich](https://docs.immich.app/administration/server-commands/)).
8. **Optional, per build:** a native passkey sheet when the build's entitlement lists `webcredentials:<RP ID>` and the association file is public at that host. It uses the same RP ID and the same passkeys. For the reference instance, that needs a public static host for the file and an RP ID that allows it (§9, open question 6).
9. **Later:** external OIDC with the server as a confidential client.
   - Its redirect is `https://<host>/auth/oidc/callback`, and it maps provider identities to existing members only.
   - The native flow doesn't change.
10. **Libraries** (versions as of 2026-11-07).
    - A TypeScript backend can use [`oidc-provider`](https://github.com/panva/node-oidc-provider) 9.13.3 (MIT, OpenID Certified; it covers PKCE, DPoP, the device grant and RFC 8414) with [SimpleWebAuthn](https://github.com/MasterKale/SimpleWebAuthn) 14.1.0.
    - A Swift backend means hand-rolling this subset; [swift-webauthn](https://github.com/swift-server/swift-webauthn) is at 1.0.0-beta.2.
    - The apps call `ASWebAuthenticationSession` directly; [AppAuth-iOS](https://github.com/openid/AppAuth-iOS) 3.0.0 is optional.

### Distribution path

- **Now.**
  - The maintainer's Developer Program membership, one app record, and a TestFlight internal group for the household: the owner, plus the partner with the Developer or Marketing role limited to on-budget (`apple-client-stack.md` §5).
  - Xcode Cloud uploads at least monthly.
  - Team ID, signing assets and any instance host come from untracked config or CI secrets, never from commits (`AGENTS.md`).
  - The official bundle ID and scheme are public by design: every build ships them, and every instance's server must accept them (Recommendation 1). They go in the code once open question 5 gives them a project domain (judgment).
- **Later, in order:**
  1. The web app, for every self-hoster.
  2. A build-from-source guide.
  3. A notarized Developer ID Mac app.
  4. An App Store or external TestFlight app, only once the owner answers questions 1 and 2 below.

### Trade-offs the owner accepts

- Sign-in shows a system browser sheet, not a native sheet, and every new device sign-in asks for a passkey.
- Every instance needs HTTPS on a hostname, and there are no passwords in v1.
- on-budget builds and tests its own small authorization server: PKCE, DPoP and token rotation.
- The partner becomes an App Store Connect user with some write rights, and builds must be re-uploaded at least every 90 days.
- Other self-hosters get the web app, or build the native apps themselves (a Mac or EAS, and 99 USD/year), until the App Store question is settled.
- Maintainer-signed apps used with other instances get no remote push unless the owner runs a relay.
- Web Push brings visible alerts only, so the web app syncs while it's open, not in the background.
- The watch on a Tailscale-only instance depends on the iPhone relay.

## Open questions and spikes

**Questions for the owner**
1. Will other self-hosters ever get maintainer-signed native apps, through the App Store or external TestFlight? That decides the legal entity, the reviewers' demo access (a public instance or a demo mode) and the DSA self-assessment, which may end as non-trader for a free, non-commercial app.
2. If so, run a push relay, which is a service for others, or ship those builds without remote push?
3. Passkeys only, or a password fallback for instances without HTTPS on a hostname?
4. External OIDC in v1, or later?
5. Should the project get its own domain? It would hold the documentation and the URL-scheme namespace, and could enable an Element X-style `https` callback. RFC 8252 §7.1 bases the official scheme on a domain the publisher controls, and every server must name that client, so Recommendation 1 waits on this answer.
6. Hostname and RP ID for the reference instance, both permanent (§9):
   - a `*.ts.net` name: tied to that tailnet, a site shared with every tailnet machine, and no public association file without Funnel;
   - the exact host under the owner's domain: the native sheet then needs split-horizon DNS;
   - a parent name used only by on-budget, with the app on `app.<rp-id>`: a public static file, as `web-client-sync.md` §5.2 plans.

   Does a third-party static host for the association file fit ADR 0003? It takes Apple's CDN fetches, not the instance's traffic.

**Spikes**
- **S1 (½ day) Browser-sheet passkeys.** Run an ephemeral session with a passkey and a `.customScheme` callback on iOS 26/27 and macOS 26/27, with Safari, Chrome and Firefox 159+ as the default browser. Test against a public host and a Tailscale-only host. Try both the button and AutoFill, check whether each browser honors the ephemeral request, and record the prompts. This is `self-hosting.md`'s P4, extended.
- **S2 (1–2 days) Authorization server.** Prove PKCE, `dpop_jkt` and DPoP-bound rotating refresh tokens with a Secure Enclave ES256 key. Compare `oidc-provider` with a hand-rolled server, decided together with the backend language.
- **S3 (½ day) Per-build associated domain.** Generate the entitlements from untracked config, have the server render the association file or copy it to a static host for a Tailscale-only instance (§9), and measure the CDN pickup and `?mode=developer`.
- **S4 (½ day) Bank return.** Test the server callback and custom-scheme bounce, including the switch to the bank's own app (`self-hosting.md` P6).
- **S5 Watch.** Implement enrollment as a custom grant. Check whether watch traffic through the paired iPhone uses the iPhone's VPN.
- **S6 Push.** Test Web Push to an installed iOS web app from a private instance, and measure what alert timing background refresh alone gives over a week on iPhone, watch and Mac.
- **S7 UI-stack gate.** The chosen TypeScript stack must expose `ASWebAuthenticationSession` on iOS and macOS, the Keychain, Secure Enclave signing and LocalAuthentication. Check that its session wrapper uses the Callback API, not the initializer deprecated in 27.3, which `expo-web-browser` still calls (§2).
- **S8 (only for the App Store path)** Ask App Review, through Meet with Apple, whether 5.1.1(ix) covers a self-hosted budgeting client.

**Not verified**
- That passkeys work in an ephemeral session on every platform. Nextcloud's shipped login is partial evidence for iOS; macOS is untested (S1).
- That ephemeral sessions skip the consent alert. Auth0's SDK FAQ says so (secondary); macOS with other default browsers is untested (S1).
- Whether passkey AutoFill works in the session on iOS 26/27. The 2024 report of it breaking after iOS 17.4 records no fix (secondary).
- How App Review would treat on-budget under 5.1.1(ix): one rejection in 2023 against three individuals' budgeting clients on sale today (§8).
- Whether Apple would approve a built-in demo mode for a self-hosted client.
- Passkeys in non-Safari default browsers on macOS inside the session, and whether those browsers honor the ephemeral request.
- Since which Chrome version release builds declare callback matching.
- Whether watch traffic over a paired iPhone goes through its VPN.
- The euro price of the membership.
- The push paths of Immich, Nextcloud and Element X.

## Sources

- **Apple, associated domains and passkeys:** [Supporting associated domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains) · [Associated domains entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.associated-domains) · [Supporting passkeys](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys) · [Platform passkey provider](https://developer.apple.com/documentation/authenticationservices/asauthorizationplatformpublickeycredentialprovider) · [Browser passkey entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.web-browser.public-key-credential) · [Apple staff on WKWebView passkeys](https://developer.apple.com/forums/thread/723273) · [Authenticating users on Apple Watch](https://developer.apple.com/documentation/watchos-apps/authenticating-users-on-apple-watch) · [SIWA for the web](https://developer.apple.com/help/account/capabilities/configure-sign-in-with-apple-for-the-web/) · [SecAccessControlCreateFlags](https://developer.apple.com/documentation/security/secaccesscontrolcreateflags)
- **Apple, web authentication sessions:** [ASWebAuthenticationSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession) · [Callback](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback) · [https(host:path:)](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/https(host:path:)) · [customScheme(_:)](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/callback/customscheme(_:)) · [prefersEphemeralWebBrowserSession](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/prefersephemeralwebbrowsersession) · [Deprecated init(url:callbackURLScheme:completionHandler:)](https://developer.apple.com/documentation/authenticationservices/aswebauthenticationsession/init(url:callbackurlscheme:completionhandler:)-7fpox) · [EphemeralBrowserSessionIsSupported](https://developer.apple.com/documentation/bundleresources/information-property-list/aswebauthenticationsessionwebbrowsersupportcapabilities/ephemeralbrowsersessionissupported) · [Authenticating through a web service](https://developer.apple.com/documentation/authenticationservices/authenticating-a-user-through-a-web-service) · [Software releases](https://developer.apple.com/news/releases/)
- **Apple, push, network, background:** [APNs token auth](https://developer.apple.com/documentation/usernotifications/establishing-a-token-based-connection-to-apns) · [APNs requests](https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns) · [earliestBeginDate](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate) · [WKApplication background refresh](https://developer.apple.com/documentation/watchkit/wkapplication/schedulebackgroundrefresh(withpreferreddate:userinfo:scheduledcompletion:)) · [NSBackgroundActivityScheduler](https://developer.apple.com/documentation/foundation/nsbackgroundactivityscheduler) · [ATS](https://developer.apple.com/documentation/security/preventing-insecure-network-connections) · [NSAllowsLocalNetworking](https://developer.apple.com/documentation/bundleresources/information-property-list/nsapptransportsecurity/nsallowslocalnetworking) · [TN3179](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy)
- **Apple, distribution and review:** [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) · [Guidelines notarization filter](https://developer.apple.com/app-store/review/guidelines/scripts/filterNotarized.js) · [TestFlight](https://developer.apple.com/testflight/) · [TestFlight overview](https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview) · [Internal testers](https://developer.apple.com/help/app-store-connect/test-a-beta-version/add-internal-testers) · [Compare memberships](https://developer.apple.com/support/compare-memberships/) · [Capabilities by membership](https://developer.apple.com/help/account/reference/supported-capabilities-ios) · [Program price](https://developer.apple.com/programs/whats-included/) · [Enrollment](https://developer.apple.com/programs/enroll/) · [Fee waivers](https://developer.apple.com/help/account/membership/fee-waivers/) · [DSA trader](https://developer.apple.com/help/app-store-connect/manage-compliance-information/manage-european-union-digital-services-act-trader-requirements) · [Unlisted apps](https://developer.apple.com/support/unlisted-app-distribution/) · [Account deletion](https://developer.apple.com/support/offering-account-deletion-in-your-app/) · [App privacy details](https://developer.apple.com/app-store/app-privacy-details/) · [Mac notarization](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution) · [Devices](https://developer.apple.com/help/account/devices/devices-overview)
- **Standards:** [WebAuthn Level 3](https://www.w3.org/TR/webauthn-3/) (W3C Recommendation, 2026-08-25) · [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252) · [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636) · [RFC 8414](https://www.rfc-editor.org/rfc/rfc8414) · [RFC 8628](https://www.rfc-editor.org/rfc/rfc8628) · [RFC 9449](https://www.rfc-editor.org/rfc/rfc9449) · [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700) · [RFC 10017](https://www.rfc-editor.org/rfc/rfc10017) · [Matrix client-server API](https://spec.matrix.org/latest/client-server-api/) · [MDN Clear-Site-Data](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Clear-Site-Data) · [MDN Site](https://developer.mozilla.org/en-US/docs/Glossary/Site) · [Public Suffix List](https://publicsuffix.org/list/public_suffix_list.dat) · [MDN Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) · [WebKit: Meet Web Push](https://webkit.org/blog/12945/meet-web-push/) · [WebKit: Web Push on iOS](https://webkit.org/blog/13878/web-push-for-web-apps-on-ios-and-ipados/) · [passkeys.dev iOS reference](https://passkeys.dev/docs/reference/ios/) (W3C CG and FIDO Alliance members)
- **Reference projects:** [Home Assistant auth API](https://contributors.home-assistant.io/core/internals/auth-api/) · [HA push](https://contributors.home-assistant.io/companion/mobile-app-api/notifications/) · [HA limits](https://companion.home-assistant.io/docs/notifications/notification-details) · [Nextcloud Login Flow](https://docs.nextcloud.com/server/latest/developer_manual/client_apis/LoginFlow/index.html) · [Nextcloud iOS PR 3996](https://github.com/nextcloud/ios/pull/3996) · [Nextcloud iOS 33.0.2](https://github.com/nextcloud/ios/releases/tag/33.0.2) · [NCLoginProvider.swift](https://github.com/nextcloud/ios/blob/master/iOSClient/Login/NCLoginProvider.swift) · [Immich OAuth](https://docs.immich.app/administration/oauth/) · [Immich server commands](https://docs.immich.app/administration/server-commands/) · [Immich demo](https://github.com/immich-app/immich#demo) · [Jellyfin Quick Connect](https://jellyfin.org/docs/general/server/quick-connect/) · [Bitwarden hosting FAQ](https://bitwarden.com/help/hosting-faqs/) · [Element X FORKING.md](https://github.com/element-hq/element-x-ios/blob/main/docs/FORKING.md) · [Actual OpenID](https://actualbudget.org/docs/config/oauth-auth/) · [ntfy config](https://docs.ntfy.sh/config/) · [Vaultwarden HTTPS](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS) · [Tailscale HTTPS](https://tailscale.com/kb/1153/enabling-https) · [Tailscale tailnet name](https://tailscale.com/kb/1217/tailnet-name) · [Tailscale Funnel](https://tailscale.com/kb/1223/funnel) · [Firefox bug 1921535](https://bugzilla.mozilla.org/show_bug.cgi?id=1921535) · [Firefox ride-the-trains commit](https://github.com/mozilla-firefox/firefox/commit/ae8ebfd437a09b6387608059057f6bb603c7c8d4) · [Firefox 159 Info.plist](https://github.com/mozilla-firefox/firefox/blob/FIREFOX_159_0_RELEASE/browser/app/macbuild/Contents/Info.plist.in) · [Chromium Info.plist](https://chromium.googlesource.com/chromium/src/+/main/chrome/app/app-Info.plist) · [expo-web-browser WebAuthSession.swift](https://github.com/expo/expo/blob/main/packages/expo-web-browser/ios/WebAuthSession.swift)
- **App Store listings (read 2026-11-07):** [Home Assistant](https://apps.apple.com/us/app/home-assistant/id1099568401) · [Immich](https://apps.apple.com/us/app/immich/id1613945652) · [Swiftfin](https://apps.apple.com/us/app/swiftfin/id1604098728) · [Bitwarden](https://apps.apple.com/us/app/bitwarden-password-manager/id1137397744) · [Abacus for Firefly III](https://apps.apple.com/us/app/abacus-for-firefly-iii/id1627093491) · [FireFlow for Firefly III](https://apps.apple.com/us/app/fireflow-for-firefly-iii/id6806560719) · [Actuali](https://apps.apple.com/us/app/actuali/id6764063765)
- **Identity providers and libraries:** [Pocket ID](https://github.com/pocket-id/pocket-id) · [Pocket ID docs](https://pocket-id.org/docs/) · [Authelia](https://github.com/authelia/authelia) · [Authelia WebAuthn](https://www.authelia.com/configuration/second-factor/webauthn/) · [authentik](https://github.com/goauthentik/authentik) · [authentik passwordless](https://docs.goauthentik.io/add-secure-apps/flows-stages/stages/authenticator_validate/) · [Kanidm](https://github.com/kanidm/kanidm) · [Kanidm OAuth2](https://kanidm.github.io/kanidm/stable/integrations/oauth2.html) · [Keycloak](https://github.com/keycloak/keycloak/releases) · [oidc-provider](https://github.com/panva/node-oidc-provider) · [SimpleWebAuthn](https://github.com/MasterKale/SimpleWebAuthn) · [swift-webauthn](https://github.com/swift-server/swift-webauthn) · [AppAuth-iOS](https://github.com/openid/AppAuth-iOS) · [Expo WebBrowser](https://docs.expo.dev/versions/latest/sdk/webbrowser/) · [Expo pricing](https://expo.dev/pricing)
- **Secondary:** [Forum: open-source finance app rejected under 5.1.1(ix)](https://developer.apple.com/forums/thread/740042) (Oct 2023) · [Forum: passkey AutoFill not triggered in ASWebAuthenticationSession](https://developer.apple.com/forums/thread/763067) (2024, Apple DTS asked for a bug report) · [Auth0.swift FAQ on the alert box](https://github.com/auth0/Auth0.swift/blob/master/FAQ.md) (updated 2026-03-03) · [REST Mail passkey design issue](https://github.com/rest-mail/rest-mail-server/issues/1331) (open design record from 2026-10-30; the browser sheet is "Leaning", not "Decided")
