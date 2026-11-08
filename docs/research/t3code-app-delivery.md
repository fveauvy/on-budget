# How T3 Code builds and ships its desktop and mobile apps

> Research note, 2026-11-08, for [#9](https://github.com/fveauvy/shoestring/issues/9) on map [#6](https://github.com/fveauvy/shoestring/issues/6). It is background: `PRODUCT.md`, the ADRs, `SPEC.md`, `docs/factory.md` and `docs/architecture.md` win. T3 Code was read at commit [`6417151`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a) (2026-11-08) from a shallow clone, never installed or run. Versions come from npm dist-tags and GitHub releases read the same day. **H**, **M** and **L** mark confidence; "judgment" marks my own assessment.

## Question

How does T3 Code build, sign, release and update its Electron desktop app and its Expo mobile app? What can Shoestring copy, given a tailnet-only self-hosted server, passkeys on the instance ([ADR 0011](../adr/0011-passkeys-on-the-instance-sign-in-page.md)), the TestFlight internal group only ([ADR 0019](../adr/0019-native-apps-through-the-testflight-internal-group-only.md)) and the macOS desktop app first?

## Answer

1. **Desktop: copy T3 Code's macOS pipeline, cut down to one target.** On each `v*` tag, one job on a GitHub-hosted macOS runner does the following:
   - builds a `dmg` and a `zip` with electron-builder 26.18.1;
   - signs them with a Developer ID Application certificate;
   - notarizes them with an App Store Connect team key;
   - opens the packaged app as a launch check;
   - attaches the files and `latest-mac.yml` to the draft GitHub Release, which goes public once every asset is there.

   It costs nothing beyond the $99 Apple membership. **H** on the mechanics, judgment on the cut.
2. **Updates: electron-updater 6.8.12 from GitHub Releases, confirmed by the member, gated by the instance.**
   - Copy T3 Code's updater behavior: no automatic download or install, a check at start-up and on an interval, a restart only on request.
   - Copy its builds without a feed for anything that isn't a release, and its mock update server.
   - Add what T3 Code doesn't need: the app offers only the version the member's instance runs. ADR 0018 ships the server first, and each instance deploys on its owner's schedule.
   - electron-updater's generic provider, pointed at that tag's release-download URL, should do this. Spike D1 confirms it. **M**
3. **Passkeys: don't copy T3 Code's.** Each of its paths needs one of two things:
   - a public `apple-app-site-association` file for the relying party's domain, through Associated Domains. Clerk's passkeys and Electron 45's new `platformPasskeys` work this way;
   - Apple's browser passkey entitlement, which `electron-webauthn` relies on. Apple grants it only to apps that act as the user's web browser.

   Neither works for a tailnet-only instance, or for one build that serves every self-hoster.
   - Sign in instead as T3 Code's own OAuth leg and ADR 0011's native apps do: the system browser opens the instance's sign-in page, with PKCE and a `shoestring://` or loopback callback.
   - Keep the tokens with Electron's asynchronous `safeStorage`.
   - This build needs no provisioning profile.

   **H** on the constraints, judgment on the choice.
4. **iPhone: copy CNG, EAS Build and EAS Submit if #10 confirms the watch app and the extensions fit.** Skip EAS Update and the fingerprint workflows.
   - T3 Code regenerates `ios/` on every build, adds its extensions through config plugins and keeps build numbers on Expo's servers.
   - On every merge, it either publishes an over-the-air update or, when the native fingerprint changed, builds and submits to TestFlight.
   - ADR 0004's no-OTA rule still holds: Expo signs updates end to end only on the $199-a-month Production plan. Without OTA, the fingerprint workflows have nothing to protect.
   - Shoestring builds once per tag and adds the build to the internal group after the deploy (ADR 0019).

   **H** on T3 Code, **M** on the fit.
5. **EAS's Free plan covers a build per tag, within limits Shoestring must measure.**
   - It gives 15 iOS builds a month, a low-priority queue (90+ minute waits at peak), a 45-minute timeout and one macOS job at a time.
   - Starter ($19 a month) moves builds to the high-priority queue and raises the timeout to 2 hours.
   - EAS's newest image carries Xcode 27.1 RC; `factory.md` pins 27.2.

   **H** on the plan; Shoestring's build time is unverified (#10).
6. **Releases: keep release-please and one version.** T3 Code's nightly-then-promote train, model-written notes and version write-back serve a product with public channels; Shoestring has tags without channels (`factory.md`). Copy three things:
   - a release stays a draft until every asset is attached;
   - the release carries a `SHA256SUMS` file;
   - a build without a feed carries no updater metadata.

   **H** on T3 Code, judgment on the fit.

## 1. Versions on 2026-11-08

| Tool | Newest stable or RC | T3 Code | Note |
|---|---|---|---|
| Electron | 45.2.3 (2026-11-04) | 44.10.1 | 46 is an alpha. 45 ships Chromium 156 and Node 24.22.0, and supports macOS Ventura and later |
| electron-builder | 26.18.1 (2026-10-26) | 26.16.1 | npm's `latest` tag still names 26.18.0; 26.18.1 is a full GitHub release under npm's `v26` tag. 27 is an alpha |
| electron-updater | 6.8.12 (2026-10-26) | `^6.8.9` | The same split: `latest` names 6.8.11. 7 is an alpha |
| `@electron/notarize`, `@electron/osx-sign`, `@electron/fuses` | 3.1.2, 2.7.2, 2.1.3 | Through electron-builder; `osx-sign` also called directly | — |
| `electron-webauthn` | 1.4.0 | 1.3.1 | — |
| Expo SDK 58 (`expo`) | 58.0.23 | 58.0.14 | `expo-updates` 58.0.21, `expo-widgets` 58.0.27, `expo-sharing` 58.0.22, `@expo/fingerprint` 0.21.9 |
| React Native | 0.88.1 | 0.88.0 | No Expo SDK supports 0.89.0-rc.0 yet |
| `eas-cli` | 27.5.0 (2026-11-05) | `latest` for production, 21.8.0 for previews | The preview pin works around an `expo-github-action` bug |
| `expo/expo-github-action` | 9.0.2 | v8, plus `continuous-deploy-fingerprint` at `main` | — |
| EAS iOS image | `macos-tahoe-26.6-xcode-27.1` (`latest`, `sdk-58`): Xcode 27.1 RC, Node 22.23.3, pnpm 11.28.2 | The same | No Xcode 27.2 image yet |

## 2. Desktop

### 2.1 Build

- The release workflow builds the platform-independent JavaScript once: the server bundle, the web client and the Electron main process. Each platform job only packages that artifact ([release.yml L508–L623](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L508-L623)).
- [`scripts/build-desktop-artifact.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L2766-L2974) prepares the build:
  - it stages an app directory and writes a `package.json` whose `build` key holds the generated electron-builder configuration;
  - it installs production dependencies;
  - it runs `electron-builder --projectDir <stage> --mac --arm64 --publish never`, and leaves publishing to the workflow ([L4018–L4042](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L4018-L4042)).
- The macOS configuration ([L2828–L2876](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L2828-L2876), [`sign-macos.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/sign-macos.ts)):
  - targets `dmg` and `zip`; the `zip` carries Squirrel.Mac updates;
  - a `sign` hook calls `@electron/osx-sign` with batched `codesign` calls;
  - `entitlements` and `provisioningProfile` are set only for passkey-signed builds;
  - `Info.plist` gets usage strings for the local network, camera, microphone and screen capture, plus `NSAutoFillRequiresTextContentTypeForOneTimeCodeOnMac`, which stops macOS 26 from offering one-time codes in every text field;
  - the app claims the `t3code` scheme, plus `http`, `https` and web-page documents, so macOS lists it as a possible default browser.
