---
status: accepted
---

# Passkeys on the instance's own sign-in page

Each instance is its own OAuth 2.0 authorization server, with its instance address (such as `https://app.<domain>`) as issuer, and every app signs in on its page with a passkey, started from a button, whose relying-party ID is the instance address. The web app then holds a `__Host-` session cookie (HttpOnly, Secure, SameSite=Strict) on that same origin, behind an exact `Origin` check and a required custom header. The native apps open the page in an ephemeral `ASWebAuthenticationSession` with PKCE and a custom-scheme callback, and get a bearer access token of about 15 minutes and a rotating 60-day refresh token, which stays in the app's own Keychain access group and is refreshed natively; reusing a rotated refresh token revokes the device. Face ID or Touch ID, or a passkey on the web, locks the app at cold start and after a while in the background (five minutes, `SPEC.md`'s default), and step-up asks again, through a biometry-bound Secure Enclave key in the apps, before bank connections, member changes, sharing and export. Why: a native passkey sheet works only for domains listed in the build's signed entitlement, and needs an association file that Apple fetches from the internet. No build can list every self-hoster's instance, and an instance on a private network publishes nothing; the browser sheet needs neither, so one build signs in to any instance.

## Considered Options

- Sign in with Apple is native only and needs the publisher's key; passwords aren't phishing-resistant; magic links need email and open Safari rather than the app.
- An external identity provider, brokered by the server, can come later without changing the apps.
- DPoP-bound tokens, with a Secure Enclave key signing a proof for every request (`native-apps-auth.md`'s pick): a stolen token would be useless off the device, but every API call would cross the JavaScript–native bridge for a proof, and the server would add proof and replay checks. They wait for a hardening ADR. The risk of bearer tokens, accepted meanwhile: a stolen access token works for up to 15 minutes from any device that reaches the instance, and a stolen refresh token until its reuse is detected.

## Consequences

- The instance address is permanent, since changing it orphans every passkey, so it is set before the first passkey registration. The relying-party ID is that exact host, never a parent domain, so sibling subdomains can't start ceremonies with Shoestring's passkeys.
- Passkeys are the only credential in v1. The owner bootstraps from a one-hour link that the host CLI prints, members redeem single-use 72-hour invites (links or QR codes, so no email server), and recovery goes through the owner, or, for the owner, through the host CLI.
- Shoestring runs and tests its own authorization server, built on oidc-provider and SimpleWebAuthn (ADR 0007); SP-10 and SP-11 confirm or amend it.
