# EAS with a watch app and iOS extensions

> **Status (2026-11-08).** Background research for ticket #10. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md) win where this note disagrees. Amended here: [`watchos-app.md`](watchos-app.md) §3.3 and [`cross-platform-ui-stack.md`](cross-platform-ui-stack.md) §4.7 say SDK 56 builds fail on the plugin's iOS 15.1 podspec ([#196](https://github.com/EvanBacon/expo-apple-targets/issues/196)); 5.0.0 already ships the 16.4 fix (§2.1).

Research note, 2026-11-08. Versions are the newest stable release that day, read from npm dist-tags and GitHub releases; nothing here recommends a beta, alpha, canary or nightly. "Secondary" marks third-party reports (issues, PR descriptions). "Judgment" marks my own assessment. Confidence: **high**, **medium**, **low**. Expo now calls EAS "Expo services" (Expo Build, Expo Submit, Expo Update), while the `eas` CLI and `eas.json` keep the name ([pricing](https://expo.dev/pricing.md)); this note says EAS.

> **Premises.** Charting decision 8 ([#7](https://github.com/fveauvy/shoestring/issues/7)) moves iPhone delivery to EAS "with a regenerated native project instead of the committed `ios/`", amending [ADR 0005](../adr/0005-swift-where-ios-needs-it-in-a-committed-ios-project.md), provided that "the watch app and the extensions fit, and the free tier is enough". [ADR 0006](../adr/0006-watch-shows-a-glance-relayed-by-the-iphone.md) sets the watchOS 26.0 floor and the native `WatchRelay` module. [ADR 0015](../adr/0015-one-message-catalog-for-every-platform.md) generates String Catalogs for the watch, the widgets and the notification service extension. [ADR 0019](../adr/0019-native-apps-through-the-testflight-internal-group-only.md) ships only to the TestFlight internal group and keeps identifiers in untracked configuration.

## Question

An Expo app is built with Continuous Native Generation (CNG) and EAS. Can it ship a SwiftUI watch app (watchOS 26 floor), a notification service extension, an App Intents extension and widgets to the TestFlight internal group? Covering: the config plugins, app groups and Keychain sharing, a WatchConnectivity module, per-target signing on EAS, Xcode 27.2 on EAS images, EAS Update and the fingerprint, the Free plan in November 2026, and Xcode Cloud after a prebuild as the fallback.

## TL;DR

1. **EAS can ship this app, but not from a regenerated native project today. Build and submit the committed `ios/` on EAS, and keep CNG behind SP-03's CNG path** (judgment). Confidence: **high** on the CNG blockers, **medium** on the EAS path until SP-03 runs it. Decision 8's first condition fails for CNG, so ADR 0005's committed project stays; only its "no EAS" consequence changes, if #18 picks EAS.
2. **One generator for the targets under CNG**, `@bacons/apple-targets` 5.0.0 (2026-07-17), and it lags: it is the SDK 55 line, it pins the alpha `@bacons/xcode`, and its SDK 57 port is an unpublished draft that puts SDK 58 out of scope. The watch's build number doesn't follow the app's, which fails the upload (error 90379), and the watch target can't link Swift packages. Expo's own `expo-widgets` covers iOS widgets only; `expo-app-intents` is alpha and builds into the app target, not an extension (§2).
3. **Per-target signing works on EAS.** Under CNG, the targets come from `extra.eas.build.experimental.ios.appExtensions`, which the docs label experimental. In a committed project, EAS finds every signable target by walking target dependencies. Capability sync enables App Groups, but registering a group needs an Apple ID session. EAS syncs nothing for Keychain sharing, whose groups carry the team ID prefix (§3, §5).
4. **No Xcode 27.2 on EAS yet.** The newest image is `macos-tahoe-26.6-xcode-27.1` (`latest`, `sdk-58`), build 27A9275, which Apple shipped as Xcode 27.1 on 2026-10-22. Xcode 27.2 (27B52) came out on 2026-10-30. Apple accepts uploads from Xcode 26 or later, and Xcode 27.1 deploys to watchOS 9 through 27, so the watchOS 26 floor builds (§6).
5. **The fingerprint misses `targets/`.** SDK 58's default `balanced` preset ignores a generated `ios/` and hashes no Swift under `targets/`. A watch-only change then keeps the runtime version, and a fingerprint workflow ships an update that can't carry it. Add `targets/` to `extraSources`. A committed `ios/` is hashed whole. Updates never carry Swift (§7).
6. **The Free plan fits the cadence.** It gives 15 iOS builds a month, a low-priority queue, a 45-minute build timeout, one macOS job at a time, 60 Workflows minutes, Submit, and updates to 1,000 MAUs, with no overage. The timeout and the queue are the risks. Starter costs $19 a month (§8).
7. **Fallbacks.** `eas build --local` runs on GitHub-hosted macOS runners: free on a public repository, 6-hour jobs, the runner's Xcode. Xcode Cloud needs a committed project, because Apple requires one that is "continuously present" (§9).

## 1. What has to ship

| Piece | Target kind | Under CNG | In a committed `ios/` |
|---|---|---|---|
| Watch app, SwiftUI, watchOS 26.0 | watchOS app in the iOS app's `Watch/` | `@bacons/apple-targets` type `watch` | Xcode target |
| Complications | WidgetKit extension inside the watch app | type `watch-widget` | Xcode target |
| Notification service extension | App extension | type `notification-service`, or the community `expo-nse-plugin` 2.2.1 ([npm](https://www.npmjs.com/package/expo-nse-plugin)) | Xcode target |
| App Intents extension (Next) | ExtensionKit extension | type `app-intent` | Xcode target |
| iOS widgets (Next) | WidgetKit extension | type `widget` in Swift, or `expo-widgets` | Xcode target |
| `WatchRelay`, `Security` | Code in the app target | Local Expo modules, autolinked by prebuild (§4) | Same |
| `WatchBridge`, `DesignSystem`, `Strings` | Swift packages linked into targets | No published support; sources in `targets/_shared` (§2.1) | Local package references |

The target types come from the plugin's README ([README](https://github.com/EvanBacon/expo-apple-targets)).

## 2. Generating the targets under CNG

### 2.1 `@bacons/apple-targets`

- **Release.** 5.0.0 (2026-07-17) is still `latest` and the only stable line ([npm](https://www.npmjs.com/package/@bacons/apple-targets)). The repository has no GitHub releases, and `main` is the 5.0.0 publish commit `8aa4c95`.
- **Dependencies.** It needs `@expo/prebuild-config` ~55.0.6 and exactly `@bacons/xcode` 1.0.0-alpha.32, with peer `expo` >=52. `@bacons/xcode` has never shipped a stable release; its latest is 1.0.0-alpha.33 ([npm](https://www.npmjs.com/package/@bacons/xcode)). The README calls it "mostly typed, very untested, and possibly full of bugs". The mods use the app's own `expo/config-plugins`; `@expo/prebuild-config` serves only asset-catalog helpers (5.0.0 source).
- **SDK line.** 5.0.0 includes the iOS 16.4 podspec bump that SDK 56 needs ([#195](https://github.com/EvanBacon/expo-apple-targets/pull/195)). [#196](https://github.com/EvanBacon/expo-apple-targets/issues/196) stays open only as a pointer to the SDK 57 port.
- **SDK 57 port.** It is draft [#210](https://github.com/EvanBacon/expo-apple-targets/pull/210), last updated 2026-11-07 (secondary: the PR description). It would publish 6.0.0 with peer `expo` >=57, and says "SDK 58 is out of scope". Publishing waits on an npm token. Its CI builds unsigned (`CODE_SIGNING_ALLOWED=NO`), and the watch "has not been tested on a physical device". Expo's latest is 58.0.23 ([npm](https://www.npmjs.com/package/expo)); no issue reports 5.0.0 working or failing on SDK 58.
- **Untested by its author.** The README says of the target types: "I haven't tested all of these and they may not work". On signing: "theoretically handled entirely by EAS Build … I've only tested this end-to-end with my Pillar Valley Widget."
- **Watch build settings** ([`configuration-list.ts`](https://github.com/EvanBacon/expo-apple-targets/blob/8aa4c95/packages/apple-targets/src/configuration-list.ts#L351-L403), [`with-widget.ts`](https://github.com/EvanBacon/expo-apple-targets/blob/8aa4c95/packages/apple-targets/src/with-widget.ts#L384)):
  - `WATCHOS_DEPLOYMENT_TARGET` comes from `deploymentTarget`, which defaults to 11.0, so set `"26.0"`;
  - `MARKETING_VERSION` is hard-coded to `"1.0"`;
  - `CURRENT_PROJECT_VERSION` is `ios.buildNumber || 1`;
  - only App Clips read `EAS_BUILD_IOS_BUILD_NUMBER`.
- **Versions break the upload.** Open since 2025-10-27, [#147](https://github.com/EvanBacon/expo-apple-targets/issues/147): "The CFBundleVersion value '1' of watch application … does not match the CFBundleVersion value '859' of its containing iOS application (90379)". The fix, [#148](https://github.com/EvanBacon/expo-apple-targets/pull/148), is unmerged. A comment on #147 reports the matching 90473 mismatch on a widget's `CFBundleShortVersionString`. The `app-intent` target also hard-codes iOS 17.0 and build `1`.
- **No Swift packages.** [#171](https://github.com/EvanBacon/expo-apple-targets/issues/171), open since 2026-02-12: the watch target has no Frameworks build phase, so SPM dependencies don't link. The target config has no package option. [#122](https://github.com/EvanBacon/expo-apple-targets/pull/122) closed unmerged in favour of a separate `@bacons/spm` ([#177](https://github.com/EvanBacon/expo-apple-targets/pull/177)), which is still open and isn't on npm. The workaround is a `targets/_shared` directory, whose files the plugin links "to all targets in your project", the iOS app included (README; `with-xcode-changes.ts`).
- **App groups.** The plugin mirrors `ios.entitlements` app groups only for some types (widget, watch-widget, share and a few more). `watch`, `notification-service` and `app-intent` need an explicit `entitlements` block ([`target.ts`](https://github.com/EvanBacon/expo-apple-targets/blob/8aa4c95/packages/apple-targets/src/target.ts#L29-L140)).
- **Other open issues.** A non-clean prebuild crashes ([#201](https://github.com/EvanBacon/expo-apple-targets/issues/201)) or duplicates `app-intent` targets ([#202](https://github.com/EvanBacon/expo-apple-targets/issues/202)). EAS prebuilds on a fresh VM, so only local runs need `--clean`. An Icon Composer `.icon` fails for the watch target ([#207](https://github.com/EvanBacon/expo-apple-targets/issues/207)).

### 2.2 Expo's packages and other routes

- **`expo-widgets` 58.0.27** ([npm](https://www.npmjs.com/package/expo-widgets)) builds iOS and Android widgets and Live Activities. Its layouts run "in an isolated runtime, with no React hooks, app state, or asynchronous work". It defaults its app group to `group.<bundle id>` and doesn't cover the watch ([docs](https://docs.expo.dev/versions/latest/sdk/widgets/)).
  - Open: with Xcode 27.2, a project that combines `expo-widgets` and the `expo-sharing` extension won't open ([#52987](https://github.com/expo/expo/issues/52987); fixes [#52467](https://github.com/expo/expo/pull/52467) and [#53083](https://github.com/expo/expo/pull/53083) are open).
  - Judgment: Shoestring's iOS widgets show the glance that `WatchRelay` writes natively, and a JS-fed widget refreshes only when JavaScript runs. A SwiftUI widget that reads the app group can share `AccessoryGlance` with the watch.
- **`expo-app-intents` 0.5.16** is alpha ([SDK 58](https://expo.dev/changelog/sdk-58)). Its intents "compile into the app target", so it builds no extension ([docs](https://docs.expo.dev/versions/latest/sdk/app-intents/)).
- **`patch-project` 58.0.20** saves hand edits of `ios/` as patches that prebuild re-applies. Its docs warn that patching the `.pbxproj` "for example, if you're adding a widget extension … may not work reliably" ([docs](https://docs.expo.dev/config-plugins/patch-project/)).
- **No Expo guide** covers Apple Watch apps: Expo's docs mention watchOS only in the CLI reference and the widget and app-integrity pages (search of `expo/expo` docs, 2026-11-08).
- **An in-repo plugin**, written or forked from the MIT plugin, is the remaining route (judgment). It means owning the version stamping, SPM linking and target wiring at every SDK and Xcode release.

## 3. App groups and Keychain sharing

- **Where entitlements live under CNG.** The app target takes them from `ios.entitlements` in the app config. Each extra target takes them from `entitlements` in its `expo-target.config.js`; the plugin writes those to `ios/.targets/<name>/generated.entitlements` (README).
- **Capability sync.** On each `eas build`, EAS enables App Groups, Push Notifications, Time Sensitive Notifications and other listed capabilities on the bundle ID ([capabilities](https://docs.expo.dev/build-reference/ios-capabilities/)). Registering and assigning App Groups needs "Apple cookies authentication (running locally)", because the App Store Connect API can't do it. So the maintainer runs the first build after each new group. `EXPO_NO_CAPABILITY_SYNC=1` turns sync off.
- **Sync fix.** eas-cli 27.3.0 (2026-11-03) stopped capability sync from unlinking an existing App Group when it links another ([#4445](https://github.com/expo/eas-cli/issues/4445)). The latest is 27.5.0 (2026-11-05, [npm](https://www.npmjs.com/package/eas-cli)).
- **Keychain sharing.** Xcode prefixes keychain groups with the team ID, and "you can use app group names as keychain access group names, without adding them to the Keychain access groups entitlement" ([Apple](https://developer.apple.com/documentation/security/sharing-access-to-keychain-items-among-a-collection-of-apps)). Apple lists "Keychain sharing" among iOS capabilities ([Apple](https://developer.apple.com/help/account/reference/supported-capabilities-ios)), but EAS's sync list has no entry for it. Judgment: a team-prefixed group needs no portal step, so the NSE can read the device key from a shared keychain group or the app group; SP-03 checks the signed entitlements.

## 4. `WatchRelay` as a native module

- **Local module.** `npx create-expo-module --local` creates `modules/<name>`, which prebuild autolinks ([docs](https://docs.expo.dev/modules/get-started/)). It lives outside `ios/`, so `--clean` keeps it.
- **App delegate hooks.** A Swift class extending `ExpoAppDelegateSubscriber`, listed under `apple.appDelegateSubscribers` in `expo-module.config.json`, receives the app delegate calls ([docs](https://docs.expo.dev/modules/appdelegate-subscribers/)). That is where `WCSession` activates at launch.
- **Scene life cycle.** SDK 58 moves to it: prebuild generates `SceneDelegate.swift`, and "Expo forwards URL, user activity, Handoff, life cycle, and quick action events to ExpoAppDelegate" ([SDK 58](https://expo.dev/changelog/sdk-58)).
- **Inline modules** (SDK 56 and later, experimental) can add their Swift to more Xcode targets through `xcodeProjectTargets` ([docs](https://docs.expo.dev/modules/inline-modules-reference/)).
- **`react-native-watch-connectivity` 2.0.0** (2026-03-20) only bridges WatchConnectivity to JavaScript. Its README says it has been used in Expo apps in the "Bare Workflow with EAS Build" ([repo](https://github.com/mtford90/react-native-watch-connectivity)).
- Judgment: the module behaves the same under CNG and in a committed project. What differs is where its sources sit and whether the fingerprint sees them (§7).

## 5. Signing, versions and TestFlight on EAS

- **CNG targets.** EAS CLI must know the extensions before prebuild runs, so it reads `extra.eas.build.experimental.ios.appExtensions`. The docs file this under "CNG projects (experimental support)" ([docs](https://docs.expo.dev/build-reference/app-extensions/)). `@bacons/apple-targets` fills the list while the config loads (`withEASTargets` in [`with-widget.ts`](https://github.com/EvanBacon/expo-apple-targets/blob/8aa4c95/packages/apple-targets/src/with-widget.ts#L397)).
- **Committed project.** EAS "will automatically detect app extensions configured in your Xcode project and generate all necessary credentials for each target" ([docs](https://docs.expo.dev/build-reference/app-extensions/)). The resolver walks target dependencies recursively, so it finds the watch app and the watch widget inside it ([`target.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-cli/src/project/ios/target.ts#L122-L248)). Credentials for watch companion apps were fixed in 2021 ([#812](https://github.com/expo/eas-cli/pull/812)).
- **Watch App IDs.** EAS maps `SDKROOT` `watchos` to no platform and falls back to iOS for App IDs and profiles ([`target.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-cli/src/project/ios/target.ts#L282-L345)).
- **Managed credentials.** Generating them needs an Apple Developer login; afterwards collaborators "can start new iOS builds with only their Expo accounts", and CI with an Expo token (judgment) ([docs](https://docs.expo.dev/app-signing/managed-credentials/)). The alternative is a `credentials.json` with one profile per target ([docs](https://docs.expo.dev/app-signing/local-credentials/)). Judgment: managed credentials put the distribution certificate and profiles on Expo's servers, a trust choice for #18.
- **How EAS stamps versions.** With `cli.appVersionSource: remote` and `autoIncrement`, EAS stores the build number ([docs](https://docs.expo.dev/build-reference/app-versions/)). At build time it writes `CFBundleVersion` and `CFBundleShortVersionString` into the `INFOPLIST_FILE` of each signed target, and skips a target that has none ([`configure.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/build-tools/src/ios/configure.ts#L68-L112)). It also exports `EAS_BUILD_IOS_BUILD_NUMBER` and `EAS_BUILD_IOS_APP_VERSION` ([`env.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/worker/src/env.ts#L123-L130)).
- **Stamp every target.** Judgment: Apple's reference doesn't say whether that written key or the generated value wins when a target sets both `GENERATE_INFOPLIST_FILE` and `INFOPLIST_FILE`, and #147 shows the watch can still end up at 1. So set `CURRENT_PROJECT_VERSION` and `MARKETING_VERSION` on every target from those variables: a small local plugin under CNG, a shared xcconfig in a committed project. Check the archive in SP-03.
- **TestFlight.**
  - `eas submit -p ios` uploads the build. `--groups` adds it to existing internal groups and `--what-to-test` sets the notes ([docs](https://docs.expo.dev/submit/testflight/), [`submit.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-cli/src/commands/submit.ts)).
  - Without `ascAppId`, submit creates an internal group named "Team (Expo)" and enrolls every Admin, even with `--no-auto-testflight-setup` ([#4397](https://github.com/expo/eas-cli/issues/4397), open; secondary).
  - In `eas.json`, only the API key fields expand environment variables, not `ascAppId` ([`types.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-json/src/submit/types.ts#L39-L43)). Judgment: write the submit profile in CI to keep the app's Apple ID out of the public repository.
  - Judgment: leave out `--groups`, so a build joins the internal group only through `just testflight`, as ADR 0019 wants.

## 6. Xcode on EAS

| Image | Xcode | Alias |
|---|---|---|
| `macos-tahoe-26.6-xcode-27.1` | "27.1 RC (27A9275)" | `latest`, `sdk-58` |
| `macos-tahoe-26.6-xcode-27.0` | 27.0 (27A266a) | — |
| `macos-tahoe-26.5-xcode-26.6` | 26.6 (17F113) | `sdk-57` |

Source: [infrastructure](https://docs.expo.dev/build-reference/infrastructure/), live page and its source, last changed 2026-11-02.

- **What 27A9275 is.** Apple released build 27A9275 as Xcode 27.1 on 2026-10-22, and Xcode 27.2 (27B52) on 2026-10-30 ([releases](https://developer.apple.com/news/releases/), [release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes)). EAS lists no 27.2 image.
- **Cadence.** EAS added the SDK 58 and Xcode 27.1 images on 2026-09-29 and made 27.1 `latest` on 2026-10-16 (the doc's commit history in `expo/expo`). Judgment: expect 27.2 within weeks, and pin `image` so a moving `latest` doesn't change a release build.
- **Apple's requirements.** Since 2026-04-28, uploads must be built "with Xcode 26 or later" against the 26 SDKs ([Apple](https://developer.apple.com/news/upcoming-requirements/)). Xcode 27.1 and 27.2 deploy to watchOS 9 through 27 and iOS 15 through 27 ([Apple](https://developer.apple.com/support/xcode/)). Xcode 27.1 carries the watchOS 27 SDK, Xcode 27.2 the watchOS 27.2 SDK. So 27.2 matters only for its SDKs or its fixes.
- **Scene requirement.** On Xcode 27 images, apps "launch on iOS 27 only if they use the UIKit scene life cycle", which SDK 58 does (infrastructure page).
- **Elsewhere.** `eas build --local` ignores `image` and uses the host's Xcode ([docs](https://docs.expo.dev/build-reference/local-builds/)). GitHub's preview `xcode-27` image of 2026-10-26 defaults to 27.1 and carries 27.2 RC (27B48), not 27B52 ([readme](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md)).

## 7. EAS Update and the fingerprint

- **Runtime versions.** With the `fingerprint` policy, the runtime version is a hash of the native layer, and an update runs only on builds with the same runtime version ([docs](https://docs.expo.dev/eas-update/runtime-versions/), [expo-updates](https://docs.expo.dev/versions/latest/sdk/updates/)).
- **SDK 58's default.** The `balanced` preset ([SDK 58](https://expo.dev/changelog/sdk-58), [`Presets.ts`](https://github.com/expo/expo/blob/1e0b936c3e/packages/@expo/fingerprint/src/Presets.ts)) skips app versions, `eas.json` and `.easignore`. Native modules and config plugins under `node_modules/` collapse to name and version; in-repo files are hashed directly ([`Utils.ts`](https://github.com/expo/expo/blob/1e0b936c3e/packages/@expo/fingerprint/src/sourcer/Utils.ts#L11), [`Expo.ts`](https://github.com/expo/expo/blob/1e0b936c3e/packages/@expo/fingerprint/src/sourcer/Expo.ts#L137)).
- **What CNG hashes.** With a generated `ios/`, the fingerprint ignores `ios/**/*` ([`Options.ts`](https://github.com/expo/expo/blob/1e0b936c3e/packages/@expo/fingerprint/src/Options.ts#L95)). Its sources are then the app config with plugins applied, the modules loaded with it (including each `expo-target.config.js`, which the plugin `require`s), autolinked modules, `cng-patches/`, patch-package and `extraSources` ([`Sourcer.ts`](https://github.com/expo/expo/blob/1e0b936c3e/packages/@expo/fingerprint/src/sourcer/Sourcer.ts#L29-L93)). None of them covers the Swift, plists or assets under `targets/` (judgment, from reading the 0.21.9 source).
- **Consequence.** A change to watch or extension Swift alone keeps the runtime version. Expo's production workflow (fingerprint, then `get-build`, then build or update) then publishes an update, and the Swift never ships ([example](https://docs.expo.dev/eas/workflows/examples/deploy-to-production/)).
  - Fix: `fingerprint.config.js` with `extraSources: [{ type: 'dir', filePath: 'targets', reasons: ['appleTargets'] }]`, plus any shared Swift directory ([docs](https://docs.expo.dev/versions/latest/sdk/fingerprint/)).
  - Check: diff `npx @expo/fingerprint` before and after a watch-only edit.
- **Local modules** outside `node_modules/` are hashed as directories, so `modules/watch-relay` counts. A workspace package reached through `node_modules/` would collapse to name and version. Not verified: how autolinking reports pnpm workspace paths.
- **Committed `ios/`.** It is hashed whole (`bareNativeDir`, Pods ignored), so every target change counts.
- **What updates carry.** Only the iPhone app's JavaScript and assets: never the watch app, an extension, a Swift module or an entitlement (judgment, from the native/update layer split in the runtime versions doc). TestFlight builds also expire after 90 days (ADR 0019), so binaries ship regardless.
- **Plan limits.** The Free plan includes 1,000 MAUs, 100 GiB of bandwidth and 20 GiB of storage. End-to-end code signing of updates needs the Production plan or above ([pricing](https://expo.dev/pricing.md)). `expo-updates` also works with any server that implements the Expo Updates protocol ([docs](https://docs.expo.dev/versions/latest/sdk/updates/)).

## 8. Pricing and the Free plan (2026-11-08)

| | Free | Starter |
|---|---|---|
| Price | $0 | $19 a month plus usage |
| iOS builds | 15 a month (plus 15 Android) | $45 of credit; an iOS build costs $2 on a medium worker, $4 on a large one |
| Queue | Low priority: "wait times of 90+ minutes" at peak | High priority |
| Build timeout | 45 minutes | 2 hours |
| Concurrency | 5 jobs, at most 1 on macOS | 5 jobs, at most 2 on macOS |
| Workflows | 60 minutes of non-build jobs | Usage-based |
| Submit | Included | Included |
| Updates | 1,000 MAUs, 100 GiB bandwidth, no overage | 3,000 MAUs, then usage-based |
| Large workers | No | Yes |

Sources: [pricing](https://expo.dev/pricing), [plain-text pricing](https://expo.dev/pricing.md), [plans](https://docs.expo.dev/billing/plans/).

- **When the quota runs out.** The Free plan bills no overage: new builds wait for the next calendar month, unless the account upgrades or runs builds locally ([FAQ](https://docs.expo.dev/billing/faq/)).
- **Who may use it.** The Free plan is "only for pre-revenue startups, proof of concepts, and hobby and student projects". Commercial projects above $1,500 a month must pay, and Expo "may change … or discontinue" the plan at any time ([acceptable use](https://expo.dev/acceptable-use), updated 2026-11-02).
- **Fit** (judgment). `factory.md` estimates 8 release archives a month, and 15 iOS builds cover them with room for previews. Workflows minutes count only if Expo Workflows run the pipeline; a GitHub Actions job calling `eas build --auto-submit` uses none.
- **Risks** (judgment). A clean build of the React Native app plus five Swift targets on a medium worker is unmeasured against the 45-minute cap. Peak queues can delay a release by hours, and one macOS job runs at a time. If the cap bites, use local builds (§9.2) or Starter.

## 9. Fallbacks

### 9.1 EAS on the committed `ios/`

- When `ios/` exists, "Expo Build will not run Prebuild" ([CNG](https://docs.expo.dev/workflow/continuous-native-generation/)). EAS detects the targets (§5), and the fingerprint hashes `ios/` (§7).
- This keeps ADR 0005's Xcode ownership and adds EAS signing, submission and, optionally, updates.
- The cost is the one ADR 0005 already accepts: app config fields such as `ios`, `plugins` and `icon` stop syncing ([check](https://github.com/expo/expo/blob/main/packages/expo-doctor/src/checks/AppConfigFieldsNotSyncedToNativeProjectsCheck.ts)).

### 9.2 `eas build --local` on GitHub-hosted macOS runners

- It runs the same build steps on any Mac, contacting Expo only to confirm the project and download managed credentials ([docs](https://docs.expo.dev/build-reference/local-builds/)).
- It ignores `image` and caching, and the host needs fastlane and CocoaPods.
- Standard GitHub-hosted runners are free on public repositories ([GitHub](https://docs.github.com/en/billing/concepts/product-billing/github-actions)), and a job may run 6 hours ([limits](https://docs.github.com/en/actions/reference/limits)).
- The Xcode is whatever the runner image carries (§6). Judgment: this removes the build quota, the 45-minute cap and the queue, at the cost of runner minutes on the critical path.

### 9.3 Xcode Cloud after a prebuild

- Apple: "Xcode Cloud requires a consistent Xcode project or workspace that's continuously present. If you use a third-party tool that dynamically generates or edits your project or workspace, the initial configuration of Xcode Cloud and subsequent builds may fail" ([Apple](https://developer.apple.com/documentation/xcode/setting-up-your-project-to-use-xcode-cloud)).
- Custom scripts sit in a `ci_scripts` directory "located in the same directory as your Xcode project or workspace" ([Apple](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts)).
- So Xcode Cloud means committing the prebuild output, which is ADR 0005's path and `factory.md`'s F1, not a CNG fallback.
- The Developer Program includes 25 compute hours a month ([Apple](https://developer.apple.com/xcode-cloud/)).
- Xcode Cloud's release notes announced Xcode 26 on 2025-09-15 and Xcode 26.2 on 2025-12-19, then stop at 2026-02-10, so Xcode 27.x availability there is unconfirmed ([release notes](https://developer.apple.com/xcode-cloud/release-notes/)).

## Recommendation for Shoestring

1. **Keep the committed `ios/` and, if #18 chooses EAS, build and submit it on EAS** (judgment, **medium**). EAS handles the watch app and every extension in a committed project with no target generator, hashes the whole native project for updates, and fits the Free plan. Decision 8's CNG amendment of ADR 0005 waits.
2. **Return to CNG only when SP-03's CNG path passes on the SDK in use.** That needs a stable `@bacons/apple-targets` release for that SDK, with #147 and #171 fixed or their workarounds proven. Adopting it also means accepting its alpha `@bacons/xcode` dependency, against the stable-only rule, or owning a fork.
3. **Settings, CNG or not:**
   - pin `image: macos-tahoe-26.6-xcode-27.1` until a 27.2 image exists;
   - use eas-cli 27.3.0 or later;
   - set `appVersionSource: remote` and stamp every target's build and marketing versions from `EAS_BUILD_IOS_BUILD_NUMBER` and `EAS_BUILD_IOS_APP_VERSION`;
   - write a submit profile with `ascAppId` in CI, pass `--no-auto-testflight-setup` and no `--groups`;
   - run the first credentials setup interactively, for the App Groups;
   - under CNG, add `targets/` to `extraSources`.
4. **Start on the Free plan.** Move to local builds on GitHub runners, or to Starter, the day a build passes 40 minutes or a queue delays a release.

**Trade-offs to accept:** a release depends on Expo's queue and terms on the Free plan. Managed credentials live with Expo unless `credentials.json` is used. SDK upgrades stay hand-applied to `ios/` until CNG returns.

## Open questions and spikes

**SP-03 additions:**
- **EAS path.** Run `eas build -p ios` on the `sdk-58` image for the committed project, with the watch app, complications, NSE, App Intents extension and an iOS widget. Then `eas submit`, without `--groups`. Pass: the watch app installs from the iPhone, every target's `CFBundleVersion` and `CFBundleShortVersionString` match the app's, and the build time is recorded against the 45-minute cap.
- **CNG path.** Build the same targets with `@bacons/apple-targets` 5.0.0 on SDK 58. Pass: a signed EAS build uploads, with version stamping from a local plugin and shared Swift in `targets/_shared`. Also check that a watch-only Swift edit changes the fingerprint only with `extraSources`.
- **Credentials.** The first build runs interactively, and later builds run with `EXPO_TOKEN` alone.

**Not verified:**
- `@bacons/apple-targets` 5.0.0 on SDK 58 or with Xcode 27.1;
- a watchOS archive on an EAS image;
- the build time on a Free-plan worker;
- whether EAS's plist stamping or the generated `CURRENT_PROJECT_VERSION` wins for plugin targets;
- when EAS adds Xcode 27.2, and whether Xcode Cloud offers 27.x;
- how autolinking reports pnpm workspace module paths to the fingerprint.

## Sources

Accessed 2026-11-08.

- **Shoestring:** [#7 charting decisions](https://github.com/fveauvy/shoestring/issues/7) · [ADR 0005](../adr/0005-swift-where-ios-needs-it-in-a-committed-ios-project.md) · [ADR 0006](../adr/0006-watch-shows-a-glance-relayed-by-the-iphone.md) · [ADR 0015](../adr/0015-one-message-catalog-for-every-platform.md) · [ADR 0019](../adr/0019-native-apps-through-the-testflight-internal-group-only.md) · [`factory.md`](../factory.md)
- **npm:** [expo](https://www.npmjs.com/package/expo) · [eas-cli](https://www.npmjs.com/package/eas-cli) · [@bacons/apple-targets](https://www.npmjs.com/package/@bacons/apple-targets) · [@bacons/xcode](https://www.npmjs.com/package/@bacons/xcode) · [expo-widgets](https://www.npmjs.com/package/expo-widgets) · [expo-app-intents](https://www.npmjs.com/package/expo-app-intents) · [expo-updates](https://www.npmjs.com/package/expo-updates) · [@expo/fingerprint](https://www.npmjs.com/package/@expo/fingerprint) · [patch-project](https://www.npmjs.com/package/patch-project) · [react-native-watch-connectivity](https://www.npmjs.com/package/react-native-watch-connectivity) · [expo-nse-plugin](https://www.npmjs.com/package/expo-nse-plugin)
- **`@bacons/apple-targets`:** [README](https://github.com/EvanBacon/expo-apple-targets) · source at [`8aa4c95`](https://github.com/EvanBacon/expo-apple-targets/tree/8aa4c95/packages/apple-targets/src) · [#122](https://github.com/EvanBacon/expo-apple-targets/pull/122) · [#147](https://github.com/EvanBacon/expo-apple-targets/issues/147) · [#148](https://github.com/EvanBacon/expo-apple-targets/pull/148) · [#171](https://github.com/EvanBacon/expo-apple-targets/issues/171) · [#177](https://github.com/EvanBacon/expo-apple-targets/pull/177) · [#195](https://github.com/EvanBacon/expo-apple-targets/pull/195) · [#196](https://github.com/EvanBacon/expo-apple-targets/issues/196) · [#201](https://github.com/EvanBacon/expo-apple-targets/issues/201) · [#202](https://github.com/EvanBacon/expo-apple-targets/issues/202) · [#207](https://github.com/EvanBacon/expo-apple-targets/issues/207) · [#210](https://github.com/EvanBacon/expo-apple-targets/pull/210) (secondary)
- **Expo docs:** [app extensions](https://docs.expo.dev/build-reference/app-extensions/) · [infrastructure](https://docs.expo.dev/build-reference/infrastructure/) · [iOS capabilities](https://docs.expo.dev/build-reference/ios-capabilities/) · [app versions](https://docs.expo.dev/build-reference/app-versions/) · [local builds](https://docs.expo.dev/build-reference/local-builds/) · [CNG](https://docs.expo.dev/workflow/continuous-native-generation/) · [managed credentials](https://docs.expo.dev/app-signing/managed-credentials/) · [local credentials](https://docs.expo.dev/app-signing/local-credentials/) · [TestFlight](https://docs.expo.dev/submit/testflight/) · [runtime versions](https://docs.expo.dev/eas-update/runtime-versions/) · [expo-updates](https://docs.expo.dev/versions/latest/sdk/updates/) · [fingerprint](https://docs.expo.dev/versions/latest/sdk/fingerprint/) · [deploy workflow](https://docs.expo.dev/eas/workflows/examples/deploy-to-production/) · [widgets](https://docs.expo.dev/versions/latest/sdk/widgets/) · [app intents](https://docs.expo.dev/versions/latest/sdk/app-intents/) · [local modules](https://docs.expo.dev/modules/get-started/) · [AppDelegate subscribers](https://docs.expo.dev/modules/appdelegate-subscribers/) · [inline modules](https://docs.expo.dev/modules/inline-modules-reference/) · [patch-project](https://docs.expo.dev/config-plugins/patch-project/) · [plans](https://docs.expo.dev/billing/plans/) · [billing FAQ](https://docs.expo.dev/billing/faq/)
- **Expo, other:** [pricing](https://expo.dev/pricing) · [pricing.md](https://expo.dev/pricing.md) · [acceptable use](https://expo.dev/acceptable-use) · [SDK 58](https://expo.dev/changelog/sdk-58) · [#52987](https://github.com/expo/expo/issues/52987) · [#52467](https://github.com/expo/expo/pull/52467) · [#53083](https://github.com/expo/expo/pull/53083) · fingerprint source at [`1e0b936c3e`](https://github.com/expo/expo/tree/1e0b936c3e/packages/@expo/fingerprint/src) · [expo-doctor unsynced-fields check](https://github.com/expo/expo/blob/main/packages/expo-doctor/src/checks/AppConfigFieldsNotSyncedToNativeProjectsCheck.ts)
- **eas-cli:** source at [`50dcb89a0c`](https://github.com/expo/eas-cli/tree/50dcb89a0c) ([`target.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-cli/src/project/ios/target.ts), [`configure.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/build-tools/src/ios/configure.ts), [`env.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/worker/src/env.ts), [`submit.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-cli/src/commands/submit.ts), [`types.ts`](https://github.com/expo/eas-cli/blob/50dcb89a0c/packages/eas-json/src/submit/types.ts)) · [releases](https://github.com/expo/eas-cli/releases) · [#812](https://github.com/expo/eas-cli/pull/812) · [#4397](https://github.com/expo/eas-cli/issues/4397) · [#4445](https://github.com/expo/eas-cli/issues/4445)
- **Apple:** [releases](https://developer.apple.com/news/releases/) · [Xcode 27.2 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes) · [Xcode support](https://developer.apple.com/support/xcode/) · [upcoming requirements](https://developer.apple.com/news/upcoming-requirements/) · [keychain sharing](https://developer.apple.com/documentation/security/sharing-access-to-keychain-items-among-a-collection-of-apps) · [supported capabilities (iOS)](https://developer.apple.com/help/account/reference/supported-capabilities-ios) · [Xcode Cloud setup](https://developer.apple.com/documentation/xcode/setting-up-your-project-to-use-xcode-cloud) · [custom build scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts) · [Xcode Cloud](https://developer.apple.com/xcode-cloud/) · [Xcode Cloud release notes](https://developer.apple.com/xcode-cloud/release-notes/)
- **GitHub:** [xcode-27 runner image](https://github.com/actions/runner-images/blob/main/images/macos/xcode-27-arm64-Readme.md) · [preview announcement](https://github.com/actions/runner-images/issues/14404) · [Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions) · [Actions limits](https://docs.github.com/en/actions/reference/limits)
- **Other:** [react-native-watch-connectivity](https://github.com/mtford90/react-native-watch-connectivity)