- The hardened runtime is electron-builder's default ([`macOptions.ts`](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/packages/app-builder-lib/src/options/macOptions.ts)). The generated entitlements ([L1354–L1403](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L1354-L1403)) grant:
  - JIT, unsigned executable memory and disabled library validation;
  - the camera and the microphone;
  - the associated domains;
  - the keychain group and the browser passkey entitlement, when the profile grants them.
- Builds can stay unsigned. Without the secrets, the script sets `CSC_IDENTITY_AUTO_DISCOVERY=false` and drops empty signing variables, which electron-builder would treat as set ([L3967–L3995](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L3967-L3995)).

### 2.2 Signing and notarization in CI

- **Secrets** ([release-desktop.yml L9–L37](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release-desktop.yml#L9-L37), [release.md §2](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L572-L629)):
  - `CSC_LINK`: the base64 `.p12` of the Developer ID Application certificate and its key;
  - `CSC_KEY_PASSWORD`;
  - `APPLE_API_KEY` (the `.p8` text), `APPLE_API_KEY_ID` and `APPLE_API_ISSUER`;
  - `MACOS_PROVISIONING_PROFILE`, a base64 profile with Associated Domains;
  - the `APPLE_TEAM_ID` variable.

  The release jobs get them as repository secrets through `secrets: inherit`, with no environment.
- **The job** writes the key to `$RUNNER_TEMP/AuthKey_<id>.p8` and exports that path as `APPLE_API_KEY`. It decodes the profile, checks it with `security cms` and passes `--signed` ([release-desktop.yml L599–L620](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release-desktop.yml#L599-L620)).
- **Notarization.** electron-builder 26 notarizes by itself once `APPLE_API_KEY`, `APPLE_API_KEY_ID` and `APPLE_API_ISSUER` are all set; `mac.notarize: false` only turns it off. It passes the key to `@electron/notarize`, which runs `notarytool` and takes `appleApiKey` as the absolute path of the `.p8` file. electron-builder's notarization page shows base64 key content instead; its code and the `@electron/notarize` README agree with T3 Code ([`getNotarizeOptions`](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/packages/app-builder-lib/src/mac/MacTargetHelper.ts#L273-L303), [`@electron/notarize` README](https://github.com/electron/notarize#readme), [electron-builder notarization](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/features/code-signing/notarization.md)). **H**
- **Runner.** A third-party Blacksmith macOS 26 runner with 12 vCPUs builds both architectures within a 30-minute job budget; the x64 build runs on arm64 with Rosetta ([release.yml L633–L683](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L633-L683)).
- **Launch check.** Before upload, the arm64 job mounts the DMG and opens the packaged app against an old seeded database, which the bundled server must migrate, then opens a thread. A build that can't start publishes nothing ([release-desktop.yml L694–L716](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release-desktop.yml#L694-L716)).
- **Apple's rules** ([Developer ID](https://developer.apple.com/support/developer-id/), [Notarizing macOS software](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)). **H**
  - Only the Account Holder creates Developer ID certificates, at most five Application certificates.
  - Notarization needs a Developer ID signature and the hardened runtime.
  - A Developer ID provisioning profile lasts 18 years, but an app that embeds one stops launching once it expires.
  - If the membership lapses, signed apps keep running, but no new certificate can be made.

### 2.3 Auto-update

- **Feed.** The build writes `publish: [{provider: "github", owner, repo, releaseType: "release"}]` into the app's `app-update.yml`, taking the repository from `T3CODE_DESKTOP_UPDATE_REPOSITORY` or `GITHUB_REPOSITORY`, so a fork's builds update from the fork. Nightly builds add `releaseType: "prerelease"` and `channel: "nightly"` ([L2691–L2715](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L2691-L2715), [L2811–L2826](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts#L2811-L2826)).
- **How the app finds a release** ([`GitHubProvider.ts`](https://github.com/electron-userland/electron-builder/blob/electron-updater%406.8.12/packages/electron-updater/src/providers/GitHubProvider.ts), [release.yml L898–L911](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L898-L911)):
  - electron-updater reads `github.com/<owner>/<repo>/releases.atom` and `…/releases/latest`, with no API call and no token, then downloads that release's `latest-mac.yml`;
  - the release job merges the two architectures' macOS manifests into one `latest-mac.yml`, and the updater picks the file named for its own architecture.
- **Channels.** The two channels are `latest` and `nightly`, and each user picks one. A channel switch is checked against a release floor, because version order can't tell whether a stable build knows a nightly's database migrations ([DesktopUpdates.ts L387–L400](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/updates/DesktopUpdates.ts#L387-L400)).
- **Behavior** ([DesktopUpdates.ts L58–L59](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/updates/DesktopUpdates.ts#L58-L59), [L1160–L1165](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/updates/DesktopUpdates.ts#L1160-L1165), [release.md L467–L486](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L467-L486)):
  - the app checks 15 s after start, then every 4 minutes;
  - `autoDownload` and `autoInstallOnAppQuit` are off: one click downloads, a second restarts into the update;
  - differential downloads are off when the Intel build runs on Apple silicon.
- **Managed policy.** An administrator's policy file can turn updates off, or replace the GitHub feed with a generic mirror through `setFeedURL`. A policy the app can't read turns updates off ([DesktopUpdatePolicy.ts](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/updates/DesktopUpdatePolicy.ts), [DesktopUpdates.ts L1134–L1146](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/updates/DesktopUpdates.ts#L1134-L1146)).
- **Mock update server.** `--mock-updates` points the build at a generic feed on `http://localhost:3000`. So does `T3CODE_DESKTOP_MOCK_UPDATES` at run time, except under a managed policy. [`scripts/mock-update-server.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/mock-update-server.ts) serves that feed from a `release-mock` folder, refusing paths outside it.
- **Preview builds.** They get `publish: null`, so electron-builder writes no feed, and the release job refuses any `*.yml` or `*.blockmap` on a preview release ([release.yml L881–L896](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L881-L896)).
- **Requirements** ([electron-builder auto-update](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/features/auto-update.md), [Electron `autoUpdater`](https://github.com/electron/electron/blob/v45.2.3/docs/api/auto-updater.md)). **H**
  - macOS updates need a signed app and the `zip` target.
  - A staged rollout is a `stagingPercentage` field in the manifest.

### 2.4 Passkeys in Electron

T3 Code uses three mechanisms, and Electron 45 adds a fourth ([connect-setup.md L90–L113](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/connect-setup.md#L90-L113), [release.md L619–L628](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L619-L628), [Passkeys.ts L126–L141](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/preview/Passkeys.ts#L126-L141), [PasskeyBackendMac.ts L34](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/preview/PasskeyBackendMac.ts#L34), [Electron `app.configureWebAuthn`](https://github.com/electron/electron/blob/v45.2.3/docs/api/app.md#appconfigurewebauthnoptions-macos), [browser entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.web-browser.public-key-credential), [`electron-webauthn` entitlements](https://github.com/iamEvanYT/electron-webauthn/blob/main/docs/entitlements-and-provisioning.md)):

| Mechanism | Used for | Credentials | Needs |
|---|---|---|---|
| `@clerk/electron` with `passkeys: true` | Signing in to T3 Connect. OAuth goes through the system browser and back to `t3code://app/` | Synced passkeys for the Clerk domain | Associated Domains `webcredentials:<Clerk host>`, a Developer ID provisioning profile, and Clerk's public association file listing the app |
| Electron's Touch ID authenticator, `app.configureWebAuthn({ touchID })` | The in-app browser | Bound to the Mac's Secure Enclave, kept in the app's keychain group, never synced | `keychain-access-groups`, which T3 Code adds only when its profile grants it |
| `electron-webauthn`, bindings to AuthenticationServices | The in-app browser, for any site | Synced passkeys and security keys | `com.apple.developer.web-browser.public-key-credential`. The Account Holder requests it, and Apple grants it only to apps that act as the user's web browser |
| Electron 45's `app.configureWebAuthn({ platformPasskeys: true })` | Unused, since T3 Code runs Electron 44 | Synced passkeys, through `ASAuthorizationController` | Associated Domains, an embedded profile, and an association file on the relying party's domain. There is no `localhost` exception, and `clientDataJSON` reports `https://<rpId>` as the origin |

ADR 0011 rejected native passkey sheets because each needs the build to list the domain and Apple to fetch that domain's association file. No build can list every self-hoster's domain, and a tailnet-only instance publishes nothing. For Shoestring, the four rows compare as follows:

- the first and the fourth hit that same wall;
- the third needs an app that is a web browser;
- the second works for any instance without an association file, but its passkeys stay on one Mac, so every Mac needs its own enrollment, and the build needs a provisioning profile.

The system browser needs none of this. T3 Code's Clerk sign-in already sends OAuth through the default browser and takes the callback on the app's own scheme. RFC 8252 lets a desktop app receive the code on a private-use scheme or a loopback redirect ([§7.1, §7.3](https://www.rfc-editor.org/rfc/rfc8252#section-7)). The ceremony runs on the instance's own origin with the member's synced passkey, the one origin ADR 0011 checks. T3 Code serves its renderer from a privileged custom scheme (`t3code://app`) rather than from files ([ElectronProtocol.ts L112–L129](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/electron/ElectronProtocol.ts#L112-L129)). **H** on the facts, judgment on the choice.

### 2.5 Release and preview workflows

- **`release-desktop.yml`** is a reusable workflow that `release.yml` calls once per target, each its own job with its own `needs`. There are six targets: macOS, Linux and Windows, each on arm64 and x64 ([release.yml L625–L785](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L625-L785)).
- **macOS PR previews** come in two halves, so the Developer ID certificate never shares a job with PR code ([release.md L62–L90](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L62-L90)):
  - `desktop-macos-preview.yml` runs on `pull_request` when the `preview:mac` label is added. It has no secrets and builds only the JavaScript bundle.
  - `desktop-macos-preview-publish.yml` runs on `workflow_run` from `main`. It checks that the PR is open, still labelled and at the built commit, and that its author is vouched for. It then removes the label, so each label signs exactly one commit. It checks the bundle's ZIP entries, then packages, signs and notarizes it with `main`'s code. The DMG goes to a rolling `desktop-preview` prerelease. A `pull_request_target` job only cleans up.
- **For Shoestring**, signed PR previews aren't worth their machinery: one maintainer builds for one household, `factory.md` forbids `pull_request_target`, and the tag's launch check covers packaging. judgment.

### 2.6 Local data, for #19

T3 Code's desktop app runs its bundled server as a child process of Electron with `ELECTRON_RUN_AS_NODE=1`, and that server keeps its state with `node:sqlite` ([DesktopBackendConfiguration.ts L523–L538](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/backend/DesktopBackendConfiguration.ts#L523-L538)). Because it relies on `ELECTRON_RUN_AS_NODE`, it can't turn off the `runAsNode` fuse, and it sets no fuses at all. electron-builder 26 passes an `electronFuses` block to `@electron/fuses` ([`configuration.ts`](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/packages/app-builder-lib/src/configuration.ts#L275-L279), [Electron fuses](https://github.com/electron/electron/blob/v45.2.3/docs/tutorial/fuses.md)).

## 3. Mobile

### 3.1 Native project and config plugins

- **CNG.** `apps/mobile/.gitignore` ignores `/ios` and `/android`, and the run scripts call `expo prebuild --clean` first ([.gitignore L40–L41](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/.gitignore#L40-L41), [package.json](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/package.json)).
- **Variants.** [`app.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/app.config.ts) defines three: development, preview and production. Each has its own name, scheme and bundle ID, with Associated Domains for Clerk and app links, and an iOS 18.0 deployment target.
- **Extensions** come from config plugins:
  - `expo-widgets` adds the WidgetKit extension (`<bundle>.widgets`), with an app group and push-driven updates;
  - `expo-sharing` adds the share extension (`<bundle>.sharing`);
  - twelve local plugins in `apps/mobile/plugins/` adjust Android, CocoaPods and the extensions;
  - ten local Expo modules in `apps/mobile/modules/` hold the Swift and Kotlin code.

  T3 Code has no watch app and no notification service extension, so it shows nothing about ADR 0005's watch problem (#10).
- **Personal Team builds.** With `T3CODE_IOS_PERSONAL_TEAM=1` and their own bundle ID, contributors build without app groups, extensions, push or Sign in with Apple ([README L82–L104](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/README.md#L82-L104), [plugin](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/plugins/withoutIosPersonalTeamCapabilities.cjs)). That suits ADR 0019's build-from-source path.

### 3.2 EAS Build and EAS Submit

- **[`eas.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/eas.json)** sets `cli.appVersionSource: "remote"`, so Expo's servers hold the build numbers and `production`'s `autoIncrement` raises them ([App versions](https://docs.expo.dev/build-reference/app-versions/)). Its profiles:
  - `development` and `development-simulator`: development clients;
  - `preview` and `preview:dev`: internal distribution;
  - `production`: the `production` channel, the image `macos-tahoe-26.6-xcode-27.1` and Corepack;
  - `testflight`: `production` with version 3.0.0.

  The submit profiles carry the App Store Connect app ID.
- **[`mobile-eas-production.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-eas-production.yml#L297-L373)** runs on Linux after every push to `main` that touches mobile inputs. It repeats one step for three targets: the iOS App Store build, the iOS TestFlight build and Android.
  - It computes `eas fingerprint:generate` and looks for a finished build with that hash (`eas build:list --fingerprint-hash`).
  - A match publishes an over-the-air update to the `production` channel.
  - No match runs `eas build --auto-submit --no-wait`, which uploads to TestFlight or Google Play.
  - The App Store iOS build is only ever made by hand.
- **Fingerprint guards.** The workflow computes fingerprints on Linux, because a macOS fingerprint differs from the build's. It refuses to publish from a fingerprint that hashes no Expo native modules ([L1–L18](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-eas-production.yml#L1-L18), [L254–L272](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-eas-production.yml#L254-L272)).
- **Beta and store builds.** Beta builds go to external TestFlight testers after Beta App Review. An App Store release is a manual dispatch that commits its version first ([release.md L314–L335](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L314-L335)).
- **Expo Workflows' `testflight` job** fits ADR 0019's flow ([pre-packaged jobs](https://docs.expo.dev/eas/workflows/pre-packaged-jobs/#testflight)). It takes `internal_groups`, which must have automatic distribution off, `changelog` for What to Test, and `asc_build_id`, which distributes a build that is already uploaded.

### 3.3 EAS Update and the fingerprint workflows

- **Update settings** ([app.config.ts L236–L248](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/app.config.ts#L236-L248)): Expo's update URL, `checkAutomatically: "ON_LOAD"` and `fallbackToCacheTimeout: 0`. The runtime-version policy is `fingerprint`, or `appVersion` in development.
- **[`fingerprint.config.js`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/fingerprint.config.js)** adds the app's major version to the fingerprint, so an update for the 3.x TestFlight builds never reaches 2.x App Store builds.
- **Native-change label.** [`mobile-fingerprint-check.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-fingerprint-check.yml) fingerprints a PR's merge commit and its first parent, with no token. [`mobile-fingerprint-label.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-fingerprint-label.yml), a trusted `workflow_run`, then sets a "Native Change" label. Native PRs wait and merge together just before a store build.
- **Expo's rules** ([Runtime versions](https://docs.expo.dev/eas-update/runtime-versions/), [Code signing](https://docs.expo.dev/eas-update/code-signing/)):
  - the `fingerprint` policy changes the runtime version whenever anything that may affect the native runtime changes;
  - end-to-end update code signing exists only on the Production and Enterprise plans. **H**

### 3.4 Preview builds

- A PR from the same repository with the "Mobile Continuous Deployment" label runs `expo/expo-github-action/continuous-deploy-fingerprint` at `main`, with the `preview:dev` profile. A new fingerprint builds a development client; a known one publishes an update to branch `pr-<n>`. A fork PR gets one reviewed commit per label ([mobile-eas-preview.yml L100–L113](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-eas-preview.yml#L100-L113)).
- That action's README says it "is still in development and might change without notice. It is not yet ready for use" ([README](https://github.com/expo/expo-github-action/tree/main/continuous-deploy-fingerprint)). Running it from a branch also breaks `factory.md`'s rule of pinning every action to a full SHA.
- Internal distribution uses ad hoc profiles that list registered device UDIDs ([Internal distribution](https://docs.expo.dev/build/internal-distribution/)).

### 3.5 EAS plans

| | Free | Starter | Production |
|---|---|---|---|
| Price | $0 | $19 a month, plus usage | $199 a month, plus usage |
| Builds | 15 iOS and 15 Android a month | $45 of credit | $225 of credit |
| Queue | Low priority, 90+ minutes at peak | High priority | High priority |
| Build timeout | 45 minutes | 2 hours | 2 hours |
| macOS jobs at a time | 1 | 2 | 10 |
| CI/CD Workflows | 60 minutes | Usage-based | Usage-based |
| Updates | 1,000 MAU, 100 GiB | 3,000 MAU | 50,000 MAU |
| Update code signing | No | No | Yes |

Sources: [pricing](https://expo.dev/pricing.md), [plans](https://docs.expo.dev/billing/plans/), [FAQ](https://docs.expo.dev/billing/faq/), [credential security](https://docs.expo.dev/app-signing/security/), [Submit](https://docs.expo.dev/submit/ios/). **H**

- A Free account never pays overage: its builds stop until the first of the next month.
- Expo keeps signing credentials encrypted with KMS and decrypts them only in a builder's memory. Submit uses an App Store Connect API key stored for the app.
- Expo has renamed EAS Build, Submit and Update to Expo Build, Submit and Update; the `eas` CLI and `eas.json` keep the old name.

## 4. Releases, versions and changelogs

- **Triggers** ([release.yml L1–L26](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L1-L26), [release.md L12–L61](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L12-L61)):
  - **Stable** releases normally come from a `workflow_dispatch` with `channel=stable`. It rebuilds the commit of the latest published nightly, so stable ships only what nightly users have run. A pushed `vX.Y.Z` tag builds that exact commit instead.
  - **Nightly** releases come from a schedule every 30 minutes, and publish when there are new commits and at least 6 hours have passed.
  - **Preview** builds are a manual test train from any branch: they get no feed, and a warning replaces their notes.
- **Versions.**
  - The build writes the release version into the server, desktop, web and contracts `package.json` files.
  - After a stable release, a GitHub App commits `chore(release): prepare vX.Y.Z` to `main` ([L1408–L1480](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L1408-L1480)).
  - A nightly takes the next patch plus `-nightly.<date>.<run>`.
  - The mobile app has its own line: 2.2.0 committed for the App Store, 3.0.0 for TestFlight, with build numbers on Expo's servers. No release tag builds it.
- **Changelog.** There is no `CHANGELOG.md`. A release's body has three parts:
  - a header;
  - a summary written by an OpenAI model, through `OPENAI_API_KEY`;
  - GitHub's generated notes against the previous tag of the same channel, which stand alone if the summary fails ([L957–L976](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L957-L976)).
- **Publishing.** Every release uploads as a draft and goes public only once all assets are attached, with a `SHA256SUMS` file. The npm launcher follows with provenance through trusted publishing; then the hosted web app's alias moves ([L1029–L1038](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml#L1029-L1038), [release.md L429–L466](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md#L429-L466)).

## 5. Costs

- **T3 Code**, as far as the repository shows:
  - an Apple Developer team;
  - Azure Trusted Signing for Windows;
  - Blacksmith runners;
  - EAS, with separate production and preview tokens;
  - Vercel;
  - an OpenAI key for release notes.

  Its Expo plan and its bills aren't visible.
- **Shoestring's desktop app** costs nothing beyond the Apple Developer Program's 99 USD a year, which `factory.md` already counts ([Apple](https://developer.apple.com/programs/whats-included/)):
  - Developer ID certificates and notarization come with the membership;
  - GitHub-hosted macOS runners are free on a public repository;
  - GitHub Releases serves the update feed.
- **Shoestring's iPhone app** runs on EAS Free at $0 while 15 builds a month and 45 minutes per build suffice, or on Starter at $19 a month otherwise. Signed over-the-air updates would need Production, at $199 a month.

## 6. What Shoestring copies

| Area | Copy | Adapt | Skip |
|---|---|---|---|
| Desktop build and signing | electron-builder `dmg` and `zip`; the hardened runtime; a Developer ID certificate in `CSC_LINK`; notarization with a team key written to a file; `--publish never`, then the workflow attaches to the draft release; the unsigned path when secrets are missing; the launch check | Secrets go in `factory.md`'s `release` environment, bound to `v*` tags, instead of repository secrets. One arm64 build, or a universal one, leaves one `latest-mac.yml` and nothing to merge | The six-target matrix, Windows signing, CLI archives, the npm launcher, Blacksmith |
| Desktop updates | The GitHub provider with the feed taken from `GITHUB_REPOSITORY`, so forks update from their own releases; no automatic download or install; no feed outside release builds; the mock update server | The app offers only its instance's version, for example with a generic feed at `https://github.com/<owner>/<repo>/releases/download/v<instance version>/` (spike D1) | Nightly channel, channel switching, managed-policy file |
| Desktop sign-in | The renderer on a privileged custom scheme; OAuth through the system browser with a callback on the app's scheme | PKCE on the instance's own sign-in page (ADR 0011); tokens in asynchronous `safeStorage` | Clerk, `electron-webauthn`, Associated Domains, the browser entitlement, the provisioning profile |
| Desktop hardening | — | Turn off `runAsNode`, `nodeOptions` and `nodeCliInspect`, and turn on embedded ASAR integrity, `onlyLoadAppFromAsar` and cookie encryption, through `electronFuses` (judgment) | — |
| iPhone, if #10 passes | CNG with `app.config.ts`, local config plugins and modules; build numbers on Expo's servers; `eas build --auto-submit` with an App Store Connect key; Personal Team builds without capabilities | Build on the `v*` tag only. Add the build to the internal group after the deploy, through the App Store Connect API or the `testflight` job (`asc_build_id`, `internal_groups`) | EAS Update; the fingerprint check and label; `continuous-deploy-fingerprint`; per-PR device builds; external TestFlight; the App Store |
| Releases | Draft until complete; `SHA256SUMS` | — | Nightly-to-stable promotion, model-written notes, version write-back (release-please does it) |

## 7. Consequences for the open tickets

- **#19.**
  - Build, sign, notarize and update as §2 and §6 describe; sign in through the system browser.
  - For storage, T3 Code's model, native SQLite in a Node process, is one option; the web app's SQLite WASM is the other.
  - The app lock and step-up need ADR 0011's biometry-bound key. Electron's `systemPreferences.promptTouchID()` only asks for consent ([docs](https://github.com/electron/electron/blob/v45.2.3/docs/api/system-preferences.md)).
- **#18.** CNG, EAS Build and Submit, once #10 passes; over-the-air updates stay off (ADR 0004).
- **#10.** T3 Code ships widgets and a share extension through CNG, but no watch app and no notification service extension. EAS has Xcode 27.1 RC where `factory.md` pins 27.2, and the Free plan times builds out at 45 minutes.
- **ADRs.**
  - ADR 0019 loses its "no Developer ID build" (charting decision 3).
  - Unlike a TestFlight build, a signed desktop app from GitHub Releases can sign in to any instance. ADR 0019 has each instance list the native apps it accepts, so the desktop app's OAuth client ID and callback belong in that list by default.
  - ADR 0005 changes only if #10 passes; ADR 0004's ban on over-the-air updates stands.
- **`factory.md`.**
  - `release.yml` gains a macOS desktop job.
  - The `release` environment gains `CSC_LINK`, `CSC_KEY_PASSWORD` and an App Store Connect key.
  - The Costs table gains EAS if Shoestring adopts it.

## Spikes and unverified

- **D1. Updates gated by the instance.** Do the generic provider's requests for `latest-mac.yml` and the `zip` survive GitHub's release-download redirect, cache-busting query included, and does the update install? The provider resolves files against its feed URL; that source was read, not run.
- **D2. Sign-in from the desktop app through the system browser.**
  - a custom scheme against a loopback redirect;
  - Safari's prompt to open the app;
  - passkeys in other default browsers.

  It folds into SP-11.
- **D3.** How long a notarized arm64 build takes on a GitHub-hosted runner; T3 Code budgets 30 minutes on a 12-vCPU runner.
- **Unverified:**
  - T3 Code's Expo plan;
  - whether macOS asks for Local Network permission to reach tailnet addresses (T3 Code declares `NSLocalNetworkUsageDescription` in both apps);
  - Shoestring's EAS build time with a watch app and extensions (#10).

## Sources

- **T3 Code** (commit `6417151`):
  - workflows: [`release.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release.yml), [`release-desktop.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/release-desktop.yml), [`desktop-macos-preview.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/desktop-macos-preview.yml), [`desktop-macos-preview-publish.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/desktop-macos-preview-publish.yml), [`mobile-eas-production.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-eas-production.yml), [`mobile-eas-preview.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-eas-preview.yml), [`mobile-fingerprint-check.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-fingerprint-check.yml), [`mobile-fingerprint-label.yml`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/.github/workflows/mobile-fingerprint-label.yml);
  - scripts: [`build-desktop-artifact.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/build-desktop-artifact.ts), [`mock-update-server.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/mock-update-server.ts), [`merge-update-manifests.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/scripts/merge-update-manifests.ts);
  - desktop app: [`apps/desktop/package.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/package.json), [`updates/`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/updates), [`preview/`](https://github.com/pingdotgg/t3code/tree/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/preview), [`DesktopClerk.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/desktop/src/app/DesktopClerk.ts);
  - mobile app: [`app.config.ts`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/app.config.ts), [`eas.json`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/apps/mobile/eas.json);
  - docs: [`docs/operations/release.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/release.md), [`docs/operations/connect-setup.md`](https://github.com/pingdotgg/t3code/blob/6417151c093e54dbba0e6c824c69d03983793b9a/docs/operations/connect-setup.md).
- **Electron** (v45.2.3): [`app.configureWebAuthn`](https://github.com/electron/electron/blob/v45.2.3/docs/api/app.md#appconfigurewebauthnoptions-macos), [`autoUpdater`](https://github.com/electron/electron/blob/v45.2.3/docs/api/auto-updater.md), [`safeStorage`](https://github.com/electron/electron/blob/v45.2.3/docs/api/safe-storage.md), [fuses](https://github.com/electron/electron/blob/v45.2.3/docs/tutorial/fuses.md), [platform support](https://github.com/electron/electron/blob/v45.2.3/README.md#platform-support), [45.0.0 release notes](https://github.com/electron/electron/releases/tag/v45.0.0).
- **electron-builder** (26.18.1): [auto-update](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/features/auto-update.md), [notarization](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/features/code-signing/notarization.md), [mac](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/mac.md), [publish](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/publish.md), [channels](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/website/docs/tutorials/release-using-channels.md), [`macOptions.ts`](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/packages/app-builder-lib/src/options/macOptions.ts), [`MacTargetHelper.ts`](https://github.com/electron-userland/electron-builder/blob/electron-builder%4026.18.1/packages/app-builder-lib/src/mac/MacTargetHelper.ts).
- **electron-updater** (6.8.12): [`GitHubProvider.ts`](https://github.com/electron-userland/electron-builder/blob/electron-updater%406.8.12/packages/electron-updater/src/providers/GitHubProvider.ts), [`GenericProvider.ts`](https://github.com/electron-userland/electron-builder/blob/electron-updater%406.8.12/packages/electron-updater/src/providers/GenericProvider.ts).
- **Other libraries:** [`@electron/notarize`](https://github.com/electron/notarize#readme), [`electron-webauthn`](https://github.com/iamEvanYT/electron-webauthn).
- **Apple:** [Notarizing macOS software](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution), [Developer ID](https://developer.apple.com/support/developer-id/), [browser passkey entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.web-browser.public-key-credential), [program price](https://developer.apple.com/programs/whats-included/).
- **Expo:**
  - billing: [pricing](https://expo.dev/pricing.md), [plans](https://docs.expo.dev/billing/plans/), [FAQ](https://docs.expo.dev/billing/faq/);
  - builds: [build infrastructure](https://docs.expo.dev/build-reference/infrastructure/), [app versions](https://docs.expo.dev/build-reference/app-versions/), [internal distribution](https://docs.expo.dev/build/internal-distribution/), [credential security](https://docs.expo.dev/app-signing/security/), [CNG](https://docs.expo.dev/workflow/continuous-native-generation/);
  - submission and updates: [Submit to iOS](https://docs.expo.dev/submit/ios/), [Workflows `testflight` job](https://docs.expo.dev/eas/workflows/pre-packaged-jobs/#testflight), [update code signing](https://docs.expo.dev/eas-update/code-signing/), [runtime versions](https://docs.expo.dev/eas-update/runtime-versions/);
  - actions: [`continuous-deploy-fingerprint`](https://github.com/expo/expo-github-action/tree/main/continuous-deploy-fingerprint).
- **Registries** (read 2026-11-08):
  - npm dist-tags: [`electron`](https://registry.npmjs.org/-/package/electron/dist-tags), [`electron-builder`](https://registry.npmjs.org/-/package/electron-builder/dist-tags), [`electron-updater`](https://registry.npmjs.org/-/package/electron-updater/dist-tags), [`electron-webauthn`](https://registry.npmjs.org/-/package/electron-webauthn/dist-tags), [`expo`](https://registry.npmjs.org/-/package/expo/dist-tags), [`eas-cli`](https://registry.npmjs.org/-/package/eas-cli/dist-tags);
  - GitHub releases: [Electron](https://github.com/electron/electron/releases), [electron-builder](https://github.com/electron-userland/electron-builder/releases), [`expo-github-action`](https://github.com/expo/expo-github-action/releases).
- **Standards:** [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252), OAuth 2.0 for native apps.
