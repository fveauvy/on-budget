# UI stack for iOS, watchOS, macOS and web

> **Status (2026-11-08).** Background research. The ADRs, `SPEC.md`, [`docs/factory.md`](../factory.md) and [`docs/architecture.md`](../architecture.md#references) win where this note disagrees. It uses on-budget, the working name, for what is now Shoestring ([ADR 0020](../adr/0020-shoestring-as-the-product-name.md)). Superseded or amended here: §4.4 watch writes, §9 regional profile, §10 web SQLite; DPoP.

Research note, 2026-11-07, revised twice the same day: after independent fact-checks, then after a completeness review (native project, sign-in, App Intents, device security, Mac paths, S2 criteria, tests and CI). The revisions follow the owner's decision of one UI codebase for the iPhone, Mac and web apps ([ADR 0002](../adr/0002-one-ui-codebase-native-watch.md)), which moves the recommendation from A to B. Versions are the latest stable release on that date (npm, GitHub releases, crates.io and vendor changelogs, all read that day) unless marked otherwise. "Secondary" marks third-party sources. "Judgment" marks my own assessment, not a sourced fact. Confidence levels are high, medium or low.

This note builds on [`apple-client-stack.md`](apple-client-stack.md), whose SwiftUI-only UI recommendation that decision supersedes, and on [`native-apps-auth.md`](native-apps-auth.md), [`sync-engine.md`](sync-engine.md), [`web-client-sync.md`](web-client-sync.md), [`watchos-app.md`](watchos-app.md), [`bank-alternatives.md`](bank-alternatives.md), [`design-system.md`](design-system.md), [`i18n-l10n.md`](i18n-l10n.md), [`factory-ci-quality.md`](factory-ci-quality.md) and [`budgeting-products.md`](budgeting-products.md). It edits none of them.

## Question

Which UI technology strategy best serves an iPhone app, an Apple Watch app, a Mac app and a web app with one strict design system and as much component reuse as possible, without giving up the native-feeling UX the owner wants every day?

The constraints come from `PRODUCT.md`, the ADRs and the 2026-11-07 additions:

- one UI codebase shared by the iOS, macOS and web apps ("same code everywhere"). [ADR 0002](../adr/0002-one-ui-codebase-native-watch.md) records it: the owner accepts "a less native feel on the Mac in exchange for one implementation per component";
- a native SwiftUI Apple Watch app;
- a web app "at full parity with the native apps, offline included";
- self-hosted open source: each instance belongs to one owner, with no public sign-up ([ADR 0001](../adr/0001-self-hosted-single-owner-instances.md)). Every feature must work when the instance "is reachable only over a private network" ([ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md)). How self-hosters get the native apps is undecided;
- light and dark themes;
- a foundations-first design system enforced in every app, "with each component shared by the apps";
- English and French now, more languages later, no right-to-left;
- European regional formats: dates, 12/24-hour clock, time zones, currencies and numbers;
- offline-first everywhere, with sync to the instance's API;
- Face ID / Touch ID on top of sign-in, alerts on category limits, and bank data kept safe on every device ("Your data, your server", `PRODUCT.md` principle 4). Sign-in follows `native-apps-auth.md`: the instance's passkey page in a browser sheet, OAuth with PKCE, and DPoP-bound tokens;
- App Intents: `bank-alternatives.md` makes Apple Pay tap capture through an App Intent the owner's second capture route, and `watchos-app.md` plans cash entry through one, so they are required. Widgets stay optional;
- one developer, with most code written by Claude Code and Codex.

## TL;DR

1. **Pick strategy B: Expo (React Native) for the iPhone, web and Mac apps, plus a native SwiftUI watch app, held together by one token source and one component spec.**
   - One React Native design-system package serves the iPhone app, the web app (react-native-web through Expo Router) and the Mac. Its components sit on RN primitives and are styled only from generated tokens.
   - On the iPhone, native tabs and Expo UI's SwiftUI controls supply system chrome where it matters. Their web versions live in the same component, in `.ios.tsx` and `.web.tsx` files.
   - Swift is more than a small extra: the watch app, a WatchConnectivity module that answers the watch natively, the notification service extension (NSE), a sign-in and Secure Enclave module, an App Intents extension for tap and cash capture, and any widgets (§4.4, §4.8).
   - Commit the `ios/` project (one `expo prebuild`, then Xcode owns it). Continuous Native Generation can't ship the watch today: `@bacons/apple-targets` 5.0 is still the SDK 55 line (§4.7).
   - The Mac starts as the Dock web app or the iPad app on the Mac (matrix in §4.2), and moves to react-native-macos once Expo supports macOS again.
   - **Confidence: medium**, conditional on spikes S2 (gate 2) and S4 (gate 3).
2. **B is the only option that passes all three gates** (§7).
   - Gate 1, one UI codebase for iPhone, Mac and web (ADR 0002), rules out A and F. **High.**
   - Gate 2, native feel on the iPhone, the main daily surface, rules out C, D and E, which draw their own pixels or run in a WebView. B keeps that feel: RN renders UIKit views, Expo UI (real SwiftUI views) has been stable since SDK 56 (May 2026), and native tabs are stable in SDK 58. But this gate is a judgment score, and it alone eliminates E, so S2 must confirm it against measured criteria. **Medium.**
   - Gate 3, a watch app with its extensions that builds and reaches TestFlight: B passes only with a committed `ios/` (§4.7). **Medium-high.**
3. **No strategy gets one implementation per component across all four apps.**
   - watchOS has neither `WKWebView` nor JavaScriptCore, and widgets, complications, Live Activities and controls are built only in SwiftUI.
   - So the watch keeps its own SwiftUI subset in every option, and every shared option (B to E) carries the same Swift glue on the iPhone. The watch shouldn't run the sync engine either: it shows a server-built glance that the iPhone relays (`watchos-app.md`).
   - **High.**
4. **Runner-up: E (one web app wrapped in native shells).**
   - It has the best weighted total among the options that pass gate 1 (124 against B's 113), the best web app and the most reuse. It carries B's Swift glue, but Capacitor commits its Xcode project by design, so watch and extension targets are ordinary Xcode work (§4.7).
   - It fails gate 2: a WebView with no system controls. Take E if S2 shows B isn't clearly better on the iPhone (pass criteria in the spikes), or if the owner accepts a web feel there.
   - C (Flutter) and D (Compose Multiplatform) fail the same gate. Flutter draws its web UI on a canvas, and Compose's web target, still Beta, draws its own pixels too (Compose: secondary).
   - **Medium.**
5. **A (SwiftUI plus a separate React web app) still scores highest (149 of 175), but ADR 0002 rules it out.**
   - It is the only option with native feel on both the iPhone and the Mac, at the price of two UI codebases. Revisit it only if the owner reverses ADR 0002.
   - F (a Rust or Kotlin core under A's native UIs, 126) is out for the same reason. Its Kotlin variant reaches Swift through Objective-C headers by default; its real costs are an extra language and a Beta watchOS target.
   - **High.**
6. **Self-hosting raises the web app's weight and fixes native sign-in** (ADR 0001, ADR 0003, `native-apps-auth.md`).
   - The web app is the one client every self-hoster can run, which is one more reason for `PRODUCT.md`'s "full parity, offline included".
   - Native passkey sheets need each domain in the signed entitlement, so every native app signs in on the instance's own page inside `ASWebAuthenticationSession`, with PKCE and DPoP-bound tokens from a Secure Enclave key.
   - Under B that is a Swift module: `expo-web-browser` lists no macOS and uses the initializer the 27.3 SDKs deprecate, and no maintained RN library signs with the Secure Enclave (§4.8).
   - **Medium-high.**
7. **Trade-offs the owner accepts with B:**
   - the Mac is the weakest client until react-native-macos catches up. Expo SDK 58 "does not support macOS yet", and react-native-macos is at 0.83 while React Native is at 0.88;
   - Swift for the watch and its complications, four native pieces on the iPhone side (WatchConnectivity module, NSE, sign-in module, App Intents extension) and any iOS widgets, plus a committed `ios/` whose native changes are applied by hand at each Expo SDK upgrade;
   - no over-the-air JavaScript updates in v1: every fix ships as a signed TestFlight build (§4.8);
   - Hermes ships only a partial `Intl` on iOS (no `formatToParts`, `signDisplay` or compact notation), so money formatting needs FormatJS polyfills or a native formatter module;
   - a new Expo SDK every few months, and Apple's changes reach B only through Expo;
   - Figma Code Connect fits the Professional plan while it maps only the RN components; mapping the SwiftUI watch components too needs Organization;
   - Claude Design prototypes arrive as HTML, so the RN and SwiftUI components are ported from them.
   - **High** on the facts.

## 1. The native floor: what every strategy writes in SwiftUI

| Surface | What it requires | Source |
|---|---|---|
| Apple Watch app | SwiftUI ("Develop powerful, personal apps for Apple Watch using SwiftUI"). `WKWebView` ships on iOS, iPadOS, Mac Catalyst, macOS and visionOS, not watchOS; JavaScriptCore isn't on watchOS either | [watchOS apps](https://developer.apple.com/documentation/watchos-apps/building_a_watchos_app), [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview), [JavaScriptCore](https://developer.apple.com/documentation/javascriptcore) |
| Widgets, watch complications, Live Activities, controls | WidgetKit: "You build widgets using SwiftUI". Live Activity UIs are built with WidgetKit and SwiftUI | [widget extension](https://developer.apple.com/documentation/widgetkit/creating-a-widget-extension), [WidgetKit](https://developer.apple.com/documentation/widgetkit), [ActivityKit](https://developer.apple.com/documentation/activitykit) |
| Siri, Shortcuts, Spotlight, Apple Intelligence | App Intents (Swift); iOS 27 adds `SyncableEntity`, `LongRunningIntent` and more | [App Intents](https://developer.apple.com/documentation/appintents), [updates](https://developer.apple.com/documentation/updates/appintents) |
| Face ID / Touch ID, Keychain, Secure Enclave, passkeys, decrypting pushes | LocalAuthentication, Security, AuthenticationServices, a notification service extension | `apple-client-stack.md` §3 and §4.3 |
| Watch ↔ iPhone data | WatchConnectivity for companion apps. An independent watch app "can't rely on" it and must sync "through your own server", but only through `URLSession` HTTP(S): watchOS counts `URLSessionWebSocketTask`, `URLSessionStreamTask`, Network framework and Bonjour as low-level networking and blocks them, except for audio streaming, VoIP calls and a tvOS service listener | [WatchConnectivity](https://developer.apple.com/documentation/watchconnectivity), [independent apps](https://developer.apple.com/documentation/watchos-apps/creating-independent-watchos-apps), [TN3135](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos) |

So every strategy ships at least a small SwiftUI design system for the watch, which ADR 0002 makes native, and for any widgets: tokens, an amount label, a progress ring, a category chip and list rows. A would extend that set to the iPhone and the Mac; B, C, D and E keep it to the watch and the widgets. The rest of the table is Swift glue that every shared option also writes: a WatchConnectivity module, the NSE, a sign-in and Secure Enclave module, and an App Intents extension (§4.4). A writes the same pieces in its own language. (Judgment; follows from the table.)

## 2. The options as of November 2026

| | iPhone | Mac | Watch | Web | Maturity |
|---|---|---|---|---|---|
| **A** Native + separate web | SwiftUI (Swift 6.4, iOS 27 SDK) | SwiftUI (multiplatform target) | SwiftUI | React [19.3.1](https://www.npmjs.com/package/react), Vite 8.3, React Aria Components [1.22.1](https://www.npmjs.com/package/react-aria-components) | All first-party or widely used |
| **B** RN universal | Expo SDK 58 ([changelog](https://expo.dev/changelog/sdk-58), 2026-10-15), RN [0.88.1](https://www.npmjs.com/package/react-native), Expo UI 58 | react-native-macos [0.83.0](https://github.com/microsoft/react-native-macos/releases) (2026-09-30, no Expo; 0.86 and 0.87 in open PRs), Mac Catalyst (bare RN), the iPad app on Mac or the Dock web app (§4.2) | SwiftUI targets in a committed `ios/`; [`@bacons/apple-targets`](https://github.com/EvanBacon/expo-apple-targets) 5.0 is the SDK 55 line (§4.7) | react-native-web [0.21.4](https://www.npmjs.com/package/react-native-web) via Expo Router | iOS mature; Mac lags; web stable but secondary |
| **C** Flutter | Flutter 3.47.9 ([releases](https://docs.flutter.dev/release/archive), 2026-10-29) | Flutter macOS | SwiftUI | Flutter web (CanvasKit / Skwasm) | Stable; Apple look lags |
| **D** Compose Multiplatform | CMP [1.12.2](https://github.com/JetBrains/compose-multiplatform/releases) (1.13 beta) | Compose Desktop (JVM) | SwiftUI | Kotlin/Wasm, **Beta** ([stability](https://kotlinlang.org/docs/multiplatform/supported-platforms.html)) | iOS stable since 2025; web beta |
| **E** Web-first wrapped | Capacitor [8.6.0](https://github.com/ionic-team/capacitor/releases) or Tauri [2.12.5](https://github.com/tauri-apps/tauri/releases) | Tauri 2 or Electron [45.2](https://www.npmjs.com/package/electron) | SwiftUI | The same web app | Stable wrappers; Tauri 3 in alpha |
| **F** Shared core | A's UIs over Rust ([UniFFI](https://github.com/mozilla/uniffi-rs) 0.32.2, [Crux](https://github.com/redbadger/crux) 0.20) or Kotlin (Objective-C framework by default; opt-in [Swift export](https://kotlinlang.org/docs/native-swift-export.html) in Alpha) | Same | Same | Same + Wasm | Rust tooling pre-1.0; Kotlin's watchOS target Beta |

**ADR 0002 check.** A and F keep two UI codebases (SwiftUI and React). B, C, D and E share one across the iPhone, Mac and web.

Not shortlisted:
- **Snap's [Valdi](https://github.com/Snapchat/Valdi)** (beta) targets iOS, Android and macOS, with no web target.
- **[Skip](https://github.com/skiptools/skip) 1.9.20** (now free, Apache-2.0) turns SwiftUI into Compose for Android, with no web target. It is the path to keep in mind if Android ever matters under A.

## 3. A: SwiftUI on Apple platforms, React on the web

**Status:** ruled out by ADR 0002, because it keeps two UI codebases. It stays here as the reference for native feel and as the fallback if the owner reverses that decision. Its SwiftUI findings still apply to the watch app.

### 3.1 Apple side: one package, four surfaces

- **Sharing.**
  - A Swift package declares iOS, macOS and watchOS in `platforms:` ([SupportedPlatform](https://developer.apple.com/documentation/packagedescription/supportedplatform)), and widget extensions import the same package.
  - The data layer compiles for the watch too: GRDB 7.12 supports watchOS 7+ ([Package.swift](https://github.com/groue/GRDB.swift/blob/master/Package.swift)), and the PowerSync Swift SDK 1.17 declares watchOS 9 ([Package.swift](https://github.com/powersync-ja/powersync-swift/blob/main/Package.swift)). A `Package.swift` entry doesn't make sync work there, though. watchOS blocks WebSocket and stream tasks for ordinary apps ([TN3135](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos)), and `watchos-app.md` concludes "Don't run the sync engine on the watch". The watch reads a small server-built glance instead.
  - What is really shared (judgment): the tokens, the components and the view models on all three platforms. Screen layouts are shared between iPhone and Mac, with a tab bar on the iPhone and a sidebar on the Mac. Watch screens are separate but built from the same components.
- **Design language.**
  - Liquid Glass "extends across iOS 26, iPadOS 26, macOS Tahoe 26, watchOS 26, and tvOS 26", with SwiftUI, UIKit and AppKit APIs for adopting it ([Apple](https://www.apple.com/newsroom/2025/06/apple-introduces-a-delightful-and-elegant-new-software-design/)).
  - In 2026 SwiftUI added, with Xcode 27 ([SwiftUI updates](https://developer.apple.com/documentation/updates/swiftui)):
    - toolbar overflow and `visibilityPriority`;
    - `reorderable()` and swipe actions in any container;
    - `alert(item:)`;
    - `ArrangementView` and hinge APIs for the foldable iPhone Duo (September 2026).
- **Accessibility and Dynamic Type.**
  - Custom fonts scale through `Font.custom(_:size:relativeTo:)` ([doc](https://developer.apple.com/documentation/swiftui/font/custom%28_:size:relativeto:%29)), and spacing through `@ScaledMetric` ([doc](https://developer.apple.com/documentation/swiftui/scaledmetric)).
  - System controls bring VoiceOver, Full Keyboard Access and Mac menu behavior with them (judgment, standard SwiftUI).
- **Mac caveat.** Polish may still need AppKit escape hatches for tables, menus and keyboard handling. That was the main risk in `apple-client-stack.md` §1 (spike S1 there).
- **Regional settings.** `Locale.current` represents "the user’s region settings at the time the property is read", and `hourCycle` returns the user's override "if the user overrode the default hour cycle" ([current](https://developer.apple.com/documentation/foundation/locale/current), [hourCycle](https://developer.apple.com/documentation/foundation/locale/hourcycle-swift.property)).
- **Tests and agents.** Use Swift Testing, swift-snapshot-testing 1.19.7 (iOS and macOS; the watch needs `ImageRenderer` diffs, §8.4) and XCUITest, plus the Xcode 27 MCP tools and MobileBuildMCP 2.7.1, all detailed in `factory-ci-quality.md`.

### 3.2 Web side

- **Stack** (npm versions):
  - React 19.3.1, Vite 8.3.6, and TanStack Router 1.170 or React Router 8.5.
  - **React Aria Components 1.22.1** provides "localized strings for 30+ languages", handles "many calendar and numbering systems", and has screen-reader and keyboard support "tested across a wide variety of devices, browsers, and screen readers" ([quality](https://react-aria.adobe.com/quality)).
  - Its `DatePicker` takes `ZonedDateTime` values and an `hourCycle` 12/24 prop ([DatePicker](https://react-aria.adobe.com/DatePicker)). `NumberField.formatOptions` is Intl.NumberFormat-compatible ([NumberField](https://react-aria.adobe.com/NumberField)).
  - Alternative: Base UI, stable since 2025-12-11 and now at 1.9.0 ([releases](https://base-ui.com/react/overview/releases)).
- **Offline storage** ([SQLite Wasm persistence](https://sqlite.org/wasm/doc/trunk/persistence.md); package 3.54.1):
  - The `opfs-sahpool` VFS needs no COOP/COEP headers but has limited multi-connection concurrency.
  - `opfs` needs COOP/COEP and Safari 17+.
  - `opfs-wl`, added in 3.53.0, works like `opfs` but locks through Web Locks, which queue requests in FIFO order. It needs `Atomics.waitAsync()`. The docs name `opfs` and `opfs-wl` for clients that need multi-tab concurrency.
  - WebKit grants persistent storage by heuristics "like whether the website is opened as a Home Screen Web App" ([WebKit](https://webkit.org/blog/14403/updates-to-storage-policy/)). So ship a PWA, call `navigator.storage.persist()`, and treat the server as the source of truth, as the sync note already does.
- **Browser support** ([MDN browser-compat-data](https://github.com/mdn/browser-compat-data)):
  - `Temporal` ships in Chrome 144 and Firefox 139, but Safari has it only in Technology Preview. Until Safari ships it, use `temporal-polyfill` 1.0.8 or `@internationalized/date`.
  - CSS `light-dark()` works in Safari 17.5+, Chrome 123+ and Firefox 120+.
- **Mac safety net.** Since Safari 17, any site can be added to the Dock and runs "in its own window as a web app", with web push and badging ([WebKit](https://webkit.org/blog/14445/webkit-features-in-safari-17-0/)). It is one of B's two first Mac clients (§4.2).
- **Auth across clients.** In a browser, WebAuthn checks the RP ID against the page's origin, so the web app needs no Apple-side setup (judgment). Native passkey sheets and universal links need each domain in the signed Associated Domains entitlement, "a list of domains for each enabled service", plus an association file that devices fetch through "an Apple-managed content delivery network (CDN)" ([entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.associated-domains), [associated domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains)). One distributed build can't cover every self-hoster's domain, so `native-apps-auth.md` signs every native app in on the instance's own page inside `ASWebAuthenticationSession`, whatever the strategy (§4.8 for B).
- **Precedent** (secondary, sponsored post). Copilot Money, native on iPhone, iPad and Mac, added a web app around December 2025 ([9to5Mac](https://9to5mac.com/2026/01/01/copilot-money-brings-clarity-to-your-finances-now-on-the-web/)). Feature parity was not verified.

## 4. B: React Native universal (Expo)

**Status:** recommended (§7 and the recommendation).

### 4.1 iPhone

- **Rendering and cadence.**
  - RN renders UIKit views, and has run only on the New Architecture since 0.82. Hermes V1 has been the default since 0.84 ([RN blog](https://reactnative.dev/blog)).
  - Release cadence: an RN minor every ~2 months, and Expo SDKs 55 (Feb), 56 (May), 57 (Jun) and 58 (Sep/Oct) in 2026 (npm release dates).
- **Real SwiftUI views.**
  - Expo UI renders SwiftUI inside a `Host` that "uses UIHostingController" ([guide](https://docs.expo.dev/guides/expo-ui-swift-ui/)). It has been "stable and ready for production" since SDK 56 ([Expo blog](https://expo.dev/blog/expo-ui-stable-sdk-56), 2026-05-26).
  - SDK 58 added `NavigationStack`, `NavigationSplitView`, toolbars, recycling lists and `ArrangementView` ([SDK 58](https://expo.dev/changelog/sdk-58)).
- **System chrome.**
  - Native tabs use the system tab bar, drawn with Liquid Glass on iOS 26+, under a stable import path since SDK 58. Limits: tabs can't change at runtime, FlatList integration is limited, and web gets a basic fallback "loosely based on iPad design" ([native tabs](https://docs.expo.dev/router/advanced/native-tabs/)).
  - `GlassView` is iOS 26+ only and "will fallback to regular View" elsewhere ([expo-glass-effect](https://docs.expo.dev/versions/latest/sdk/glass-effect/)).
- **Dynamic Type.** RN text scales with it by default (`allowFontScaling`, `dynamicTypeRamp`; [Text](https://reactnative.dev/docs/text)).
- **Design system.** Expo says "Expo UI isn't a UI component library or a fancy design system" and recommends RN `View`/`Text` for brand UI ([blog](https://expo.dev/blog/expo-ui-stable-sdk-56)). A B design system is therefore custom RN components for brand pieces plus Expo UI SwiftUI controls on iOS. Metro picks `.ios`, `.native` and `.web` variants of a file per platform; inside Expo Router's `app` directory only when a non-platform version also exists ([platform-specific modules](https://docs.expo.dev/router/advanced/platform-specific-modules/)).
- **Lag behind Apple** (judgment, from the changelogs). Apple's changes reach B through Expo. For example, apps built with the iOS 27 SDK "must use the UIKit scene-based life cycle". This was opt-in in SDK 57 (2026-06-30) and became the default in SDK 58 (2026-10-15) ([SDK 57](https://expo.dev/changelog/sdk-57), [SDK 58](https://expo.dev/changelog/sdk-58)).

### 4.2 Mac

- **react-native-macos.**
  - Expo: "SDK 58 does not support macOS yet". "Support will return when `react-native-macos` is updated to at least React Native 0.86" ([SDK 58](https://expo.dev/changelog/sdk-58)).
  - react-native-macos 0.83.0, "the first 0.83 release", syncs with RN 0.83.10, five minors behind core ([release](https://github.com/microsoft/react-native-macos/releases)).
  - The catch-up sits in open PRs: "prepare stable 0.86.0 with React Native 0.86.3" (#3036, opened 2026-07-18) and the 0.87.1 sync (#3100, #3104 and #3105, opened 2026-09-22). None is merged, and none has changed since 2026-09-23 ([PRs](https://github.com/microsoft/react-native-macos/pulls)). B's Mac score in §7 is therefore dated 2026-11-07 and may rise once 0.86 ships with Expo support.
  - Expo's macOS guide warns that iOS-first libraries usually need extra work ([doc](https://docs.expo.dev/modules/additional-platform-support/)).
- **The iOS app on Mac.**
  - "iPhone apps always run in a fixed-size window". iPad apps resize only if they support iPad multitasking ([Apple](https://developer.apple.com/documentation/apple-silicon/running-your-ios-apps-in-macos)), and they report the `.pad` idiom ([Apple](https://developer.apple.com/documentation/apple-silicon/adapting-ios-code-to-run-in-the-macos-environment)).
  - Mac Catalyst "can only use AppKit APIs marked as available in Mac Catalyst" ([Apple](https://developer.apple.com/documentation/uikit/mac-catalyst)). It works in bare RN with Podfile changes (secondary, [Software Mansion](https://docs.swmansion.com/react-native-audio-api/docs/other/maccatalyst)), but Expo doesn't officially support it.

**Mac paths under B, as of 2026-11-07** (the matrix behind owner question 1). All stay within one codebase; a separate SwiftUI Mac app would break ADR 0002, and Mac Catalyst has no Expo support (above).

| | Dock web app (§3.2) | iPad app on the Mac | react-native-macos 0.83 (bare RN) |
|---|---|---|---|
| Available | Now | Now. TestFlight serves iPhone and iPad builds to Apple silicon Macs, a setting made "at the group level" ([Apple](https://developer.apple.com/help/app-store-connect/test-a-beta-version/test-iphone-and-ipad-apps-on-macs-with-apple-silicon)) | Not with Expo: SDK 58 has no macOS |
| Extra work | None beyond the web app | An unplanned iPad target. Expo's `ios.supportsTablet` "Defaults to `false`" ([app config](https://docs.expo.dev/versions/latest/config/app/)), resizable Mac windows need iPad multitasking ([Apple](https://developer.apple.com/documentation/apple-silicon/adapting-ios-code-to-run-in-the-macos-environment)), and the tablet layouts ship to iPads too | Unistyles 3.8 requires "React Native 0.85+" ([README](https://github.com/jpudysz/react-native-unistyles)); PowerSync's React Native table doesn't list it ([platforms](https://docs.powersync.com/resources/supported-platforms)); `expo-web-browser`, `expo-local-authentication`, `expo-secure-store` and `expo-localization` list no macOS ([web browser](https://docs.expo.dev/versions/latest/sdk/webbrowser/), [local auth](https://docs.expo.dev/versions/latest/sdk/local-authentication/), [secure store](https://docs.expo.dev/versions/latest/sdk/securestore/), [localization](https://docs.expo.dev/versions/latest/sdk/localization/)) |
| Offline data | SQLite Wasm in OPFS. Safari copies only cookies into the new app, and whether a Dock app escapes ITP's 7-day deletion of script-written storage is unverified (`web-client-sync.md` §2.2; S3 tests it) | Native SQLite, as on the iPhone | op-sqlite lists macOS ([README](https://github.com/OP-Engineering/op-sqlite)) |
| Sign-in, Touch ID | The web sign-in and cookie session; Touch ID through WebAuthn | The iPhone's Swift module (§4.8); LocalAuthentication in this mode unverified | A macOS build of that module (judgment) |
| Regional settings | Browsers don't expose macOS region overrides, so in-app settings decide (`i18n-l10n.md` §3.2) | Native, as on the iPhone | Native code, since `expo-localization` has no macOS |
| Icons, menus | Lucide, because SF Symbols are licensed for Apple-platform apps only (`design-system.md` §2.8); no web API adds menu-bar commands (judgment) | SF Symbols; menu-bar commands only as UIKit offers iPad apps (not checked) | SF Symbols; AppKit menus through native code |
| Verdict (judgment) | v1 candidate | v1 candidate | Not before 0.86 ships with Expo support |

### 4.3 Web

- **Expo UI.** Universal components (Button, List, Picker, TextInput and others) delegate to SwiftUI on iOS, but on the web they are JS implementations "not at the same quality bar as native yet" and "experimental" ([blog](https://expo.dev/blog/expo-ui-stable-sdk-56), [docs](https://docs.expo.dev/versions/latest/sdk/ui/universal/)). So iOS-native controls still need their own web implementations.
- **react-native-web.** On 2026-10-07, zoontek, necolas and Expo took over "active maintenance and development", and the repository moves to the React Foundation ([discussion](https://github.com/necolas/react-native-web/discussions/2872)). Releases resumed: 0.21.3 and 0.21.4 in September and October 2026, 11 months after 0.21.2.
- **React Strict DOM** is still at 0.0.55, published 2026-01-09 ([npm](https://www.npmjs.com/package/react-strict-dom)).
- **Expo Router web output.**
  - SDK 58 Router projects use `web.output: "server"` when it isn't set, and server output "renders HTML on each request instead of prerendering it during export" ([SDK 58](https://expo.dev/changelog/sdk-58)). That needs a running server per request, which an offline-first PWA can't rely on.
  - So set `web.output` to `single` (one `index.html`, an SPA) or `static` (HTML per route) ([app config](https://docs.expo.dev/versions/latest/config/app/)), and let the instance serve the files (judgment on the choice).
  - URLs come for free on both web and native (an advantage over A).

### 4.4 Apple surfaces: the Swift side of B

| Piece | Why it is Swift | Detail |
|---|---|---|
| Watch app and complications | ADR 0002; watchOS runs no JavaScript (§1) | `watchos-app.md` |
| WatchConnectivity module (Expo Modules API) | It owns `WCSession`, caches the glance and answers `sendMessage` natively, "because the JavaScript runtime may not be running when the watch wakes the app". `react-native-watch-connectivity` 2.0.0 (2026-03-20) only bridges WC to JavaScript ([npm](https://www.npmjs.com/package/react-native-watch-connectivity)) | `watchos-app.md` §3.3 |
| Notification service extension | Decrypts alert payloads before display | `apple-client-stack.md` §4.3 |
| Sign-in and Secure Enclave module | Browser sheet, device key, DPoP, Keychain | §4.8 |
| App Intents extension | Tap and cash capture without launching the app | Below |
| iOS widgets (optional) | `expo-widgets` renders Expo UI SwiftUI components in an isolated runtime "with no React hooks, app state, or asynchronous work" ([docs](https://docs.expo.dev/versions/latest/sdk/widgets/)); anything richer is WidgetKit in Swift | §4.7 |

**App Intents are a capture route, not an extra.** `bank-alternatives.md` §3 gives every iPhone user a `RecordWalletTap` intent fed by the Shortcuts Transaction trigger, and `watchos-app.md` plans cash entry through an intent.
- `expo-app-intents` 0.5.16 is alpha ([SDK 58](https://expo.dev/changelog/sdk-58)). Its intents are Swift compiled "into the app target", and JavaScript gets each invocation later, from a queue that "holds at most 100 invocations and drops the oldest to make room", with "at-least-once delivery" ([docs](https://docs.expo.dev/versions/latest/sdk/app-intents/)).
- Apple: code in an App Intents extension "runs only in the background", and the system "prefers to run a background app intent from your app extension" ([runtime behavior](https://developer.apple.com/documentation/appintents/configuring-the-runtime-behavior-of-your-app-intents)).
- **Design** (judgment): `RecordWalletTap` and `AddCash` live in an App Intents extension that neither writes the outbox nor calls the API. Writing the outbox would add a second writer to the local database (`sync-engine.md`, "One writer per local database"); calling the API fails whenever the tailnet or the network is down.
  - The extension appends each capture (UUIDv7, amount, card, merchant, time) to a durable handoff file in the app group, and marks the cached glance `pending`, so widgets show the tap at once.
  - The JS sync client, the database's only writer, reads the file from the app group container (`Paths.appleSharedContainers` in `expo-file-system`, [docs](https://docs.expo.dev/versions/latest/sdk/filesystem/)) and drains it into named mutations (`addManual@1`, source `wallet_tap` or `cash`), idempotent by UUID: at launch, on foreground and in `expo-background-task` runs, which BGTaskScheduler starts when it decides, no more often than every 15 minutes ([docs](https://docs.expo.dev/versions/latest/sdk/background-task/)).
  - The WatchConnectivity module drops WC-only watch writes in the same file, as `watchos-app.md` §2.5 proposes.
  - The cost: a capture reaches the server, and the partner's devices, only at the phone app's next sync.

### 4.5 RN component and styling libraries

| Library | Version (date) | Model | Web | Maintenance signal |
|---|---|---|---|---|
| [Tamagui](https://github.com/tamagui/tamagui) | 3.2.3 (2026-11-05) | Styled components plus an optimizing compiler: "a simple `div` alongside atomic CSS on the web, or a View with its style objects hoisted on native" | Yes | Majors 2.0 (2026-05-23) and 3.0 (2026-10-10): fast churn |
| [Unistyles](https://github.com/jpudysz/react-native-unistyles) | 3.8.1 (2026-11-03) | StyleSheet API on a C++/JSI core (Nitro), themes, breakpoints | Yes ("custom web parser") | Requires RN 0.85+ on the New Architecture; 1 open issue or PR |
| [NativeWind](https://github.com/nativewind/nativewind) | 4.2.7 stable; 5.0.0-rc.1 (2026-11-02) | Tailwind compiled for RN | Yes | v5 (Tailwind 4) is still a release candidate |
| [gluestack-ui](https://github.com/gluestack/gluestack-ui) | v5.0.0 (2026-06-25) | Copy-paste components on NativeWind | Yes | v5 is built on NativeWind v5, which is still RC |
| [Restyle](https://github.com/Shopify/restyle) | 2.4.5 (2025-03-19) | Typed theme props | Via react-native-web | No release for 20 months |

For B: Unistyles for styling plus your own components (judgment). It has the smallest API, a stable major, and theme objects that map directly to generated tokens.

### 4.6 Localization, tests, CI, agents

- **`Intl`.**
  - Hermes's docs (rewritten 2026-10-30) say the `Intl` API is "Not supported". A partial implementation calls Java on Android and Foundation on Apple platforms. "It is not built by default (`HERMES_ENABLE_INTL` is `OFF`)", and even where enabled, `RelativeTimeFormat`, `PluralRules` and `formatRange` "are missing or partial". Meta "cannot invest in it", the code is expected to move to `extensions/contrib`, and the advice is to "Use a polyfill such as FormatJS instead, accepting its start-up cost" ([Features.md](https://github.com/meta/hermes/blob/static_h/doc/Features.md)).
  - React Native's builds turn it on. RN downloads prebuilt Hermes from Maven ([hermes-utils.rb](https://github.com/facebook/react-native/blob/main/packages/react-native/sdks/hermes-engine/hermes-utils.rb)). Hermes's "RN Build Static Hermes" workflow builds the Apple slices with `-DHERMES_ENABLE_INTL:BOOLEAN=true` ([build-apple-framework.sh](https://github.com/meta/hermes/blob/static_h/utils/build-apple-framework.sh)), RN's own source build passes the same flag ([RN script](https://github.com/facebook/react-native/blob/main/packages/react-native/sdks/hermes-engine/utils/build-apple-framework.sh)), and the Android build passes `-DHERMES_ENABLE_INTL=True` ([build.gradle.kts](https://github.com/meta/hermes/blob/static_h/android/build.gradle.kts)). So B gets the partial, platform-backed `Intl`, not an undefined one.
  - On iOS it lacks `NumberFormat.formatToParts` (Android only), `signDisplay`, compact and engineering notation, and `DateTimeFormat`'s `numberingSystem` ([IntlAPIs.md](https://github.com/meta/hermes/blob/static_h/doc/IntlAPIs.md)).
  - For a money app with European formats, this means targeted FormatJS polyfills or a native formatter module (judgment); spike S6 measures the start-up cost.
  - [`react-native-localize`](https://github.com/zoontek/react-native-localize) 3.7.2 exposes `uses24HourClock()`, `getTimeZone()` and `getNumberFormatSettings()`.
- **Tests and CI** (choices are judgment; tools as of 2026-11-07). `factory-ci-quality.md` waits on this list to plan its `ui` job.

  | Layer | Tool | Runs on |
  |---|---|---|
  | Logic and components | Jest with React Native Testing Library | Linux |
  | Component images, iOS and web | [React Native Harness](https://github.com/callstackincubator/react-native-harness) 1.7.0 (Callstack, MIT, first UI release 2026-01): `render()`, `screen.screenshot()` and `toMatchImageSnapshot` on the iOS simulator and, through Playwright, the web. Fallback: an XCUITest target that opens each Storybook story by deep link and compares the screenshot with swift-snapshot-testing's `UIImage` strategy, which takes `precision` and `perceptualPrecision` ([source](https://github.com/pointfreeco/swift-snapshot-testing/blob/main/Sources/SnapshotTesting/Snapshotting/UIImage.swift)) | Hosted macOS, one pinned simulator |
  | Web pages and a11y | Playwright 1.65 `toHaveScreenshot` and axe-core over the react-native-web stories (Storybook `react-native-web-vite` [10.6.6](https://www.npmjs.com/package/@storybook/react-native-web-vite)) | Linux |
  | E2E | Maestro CLI [2.12.1](https://github.com/mobile-dev-inc/Maestro/releases). Its `assertScreenshot` passes at a 95% match by default ([docs](https://docs.maestro.dev/reference/commands-available/assertscreenshot)), too loose for component diffs | Hosted macOS |
  | Launch, hitches, iOS a11y | XCUITest: `XCTApplicationLaunchMetric`, `XCTHitchMetric` (iOS 26+) and `performAccessibilityAudit()` ([launch](https://developer.apple.com/documentation/xctest/xctapplicationlaunchmetric), [hitches](https://developer.apple.com/documentation/xctest/xcthitchmetric)) | Hosted macOS, main branch |
  | Watch and widgets | `ImageRenderer` plus an in-repo image diff, or XCUITest screenshots: swift-snapshot-testing has no watchOS support ([#705](https://github.com/pointfreeco/swift-snapshot-testing/issues/705), open; `watchos-app.md` §3.5). iOS widgets: swift-snapshot-testing through Prefire (`design-system.md` §6) | Hosted macOS |

  - **PR CI:** GitHub-hosted macOS runners, free on the public repo (`factory-ci-quality.md` §1), build the committed `ios/` with `xcodebuild` and run the macOS rows; Linux runs the rest.
  - **Release:** Xcode Cloud archives, as `factory-ci-quality.md` plans. Its image includes Homebrew, which a post-clone script uses to install tools such as CocoaPods ([Apple](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud)); here that script also installs Node and the JS dependencies, which the bundling phase needs (judgment; S4 tests it).
  - **EAS is not needed** with a committed `ios/`. It would be under CNG, where Xcode Cloud can't build, and Expo's free plan gives "Up to 15 Android and 15 iOS builds" a month on a low-priority queue, with workflows "Up to 60 minutes" ([pricing](https://expo.dev/pricing)).
  - CocoaPods remains Expo's "default and supported path" in SDK 58 ([SDK 58](https://expo.dev/changelog/sdk-58)). CocoaPods trunk stops accepting podspecs on 2026-12-02, which "will keep all existing builds working" ([CocoaPods](https://blog.cocoapods.org/CocoaPods-Specs-Repo/)).
- **Agents.** The [Expo MCP server](https://docs.expo.dev/eas/ai/mcp/) offers docs, simulator screenshots and taps, logs and EAS builds. Doc search needs a paid plan, and local iOS support covers simulators only. The committed `ios/` and the Swift pieces use Xcode's MCP tools and MobileBuildMCP (`factory-ci-quality.md` §4).

### 4.7 Native project: Continuous Native Generation or a committed `ios/` (gate 3)

Expo's default, Continuous Native Generation (CNG), generates "short-lived native projects … only when needed", and config plugins make every native change ([CNG](https://docs.expo.dev/workflow/continuous-native-generation/)). Watch, widget and extension targets then come from `@bacons/apple-targets`, whose blockers `watchos-app.md` §3.3 lists; rechecked on 2026-11-07:
- 5.0.0 (2026-07-17) is still the latest release and depends on `@expo/prebuild-config` ~55.0.6 ([npm](https://www.npmjs.com/package/@bacons/apple-targets)), while Expo is at 58.0.23. SDK 56 builds fail ([#196](https://github.com/EvanBacon/expo-apple-targets/issues/196), open), and the SDK 57 port is still a draft ([#210](https://github.com/EvanBacon/expo-apple-targets/pull/210)).
- The watch target has no Frameworks build phase, so SPM packages don't link ([#171](https://github.com/EvanBacon/expo-apple-targets/issues/171), open). Neither `design-system.md`'s `Packages/DesignSystem` nor `i18n-l10n.md`'s `Packages/Strings` can reach the watch.
- The watch's `CFBundleVersion` doesn't follow the app's, so App Store submissions fail with error 90379 ([#147](https://github.com/EvanBacon/expo-apple-targets/issues/147); fix [#148](https://github.com/EvanBacon/expo-apple-targets/pull/148) unmerged).

| | CNG with `@bacons/apple-targets` | Committed `ios/` (one prebuild, then Xcode) |
|---|---|---|
| Watch app, NSE, intents, widgets | Blocked today (above) | Ordinary Xcode targets that link local packages. Capacitor works this way by design: its Xcode project is a "source asset", checked "into source control" ([Capacitor](https://capacitorjs.com/docs/cordova)) |
| Xcode Cloud | Needs the generated project committed (`factory-ci-quality.md` §1), which defeats CNG; EAS instead | Works, with a post-clone script (§4.6) |
| App config | Applied at every prebuild | "Expo Build will not run Prebuild" once native folders exist (CNG page), and expo-doctor warns that fields such as `ios`, `plugins`, `icon`, `scheme` and `locales` stop syncing ([check](https://github.com/expo/expo/blob/main/packages/expo-doctor/src/checks/AppConfigFieldsNotSyncedToNativeProjectsCheck.ts)) |
| `i18n-l10n.md`'s per-target strings | Expo `locales` JSON for the app target | An `InfoPlist.xcstrings` for the app target, like every other target |
| Expo SDK upgrades | Regenerate | Apply each SDK's native diff "by copying and pasting or manually making changes" ([upgrade helper](https://docs.expo.dev/bare/upgrade/)) |
| Libraries' config plugins | Run at prebuild | Run once in a scratch prebuild, then port the diff (judgment) |

**Pick: commit `ios/`** (judgment, **medium**). It is the only path that builds and ships the watch today, and it keeps Xcode Cloud and `factory-ci-quality.md`'s plan. The price is a hand-applied native diff at every SDK upgrade, every few months. Return to CNG if `watchos-app.md`'s spike W7 passes on the chosen SDK with #147, #171 and #196 closed. Spike S4 settles the gate.

### 4.8 Sign-in, tokens and data on the device

This applies `native-apps-auth.md` and `apple-client-stack.md` §3.3 to B, and replaces this note's earlier "pairing step, tokens behind Face ID", which contradicted both.
- **Sign-in.** An ephemeral `ASWebAuthenticationSession` on the instance's `/oauth/authorize` with PKCE S256, `dpop_jkt` and the `.customScheme` callback. Then a ~15-minute access token and a rotating refresh token, both DPoP-bound to a Secure Enclave P-256 key (`native-apps-auth.md`, Recommendation 3).
- **What Expo covers**, as of SDK 58:
  - `expo-web-browser`'s `openAuthSessionAsync` uses that session on iOS, but through the initializer that the 27.3 SDKs deprecate for custom schemes, and the module lists no macOS (`native-apps-auth.md` §2, §10);
  - `expo-secure-store` stores Keychain values with `AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY` and an `accessGroup` ([source](https://github.com/expo/expo/blob/main/packages/expo-secure-store/src/SecureStore.ts)), but creates no keys and signs nothing;
  - npm has no maintained Secure Enclave signer: [`react-native-device-crypto`](https://www.npmjs.com/package/react-native-device-crypto) 0.1.7 dates from 2022, and [`react-native-secure-sign`](https://www.npmjs.com/package/react-native-secure-sign) is at 0.0.2.
- **So B writes one Swift Expo module, `Security`** (judgment): the session with the Callback API, the device key and DPoP proofs, the Keychain items and the LocalAuthentication app lock. It refreshes tokens natively, so the refresh token never enters JavaScript, which sees only access tokens. This is `native-apps-auth.md`'s gate S7, scored in §7 and tested in spike S7 here.
- **Keychain classes**, as in `apple-client-stack.md` §3.3: the refresh token `AfterFirstUnlockThisDeviceOnly` with no biometry flag, so background sync can read it; the device key with `privateKeyUsage`; the push-payload key shared with the NSE; a second key with `biometryCurrentSet`, only for step-up (`native-apps-auth.md` §10). Unlike `apple-client-stack.md`, keep the refresh token out of the extensions' access group, since no extension calls the API here (judgment, §4.4).
- **Local database.** It stays in the app's own container under the default Data Protection class, `completeUntilFirstUserAuthentication`, which background sync needs (`apple-client-stack.md` §3.1). Extensions read only the glance and the handoff file (§4.4).
  - Encrypt it as well. Backups can carry app files, and Apple calls the backup-exclusion flag "not a mechanism to guarantee those items never appear in a backup" ([Apple](https://developer.apple.com/documentation/foundation/optimizing-your-app-s-data-for-icloud-backup)). A key kept as `AfterFirstUnlockThisDeviceOnly` comes back only to the same device (`apple-client-stack.md` §3.1).
  - `expo-sqlite`'s `useSQLCipher` covers "Android, iOS, and macOS", but "If the app binary does not include SQLCipher, `PRAGMA key` has no effect and does not throw an error" ([docs](https://docs.expo.dev/versions/latest/sdk/sqlite/)), so a test must prove the cipher is on. PowerSync's React Native SDK encrypts through op-sqlite's SQLCipher, and its web SDK through SQLite3MultipleCiphers ([PowerSync](https://docs.powersync.com/client-sdks/advanced/data-encryption)).
- **No over-the-air JavaScript updates in v1** (judgment). EAS Update's code signing "is only available to accounts subscribed to the Production or Enterprise plans" ([docs](https://docs.expo.dev/eas-update/code-signing/)), from $199 a month ([pricing](https://expo.dev/pricing)), and any update channel lets its key holder change the code of every installed app that holds bank data, outside TestFlight. Leave `expo-updates` out, or set `updates.enabled` to false so builds "only use code and assets bundled at time of build" ([app config](https://docs.expo.dev/versions/latest/config/app/)). Expo's open updates protocol defines signed manifests for a self-hosted server, if OTA is ever wanted ([spec](https://docs.expo.dev/technical-specs/expo-updates-1/)).
- **npm supply chain.** Every dependency ships inside the signed app, native code included through autolinked pods (judgment). Use pnpm 12 with `minimumReleaseAge` (default 1,440 minutes), `blockExoticSubdeps` (default on), `trustPolicy: no-downgrade`, and build scripts "disallowed by default" unless listed in `allowBuilds` ([resolution](https://pnpm.io/settings/dependency-resolution), [build](https://pnpm.io/settings/build)). Add the lockfile, Dependabot and the rails of `factory-ci-quality.md`, and on the web CSP, Trusted Types and SRI (`web-client-sync.md` §4.4).
- **B against E** (judgment). B renders no HTML, so bank text can't turn into script; a compromised package runs with the app's privileges in both. E renders bank text in a WebView, where "malicious JavaScript code has the same privileges as the legitimate application code" ([RFC 10017](https://www.rfc-editor.org/rfc/rfc10017.html)), native bridge included.

## 5. C, D and E: their own renderers or WebViews

**C. Flutter 3.47.9** (Dart 3.13.7)
- **Design libraries.** Material and Cupertino were frozen in the framework and republished as the `material_ui` and `cupertino_ui` packages ([Flutter blog](https://flutter.dev/blog/decoupling-material-cupertino)).
- **iPhone.** "Support for iOS 26 'Liquid Glass' Design in Cupertino Widgets" has been open since 2025-06-10, at P3 with 89 comments ([#170310](https://github.com/flutter/flutter/issues/170310)).
- **Mac.** Flutter points to the third-party `macos_ui` package for the macOS look ([docs](https://docs.flutter.dev/platform-integration/macos/building)).
- **Web.**
  - It renders through CanvasKit/Skwasm on WebGL/WebGPU ([renderers](https://docs.flutter.dev/platform-integration/web/renderers)) and is "not suitable for static websites with text-rich flow-based content" ([FAQ](https://docs.flutter.dev/platform-integration/web/faq)).
  - Web accessibility is "not on by default". Users must press an invisible "Enable accessibility" button, or the app calls `ensureSemantics()` ([docs](https://docs.flutter.dev/ui/accessibility/web-accessibility)).
- **watchOS** isn't a Flutter platform ([supported platforms](https://docs.flutter.dev/reference/supported-platforms)).
- **Strong points:**
  - `alwaysUse24HourFormat` mirrors the iOS "24-Hour Time" setting ([API](https://api.flutter.dev/flutter/widgets/MediaQueryData/alwaysUse24HourFormat.html)), and dates and numbers are formatted by the `intl` package ([i18n](https://docs.flutter.dev/ui/internationalization));
  - golden tests ([matchesGoldenFile](https://api.flutter.dev/flutter/flutter_test/matchesGoldenFile.html)).

**D. Compose Multiplatform 1.12.2**
- **Stability.** iOS and Desktop (JVM) are Stable, Web (Kotlin/Wasm) is Beta, and Kotlin's watchOS target is Beta ([JetBrains](https://kotlinlang.org/docs/multiplatform/supported-platforms.html)).
- **iPhone fidelity** is still converging: 1.12 (2026-08-25) shipped "selection handles now match the style of iOS", the text-field "Select" menu item and Full Keyboard Access fixes ([notes](https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.12.0)).
- **Mac.** The desktop target is AWT/Swing-based (`ComposeWindow`, `SwingPanel`; same notes).
- **Swift interop.**
  - By default, Swift uses Kotlin through the Objective-C headers of a generated framework ([Objective-C interop](https://kotlinlang.org/docs/native-objc-interop.html)), on KMP's Stable iOS target ([platforms](https://kotlinlang.org/docs/multiplatform/supported-platforms.html)).
  - Swift export is opt-in, through a `swiftExport {}` block and a different Xcode build phase, and "currently in Alpha… breaking changes are expected" ([Kotlin docs](https://kotlinlang.org/docs/native-swift-export.html)).
  - Importing Objective-C libraries into Kotlin (cinterop) is Beta ([stability](https://kotlinlang.org/docs/components-stability.html)).
- **Tooling:** multiplatform `ComposeUiTest` ([docs](https://kotlinlang.org/docs/multiplatform/compose-test.html)), and an MCP server for Compose Hot Reload on desktop (1.12 notes).

**E. Web-first wrapped** (Capacitor 8.6, Tauri 2.12.5, Electron 45.2)
- **UI.** Everything is web content in a WebView: no system controls, no Liquid Glass, and Dynamic Type only as far as CSS reaches (judgment).
- **Tauri mobile.** Tauri 2.0 (2024-10-02) added iOS and Android, but its authors were "not completely happy about the developer experience" ([Tauri 2.0](https://v2.tauri.app/blog/tauri-20/)).
- **Biometrics.** Touch ID on the Mac works in both desktop wrappers.
  - tauri-plugin-biometric 2.5.0 (crates.io, 2026-10-14) added "macOS Touch ID support via LocalAuthentication framework", and its README now lists Android, iOS and macOS ([changelog](https://github.com/tauri-apps/plugins-workspace/blob/v2/plugins/biometric/CHANGELOG.md), [README](https://github.com/tauri-apps/plugins-workspace/blob/v2/plugins/biometric/README.md)). The [docs page](https://v2.tauri.app/plugin/biometric/) still says "Android and iOS".
  - Electron has `systemPreferences.canPromptTouchID()` and `promptTouchID()` on macOS ([docs](https://github.com/electron/electron/blob/main/docs/api/system-preferences.md)).
- **Upside** (judgment): one design system and one codebase for iPhone, Mac and web, which meets ADR 0002; the largest agent corpus; and Playwright and Storybook for everything.

## 6. F: a shared core under native UIs

**Status:** ruled out by ADR 0002, because its UIs are A's two codebases.

- **Rust.**
  - UniFFI ships first-party bindings for "Kotlin, Swift, Python and Ruby". JS/Wasm comes from the third-party [`uniffi-bindgen-react-native`](https://github.com/jhugman/uniffi-bindgen-react-native), now at a 0.31.0 pre-release (2026-09-25), which can generate a `wasm-bindgen` crate ([UniFFI README](https://github.com/mozilla/uniffi-rs)).
  - On watchOS, `aarch64-apple-watchos` is Tier 2, but `arm64_32-apple-watchos` (watches with 32-bit pointers) is Tier 3 ([rustc platform support](https://doc.rust-lang.org/nightly/rustc/platform-support.html)).
- **Kotlin.** The default bridge is the Objective-C framework on KMP's Stable iOS target, and Swift export is an opt-in Alpha (§5, D). The real costs are an extra language and the Beta watchOS target.
- **Swift on the web.**
  - The Wasm SDKs exist since Swift 6.2. The effort "started out as a community project", Embedded mode is "experimental", and the SDK must exactly match the toolchain ([swift.org](https://www.swift.org/documentation/articles/wasm-getting-started.html)).
  - JavaScriptKit's BridgeJS reached MVP in February 2026 ([forums](https://forums.swift.org/t/swift-for-wasm-february-2026-updates/85046)).
  - A future way to run Swift domain code in the browser, but not production-ready today (judgment).
- **What on-budget would share** (judgment): the budget engine and the sync client, both cheaper than an FFI core.
  - **Under B:** TypeScript already shares both across the iPhone, Mac and web apps. The Swift watch app needs neither, because it shows a server-built glance (`watchos-app.md`). Across languages, only the OpenAPI contract, JSON test vectors (inputs → safe-to-spend, category balances) and the token and string generators remain. PowerSync has React Native ([2.4.0](https://www.npmjs.com/package/@powersync/react-native)) and web ([2.6.0](https://www.npmjs.com/package/@powersync/web)) SDKs if the sync spike adopts it.
  - **Under A:** SQL views and queries kept in `.sql` files, executed by GRDB on Apple and by SQLite Wasm on the web, plus the same JSON test vectors. The sync client is PowerSync (Swift SDK with GRDB integration in alpha, `@powersync/web` 2.6.0) if the spike adopts it. Otherwise it is written twice against one protocol spec and checked with the deterministic simulator from `sync-engine.md`.

## 7. Scored comparison

**Gates** (revised twice):
1. **One UI codebase for the iPhone, Mac and web apps** (`PRODUCT.md` Stack, ADR 0002). A and F fail. **High**: an owner decision.
2. **iPhone feel ≥ 4.** The question asks for native-feeling UX every day, `PRODUCT.md` puts the daily short sessions on the iPhone or the watch, and ADR 0002 concedes a less native feel only on the Mac. C, D and E fail. **Medium**: the score is judgment, and it alone removes E, which leads B on the weighted total. Spike S2 turns it into measured pass criteria.
3. **A watch app, with the NSE, intents and local Swift packages, that builds and reaches TestFlight** (ADR 0002, `watchos-app.md` W7). C, D and E keep an Xcode project in the repo, so they pass; B passes only with a committed `ios/` (§4.7). **Medium-high**; S4 confirms it.

The first draft's gate (iPhone ≥ 4 and Mac ≥ 4) rested on a misreading: `PRODUCT.md` requires native SwiftUI only for the watch.

**Scores.** From 1 (poor) to 5 (best). The scores are judgment drawn from the findings above; the weights reflect the brief.
- After the fact-checks: Mac weight 4 → 3, because ADR 0002 accepts a less native Mac; web weight 3 → 4, for full parity offline and because the web is the one client every self-hoster can run.
- After the completeness review:
  - a security row (weight 3), since B and E differ on the WebView's script surface and on token custody (§4.8);
  - Apple integration now includes the browser-sheet sign-in with a Secure Enclave key and App Intents as a capture route. B drops 4 → 3: the Swift module, an alpha `expo-app-intents`, and target tooling blocked under CNG;
  - agent-friendliness counts the Swift glue that every shared option carries (§1, §4.4): B 4 → 3, E 5 → 4, C and D 3 → 2;
  - maturity and cost: B 3 → 2, for SDK upgrades applied by hand to the committed `ios/`, on top of the macOS fork.

| Criterion (weight) | A | B | C | D | E | F | Basis |
|---|---|---|---|---|---|---|---|
| iPhone feel: controls, navigation, text input, a11y, Dynamic Type, Liquid Glass (5) | 5 | 4 | 2 | 2 | 2 | 5 | System views (A, F); UIKit views + Expo UI, with a lag (B); own renderer (C, D); WebView (E) |
| Mac feel: menus, keyboard, windows (3) | 4 | 2 | 2 | 1 | 3 | 4 | SwiftUI + AppKit hatches; Dock web app or iPad app (B, §4.2 matrix); JVM (D) |
| Apple integration: Face ID / Touch ID, browser-sheet sign-in with a Secure Enclave key, alerts, watch link, App Intents; widgets as extras (3) | 5 | 3 | 3 | 3 | 3 | 4 | Direct in Swift (A); a Swift module, alpha intents, blocked CNG targets (B); plugins or custom native code (C, D, E); Rust on the watch is partly Tier 3 (F) |
| Web quality at full parity: a11y, performance, URLs, offline (4) | 4 | 3 | 2 | 2 | 5 | 4 | DOM + React Aria (A); react-native-web with experimental Expo UI web (B); self-drawn web UI, accessibility opt-in (C), Beta (D) |
| One strict design system from one token source (3) | 4 | 4 | 4 | 4 | 5 | 4 | Two implementations to keep in parity, or one plus a watch/widget subset |
| UI code genuinely shared (2) | 2 | 4 | 4 | 4 | 5 | 2 | A and F share within Apple only; B shares everything but the watch and iOS-only controls; E shares everything but the watch |
| Offline storage on every platform (2) | 4 | 4 | 3 | 3 | 3 | 4 | GRDB + SQLite Wasm; expo-sqlite or PowerSync, native + web |
| Security of tokens and bank data on devices (3) | 5 | 4 | 4 | 4 | 3 | 5 | Native only (A, F); no HTML, but third-party packages in-process (B: npm; C, D: pub.dev, Maven); a WebView DOM next to the native bridge (E) |
| Testing: unit, visual, E2E (2) | 5 | 3 | 4 | 3 | 4 | 3 | Mature tools on both sides (A); a young screenshot tool or an XCUITest harness on iOS (B, §4.6); FFI boundary (F) |
| Agent-friendliness (4) | 4 | 3 | 2 | 2 | 4 | 2 | Two mainstream stacks with MCP tooling (A); TS + six Swift pieces + native config + churn (B); Dart or Kotlin + Swift (C, D); three languages + FFI (F) |
| Maturity and cost for a team of one (4) | 4 | 2 | 3 | 2 | 3 | 2 | First-party APIs (A); SDK every few months, by hand in `ios/`, + macOS fork (B); Wasm beta (D) |
| **Weighted total (max 175)** | **149** | **113** | 99 | 90 | 124 | 126 | |
| **Gate 1: one UI codebase** | fail | pass | pass | pass | pass | fail | |
| **Gate 2: iPhone feel ≥ 4** | pass | pass | fail | fail | fail | pass | |
| **Gate 3: watch reaches TestFlight** | pass | pass, with `ios/` | pass | pass | pass | pass | |

- **Sensitivity.** B comes first only through the gates.
  - Without gate 2, E leads B under every re-weighting tried: by 11 points with these weights, 10 with the original ones (Mac 4, web 3), 15 with web weight 6, 9 with agent weight 2 or Mac weight 1, and 8 with security weight 6. B overtakes E only if iPhone feel weighs 11 or more.
  - Without gate 1, A leads B by 32 points or more under the same re-weightings.
  - So the decision rests on two owner choices, shared code (ADR 0002) and native feel on the iPhone, and on one fact S2 must establish: that B's iPhone feel is measurably better than E's. If the owner gives up native feel on the iPhone, or S2 fails, pick E.
- **Agent evidence is thin.** SwiftEval (28 problems, 44 models) reports a "significant LLM scores drop for problems requiring language-specific features" ([arXiv 2505.24324](https://arxiv.org/abs/2505.24324)). On SWE-Bench Mobile, built from a production Swift/Objective-C codebase, "even the best configurations achieve only 12% task success rate" ([arXiv 2602.09540](https://arxiv.org/abs/2602.09540)). Neither compares SwiftUI with React directly. The agent row is **low-medium** confidence.

## 8. One strict design system across React Native and SwiftUI

Written for B: React Native on the iPhone, Mac and web, SwiftUI on the watch and in widgets. Where A would differ, it says so. `design-system.md` is the detailed note on foundations, tiers and outputs.

### 8.1 Token pipeline

- **Source.** DTCG JSON in the repo. The Format, Color and Resolver modules 2025.10 were declared stable on 2025-10-28 as a W3C Community Group report, not a W3C Standard ([final report](https://www.w3.org/community/reports/design-tokens/CG-FINAL-format-20251028/)). The Resolver module expresses contexts such as `light`, `dark` and high contrast as "modifiers" ([Resolver](https://www.designtokens.org/tr/2025.10/resolver/)). `design-system.md` §1.3 sets the tiers and two modifiers, `appearance` and `platform`.
- **Build: Terrazzo 2.10**, aligned with `design-system.md` (TL;DR 1, §1.2).
  - Terrazzo 2.0.0 shipped "Full support for the DTCG v2025.10 spec, including resolvers" ([changelog](https://github.com/terrazzoapp/terrazzo/blob/main/packages/cli/CHANGELOG.md)), and it has built-in token lint.
  - Its Swift plugin is "still experimental" and emits only color asset catalogs ([Terrazzo](https://terrazzo.app/docs/integrations/swift)). So the Swift constants and the RN theme come from small in-repo plugins.
- **Fallback: Style Dictionary 5.8.0** (2026-11-03), run once per theme context.
  - It added DTCG 2025.10 piece by piece: structured colors in 5.3 (2026-02-09), dimension objects in 5.4 (2026-03-22) and durations in 5.7 (2026-10-16) ([releases](https://github.com/style-dictionary/style-dictionary/releases)). Its docs still say 2025.10 "does not have full support yet" ([DTCG page](https://styledictionary.com/info/dtcg/)), and resolver support is still open ([#1590](https://github.com/style-dictionary/style-dictionary/issues/1590)).
  - Its `ios-swift/*` formats default to `import UIKit` but take an `import` option ([formats](https://styledictionary.com/reference/hooks/formats/predefined/)). The built-in `color/ColorSwiftUI` and `size/swift/remToCGFloat` transforms emit SwiftUI `Color(red:…, opacity:)` and `CGFloat` values ([transforms](https://styledictionary.com/reference/hooks/transforms/predefined/)); only the `ios-swift` transform group defaults to `UIColor`.
  - So static SwiftUI token enums need only a custom transform list. Custom formats are needed for asset-catalog color sets with Any, Dark and high-contrast variants (there is no built-in xcassets format), dynamic light/dark colors and composite type roles. `design-system.md` §1.2 says Style Dictionary has no SwiftUI output; the built-in `color/ColorSwiftUI` transform contradicts that.
- **Same names everywhere** (judgment), using `design-system.md` §1.3's scheme: `sys.bg.base` → `tokens.bg.base` (RN) → `DS.Color.bgBase` (SwiftUI). Agents then translate one-to-one between the RN and SwiftUI components.

```text
design/tokens/                DTCG 2025.10 JSON + resolver (appearance: light|dark|lightHC|darkHC; platform: ios|macos|watchos|web)
design/components/            one spec per component (the contract)
tools/tokens/                 Terrazzo config + in-repo plugins (RN theme, Swift xcassets + constants)
l10n/                         one ICU message catalog → FormatJS catalogs + Localizable.xcstrings (i18n-l10n.md)
packages/design-system/       generated tokens.native.ts + Unistyles themes; RN components (.ios/.web variants where needed); stories
app/                          Expo app: iPhone + web (react-native-web); Mac later
app/ios/                      committed Xcode project: app, NSE, App Intents extension, watch targets (§4.7)
app/modules/                  local Expo modules in Swift: Security (§4.8), WatchLink (§4.4)
apple/Targets/                Swift sources: watch app and its widgets, NSE, App Intents extension
apple/Packages/DesignSystem/  generated Tokens.xcassets + Tokens.swift; watch and widget components; previews
apple/Packages/Strings/       generated Localizable.xcstrings + Swift accessors (i18n-l10n.md)
```

### 8.2 What maps one-to-one and what does not

| Foundation | React Native (iPhone, Mac, web) | SwiftUI (Watch, widgets) | Judgment |
|---|---|---|---|
| Color (semantic, light/dark/high contrast) | Generated Unistyles themes. On iOS, `DynamicColorIOS` with light, dark and high-contrast values; RN documents hex/rgb/hsl/hwb only, no `oklch()` or P3 (`design-system.md` §1.4) | Asset-catalog color sets → `Color.surfaceRaised` | 1:1 |
| Spacing, radius, sizes | Numbers in the theme (points on iOS, CSS pixels on the web) | `enum Space`; `@ScaledMetric` where tied to text | 1:1 |
| Type scale | Roles with `allowFontScaling` and `dynamicTypeRamp` for Dynamic Type on iOS; `fontVariant: ['tabular-nums']` for amounts ([Text](https://reactnative.dev/docs/text), [text style props](https://reactnative.dev/docs/text-style-props)) | Roles mapped to text styles, so Dynamic Type works; `monospacedDigit()` for amounts | Roles 1:1, sizes differ by platform |
| Typeface | System font: SF on Apple devices, the OS font elsewhere on the web, or one open font. Apple's font license limits SF to mock-ups for Apple-platform UIs and bars embedding (secondary summary; check the license in the [installer](https://developer.apple.com/fonts/)) | System font (SF Compact) | Owner decision |
| Surfaces, elevation | Shadow tokens; native tabs and `GlassView` (iOS 26+) for floating chrome on iOS; approximations on the web | System materials | Not 1:1: chrome is native on iOS and the watch |
| Controls | Own components on RN primitives; Expo UI SwiftUI controls on iOS where system behavior matters, with web versions in the same component | Wrap system controls with `ButtonStyle`, `ToggleStyle` and `LabelStyle`; don't redraw them | Same API and states, different chrome |
| Motion | Presets from duration and easing tokens | `Animation` presets | 1:1 values, platform curves |

Under A, the iPhone and the Mac use the SwiftUI column. Its web uses custom properties with `light-dark()` exposed through Tailwind `@theme`, a rem scale with `font-variant-numeric: tabular-nums`, and React Aria primitives styled with tokens.

### 8.3 Component contract and documentation

- **Spec file** (judgment). One Markdown file per component: purpose, anatomy, variants, states, tokens used, accessibility (labels, traits, keyboard), platform mapping (RN component, `.ios`/`.web` variants, watch SwiftUI type), examples and snapshot matrix. Agents implement both sides from it, and a CI script checks that every spec names an existing RN export and, for the watch subset, a SwiftUI type.
- **Catalogs.**
  - RN: Storybook for React Native 10.7 on device, and the same stories in a browser through Storybook's `react-native-web-vite` framework (10.6.6). Storybook's MCP addon gives agents `docs-list`, `docs-show` and `test-run`, but "Storybook's AI capabilities are currently in preview" ([Storybook AI](https://storybook.js.org/docs/ai)).
  - Apple: a `Catalog` target with previews for the watch components, and DocC for the package (judgment).
- **Shared documentation.** The specs stay the only document shared across platforms.

### 8.4 Enforcement: what makes it strict

- **Tokens.** Generated token files are committed; CI regenerates them and fails on any diff (judgment).
- **TypeScript.** ESLint's `react-native/no-color-literals` and `react-native/no-inline-styles` (eslint-plugin-react-native 5.0.0, [README](https://github.com/Intellicode/eslint-plugin-react-native)) outside `packages/design-system`, plus a small custom rule that rejects numeric spacing and font sizes there (judgment). Components accept token names, not raw values (`design-system.md` TL;DR 5).
- **Swift** (watch and widgets). SwiftLint regex `custom_rules` with `included`/`excluded` path filters ([README](https://github.com/realm/SwiftLint)) forbid `Color(red:`, `.font(.system(size:` and numeric `.padding(` outside `apple/Packages/DesignSystem`.
- **Web under A.**
  - Tailwind 4.3: `@theme { --*: initial; … }` removes every default utility, so "you'll only be able to use utility classes matching your custom theme variables" ([Tailwind](https://tailwindcss.com/docs/theme)).
  - Add a lint step that rejects arbitrary values such as `bg-[#…]` (judgment).
  - Or use CSS Modules with `stylelint-declaration-strict-value` 1.12 ([npm](https://www.npmjs.com/package/stylelint-declaration-strict-value)).
- **Visual tests.** Snapshot every component × light/dark × text size × EN/FR:
  - web: the react-native-web stories with Playwright 1.65 `toHaveScreenshot` ([docs](https://playwright.dev/docs/test-snapshots)) or Storybook visual tests ([docs](https://storybook.js.org/docs/writing-tests/visual-testing));
  - iOS: React Native Harness image snapshots on one pinned simulator, or the XCUITest fallback with swift-snapshot-testing's `UIImage` strategy (§4.6);
  - watch: `ImageRenderer` plus an in-repo image diff, or XCUITest screenshots, because swift-snapshot-testing has no watchOS support (#705; `watchos-app.md` §3.5). iOS widgets keep swift-snapshot-testing.

### 8.5 Figma and Claude Design

- **Figma.**
  - Variables support modes for light/dark ([guide](https://help.figma.com/hc/en-us/articles/15339657135383-Guide-to-variables-in-Figma)).
  - Native DTCG import and export are documented in Figma's help center (updated 2026-11-05). "Import modes" requires a JSON file in the DTCG format, with colors in sRGB or HSL only and dimensions in `px` only. "Export modes" writes a JSON file for one mode or a whole collection ([Modes for variables](https://help.figma.com/hc/en-us/articles/15343816063383-Modes-for-variables)).
  - Code Connect recommends template files (`.figma.ts`): "The recommended way to use Code Connect CLI is with template files". The React (and React Native), HTML, SwiftUI and Compose parsers are now "Legacy framework-specific APIs" ([Code Connect](https://developers.figma.com/docs/code-connect/)).
  - It needs "a Dev or Full seat on the Professional, Organization, and Enterprise plans", and "On Professional plans, you can have one active framework at a time" (same page). The pricing page lists Professional's Code Connect as "One active library and one framework", and higher plans as "Unlimited libraries and multiple frameworks" ([pricing](https://www.figma.com/pricing/)).
  - Cost per seat per month (USD, [pricing](https://www.figma.com/pricing/)): Professional $16 Full or $12 Dev; Organization $55 Full or $25 Dev, billed annually. Under B, mapping only the RN components fits Professional. Mapping the SwiftUI watch components too needs Organization, where a developer who only uses Code Connect needs a $25 Dev seat, not a $55 Full seat.
- **Claude Design.**
  - It launched as an Anthropic Labs research preview on 2026-04-17 ([Anthropic](https://www.anthropic.com/news/claude-design-anthropic-labs)). It is now "one of the templates you can start an artifact from": in chats, in the Artifacts tab and in Claude Code (`/design`, and `/design-sync` to bring in design systems "from a GitHub repo"), on plans from Free to Enterprise. The standalone claude.ai/design "closes on December 14, 2026" ([support](https://support.claude.com/en/articles/14604416-get-started-with-claude-design)).
  - Exports: zip, PDF, PPTX, Google Slides and standalone HTML, plus sending designs to Canva, Gamma, Miro, Vercel and Wix, and a handoff to Claude Code (same page). Native or SwiftUI output isn't mentioned.
  - So its prototypes arrive as HTML. Under B, the RN components are ported from them, and the SwiftUI watch side from the spec and the prototype (judgment).
  - Keep tokens and specs in the repo as the single truth, so Figma and Claude Design both read from it (judgment).

## 9. Localization and regional formats

| Concern | SwiftUI (A; the watch in every option) | Web (A, E; B in a browser) | RN / Hermes (B on iPhone and Mac) | Flutter (C) |
|---|---|---|---|---|
| Strings | String Catalogs: plurals and device variants ([Apple](https://developer.apple.com/documentation/xcode/localizing-and-varying-text-with-a-string-catalog)) | Lingui 6.12, i18next 27 or react-intl 12 (npm) | The same JS catalogs, plus `.xcstrings` for the Swift watch and extensions, both generated from one catalog (`i18n-l10n.md`) | ARB + gen-l10n |
| User's 12/24-hour and first-weekday choices | `hourCycle` returns the user's override; `firstDayOfWeek` ([Apple](https://developer.apple.com/documentation/foundation/locale/firstdayofweek)) | Locale defaults only (judgment), so this needs an in-app setting. `Intl.Locale.getWeekInfo()` in Chrome 130, Firefox 153, Safari 17 ([MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/getWeekInfo)) | `react-native-localize` | `alwaysUse24HourFormat` |
| Time zones | Foundation `TimeZone` and format styles | `Intl` + `Temporal` polyfill (Safari lacks `Temporal`) | Hermes's partial `Intl`, FormatJS polyfills where it falls short, and a `Temporal` polyfill (not verified on Hermes) | Not verified |
| Money | Currency format styles | `Intl.NumberFormat` (`currency`, `signDisplay`, `formatToParts`) | Currency formatting works, but iOS lacks `formatToParts` and `signDisplay` (§4.6): FormatJS polyfill or a native formatter | `intl` `NumberFormat` |

Recommendations (judgment; `i18n-l10n.md` has the details):
- **A synced "regional profile"** in the account: language, region, hour cycle, first weekday and currency display. It defaults to the device on Apple and to the browser on the web. Express it with Unicode locale keys: Apple's `Locale` documents `hc` and `fw` (`en-US@hc=h23`, `en-US@fw=mon`), and `Intl` takes the hour cycle as `fr-FR-u-hc-h23` (`Intl.Locale.hourCycle` in every engine, per MDN data). On the web, keep the first weekday as an app setting.
- **Money.** Store integer minor units plus the ISO 4217 code (`apple-client-stack.md` §2.2). Format with each platform's formatter, and snapshot-test EUR, CHF, GBP, SEK, DKK, PLN, CZK and HUF in French and English. Under B that covers three formatters: Hermes plus FormatJS on the iPhone, the browsers, and Foundation on the watch. Expect small differences between their CLDR data, such as the spaces in French grouping.
- **Strings.** Keep one key set in the repo, generate `.xcstrings` and the web catalogs from it, and make CI fail on missing keys or plural forms.

## 10. Offline storage, tests and agent loop by stack

| Stack | Local store | Unit / visual / E2E | Agent tooling |
|---|---|---|---|
| A, Apple | GRDB (iOS, macOS, watchOS), app group shared with widgets | Swift Testing / swift-snapshot-testing / XCUITest | Xcode 27 MCP, MobileBuildMCP, `swift test` without a simulator (`factory-ci-quality.md`) |
| A, web | SQLite Wasm (`opfs-sahpool` behind one leader worker, or `opfs-wl` for multi-tab concurrency) or PowerSync web | Vitest 5.1 / Playwright screenshots, Storybook / Playwright 1.65 + axe-core 4.14 | Storybook MCP (preview), headless browsers |
| B | `expo-sqlite`: native with SQLCipher and FTS; web via Wasm with COOP/COEP, whose default build has no FTS; `withExclusiveTransactionAsync` unsupported on web ([docs](https://docs.expo.dev/versions/latest/sdk/sqlite/)). Or PowerSync's React Native (op-sqlite) and web SDKs | Jest + RNTL / React Native Harness on iOS and web, Playwright over the web stories / Maestro (§4.6) | Expo MCP; Xcode MCP for `ios/` |
| Watch (every option) | A small snapshot file of the glance; no sync engine (`watchos-app.md` §2.9) | Swift Testing / `ImageRenderer` diffs or XCUITest screenshots (no watchOS in swift-snapshot-testing) | Xcode 27 MCP |
| C | Plugins (not verified) | Widget tests / goldens / integration tests ([docs](https://docs.flutter.dev/testing/overview)) | Not verified |
| D | Kotlin libraries (not verified) | `ComposeUiTest` | Compose Hot Reload MCP (desktop) |
| E | Web storage inside the WebView; SQLite plugins | Web tooling | Best corpus |

## Recommendation for on-budget

**Decision: B, Expo (React Native) for the iPhone, web and Mac apps, with a native SwiftUI watch app and a committed `ios/` project.** It is the only strategy that passes all three gates: one UI codebase for iPhone, Mac and web (ADR 0002), native feel on the iPhone, and a watch that ships. **Confidence: medium.** The main uncertainties are whether B's iPhone feel beats E's by S2's measured criteria, the quality of react-native-web at full parity (S2, S3), the native project and watch targets (S4), and the Swift modules (S7). E has the higher weighted total and is the fallback if S2 fails or the owner gives up native feel on the iPhone. A scores highest but needs ADR 0002 reversed.

| Layer | Choice | Trade-off accepted |
|---|---|---|
| iPhone | Expo SDK 58 (RN 0.88, New Architecture, Hermes V1); Expo Router with native tabs; Expo UI SwiftUI controls where system behavior matters | Apple's changes arrive through Expo; an SDK upgrade every few months |
| Native project | One `expo prebuild`, then a committed `ios/` owned in Xcode; CNG again only if W7 passes (§4.7) | Each SDK's native diff applied by hand; app config native fields stop syncing |
| Web | The same Expo app through react-native-web, with `web.output` set to `single` or `static`, installed as a PWA | Expo UI's web components are experimental, so iOS-only controls need web versions in the same component |
| Mac | The Dock web app or the iPad app on the Mac first, chosen from the §4.2 matrix; react-native-macos when Expo supports it again (≥ 0.86) and Unistyles runs on it | The least native client, as ADR 0002 accepts: web limits (storage, regional overrides, menus) or an iPad layout |
| Watch and widgets | SwiftUI watch app + WidgetKit extension in `ios/`, fed by a server-built glance over WatchConnectivity through a native Swift module (`watchos-app.md`); iOS widgets through `expo-widgets` or WidgetKit if wanted | A second, small component set in Swift |
| Capture | `RecordWalletTap` and `AddCash` in a Swift App Intents extension, through a handoff file drained by the JS sync client (§4.4) | A capture reaches the server at the phone app's next sync |
| Sign-in and device security | `native-apps-auth.md`'s browser-sheet passkey sign-in, PKCE and DPoP in one Swift `Security` module; SQLCipher database; Keychain classes from `apple-client-stack.md` §3.3; no OTA updates; pnpm release-age and build-script allowlists (§4.8) | A Swift module to own; every JavaScript fix ships as a TestFlight build |
| Tests and CI | Jest + RNTL, React Native Harness screenshots on iOS and web, Playwright, Maestro, XCUITest metrics; GitHub-hosted macOS for PRs, Xcode Cloud for releases, no EAS (§4.6) | A young screenshot tool, with an XCUITest fallback |
| Tokens | DTCG 2025.10 in the repo; Terrazzo 2.10 with in-repo RN and Swift plugins, as in `design-system.md`; Style Dictionary 5.8 as the fallback | A few hundred lines of plugins to own |
| Component contract | One spec per component; RN implementation with `.ios`/`.web` variants where needed; SwiftUI for the watch subset; parity check in CI; image snapshots on web, iOS and the watch (§8.4) | Spec-first discipline; re-recording after OS and browser updates |
| Strings and formats | One ICU catalog → FormatJS catalogs + `.xcstrings` (`i18n-l10n.md`); a synced regional profile; FormatJS polyfills for Hermes's iOS gaps; `Temporal` polyfill | Polyfill start-up cost; locale snapshot tests across three formatters |
| Shared logic | TypeScript domain code and sync client shared by the iPhone, Mac and web apps; OpenAPI-generated client; JSON test vectors for the server and the clients; PowerSync's RN and web SDKs if adopted | The watch shares no domain code and reads a glance |
| Design tools | The repo is the truth; Figma or Claude Design read from it | Code Connect for RN alone fits Figma Professional; adding SwiftUI needs Organization |

**Build order** (judgment):
1. The committed `ios/` with the app, watch, NSE and App Intents targets building in CI and reaching TestFlight through Xcode Cloud (S4), and the `Security` module signing in (S7).
2. Tokens and about ten core components in RN (iPhone and web), plus the watch subset in SwiftUI, with image snapshots.
3. The daily loop (inbox and safe-to-spend) on the iPhone and the web together, offline included, since they share the code.
4. The watch glance, actionable alerts and the capture intents.
5. The Mac: the Dock web app or the iPad app on the Mac, then react-native-macos when Expo supports it.
6. iOS widgets, if wanted.

**Trade-offs the owner must accept:**
- One TypeScript UI codebase for the iPhone, Mac and web, and a second, smaller SwiftUI codebase for the watch and any widgets, plus four Swift pieces on the iPhone side.
- An iPhone feel close to SwiftUI but not equal: Apple's changes reach B through Expo, and some system controls need Expo UI wrappers with separate web versions.
- A committed `ios/` whose native changes are merged by hand at each SDK upgrade, until CNG can build the watch.
- A Mac app that starts as a web or iPad app, and later rides a fork that trails React Native core.
- A partial `Intl` on iOS, patched with polyfills that cost start-up time.
- No over-the-air updates: every JavaScript fix waits for a TestFlight build.
- Agents must keep the RN components and the SwiftUI watch subset in sync. The spec files, snapshot tests and CI parity checks are the price of "strict".
- No Android app is planned, but B would make one cheap if it is ever wanted (judgment).

**Revisit:**
- E, if spike S2 fails its pass criteria: the owner prefers E or can't tell the two apart in the blind test, B misses two or more iOS targets that E meets, or react-native-web fails the parity checklist;
- CNG, once `@bacons/apple-targets` supports the chosen SDK and closes #147, #171 and #196 (W7);
- the Mac path when react-native-macos ships 0.86 or later and Expo lists macOS again (PRs #3036, #3100, #3104 and #3105 were still open on 2026-11-07);
- A, only if the owner reverses ADR 0002, for example after S2 shows the Mac or the iPhone falling short (judgment).

## Open questions and spikes

**Spikes** (≤ 2 days each; pass means built by agents and checked from the CLI):
- **S1 Token pipeline and three components.** `Button`, `AmountText` and `CategoryRow` in RN (iPhone and web) and SwiftUI (watch), in light and dark, EN and FR, at three text sizes, with image snapshots in CI (React Native Harness on iOS and web, `ImageRenderer` diffs on the watch). Pass: one token edit reaches both sides with no manual change.
- **S2 Feel and parity, with E as the control** (decides gate 2).
  - Build the review inbox from the same specs and tokens in Expo with Expo UI (B) and in Capacitor (E). Run both on an iPhone 11 (A13), the oldest class that runs iOS 26, and on a current iPhone. Run B on the web, as the iPad app on the Mac and as the Safari Dock web app; react-native-macos waits for 0.86 (§4.2).
  - Measure both builds with one XCUITest suite:
    - launch to first frame (`XCTApplicationLaunchMetric`) with the FormatJS and Temporal polyfills loaded: at most 400 ms on the current iPhone, Apple's stated goal ([WWDC19](https://developer.apple.com/videos/play/wwdc2019/423/)), with the iPhone 11 figure recorded;
    - hitches while scrolling 500 inbox rows (`XCTHitchMetric`): at most 10 ms/s, Apple's "good" band ([Apple](https://developer.apple.com/documentation/xcode/understanding-hitches-in-your-app));
    - input: no dropped keystrokes typing "1 234,56" on a French keyboard, and tap-to-feedback within 100 ms on a 120 fps recording (judgment);
    - accessibility: no `performAccessibilityAudit()` issues, a scripted VoiceOver pass that reaches every row action and reads amounts correctly in French, and no clipped amounts at AX3.
  - Blind test: both builds on the owner's iPhone under neutral names, a week each in random order; the owner picks one for daily use without knowing which is which.
  - react-native-web checklist in Safari, Chrome and Firefox: no serious or critical axe-core issues; full keyboard use with visible focus; a URL per screen with working back and forward; reflow at 320 CSS px and text at 200% (WCAG 2.2 1.4.10, 1.4.4); selectable amounts; French decimal input; an offline reload; INP at or under 200 ms at p75 ([web.dev](https://web.dev/articles/inp)).
  - **Pass** (judgment): B meets every iOS target, is no worse than E on launch and hitches, wins the blind test, and passes the web checklist. One missed target means fix and re-run; the Revisit list says when to switch to E. Record agent time and defects for both builds.
- **S3 Offline web in Safari.** The Expo web build with `expo-sqlite` or PowerSync web on iOS and macOS Safari, as a tab, a Home Screen app and a Dock app, with `web.output` set to `single` and then `static`. Check persistence grants, eviction and two open tabs, comparing `opfs-sahpool` behind one leader worker (`web-client-sync.md` §2) with `opfs-wl`. Leave the Dock app untouched for eight days of Safari use, then check whether OPFS survived ITP (§4.2).
- **S4 Watch and native project** (decides gate 3; includes `watchos-app.md` W7).
  - CNG path (W7): generate `watch` and `watch-widget` targets with `@bacons/apple-targets` on SDK 58, link one local Swift package, and upload to TestFlight, which checks #147, #171 and #196.
  - Committed path: add the same targets, the NSE and the App Intents extension in Xcode, link `Packages/DesignSystem` and `Packages/Strings`, archive in Xcode Cloud with the post-clone script, and upload to TestFlight.
  - Pass: a TestFlight build whose watch app installs from the iPhone. Keep CNG only if its path passes.
  - Then the watch slice: the server-built glance over WatchConnectivity, answered by the native module while the JS runtime is down, and watch writes through the handoff file (§4.4). No sync engine runs on the watch: watchOS blocks WebSocket and stream tasks (TN3135), and the watch can't reach a Tailscale-only instance on its own (`watchos-app.md`). Include how the watch is enrolled.
- **S5 Typeface.** System fonts per platform against one open font everywhere: Dynamic Type, French text length, tabular figures and the web fallback on Windows and Android.
- **S6 Regional profile.** Round-trip 12/24-hour, first weekday and number locale on the iPhone (Hermes `Intl` + FormatJS), the web and the watch. Check currency output for eight European currencies in FR and EN, and measure the polyfills' start-up cost on an iPhone.
- **S7 Swift modules** (`native-apps-auth.md` S7, `bank-alternatives.md` spike 2).
  - `Security`: an ephemeral `.customScheme` session on a passkey page, a Secure Enclave key signing DPoP proofs, and the refresh token in the Keychain with the §4.8 classes, refreshed natively in the background. On an iPhone and in the iPad app on the Mac, where LocalAuthentication and the session are unverified.
  - Capture: with the app killed, a Shortcuts Transaction automation runs `RecordWalletTap` from the App Intents extension; the next launch drains it into the outbox once, and a replayed file adds no duplicate.
  - SQLCipher: the database file is unreadable without its key, and a build without SQLCipher fails the test.

**Open questions for the owner:**
1. Which Mac client for v1, from the §4.2 matrix: the Dock web app (no new target, but web limits on storage, regional overrides, icons and menus) or the iPad app on the Mac (native, but an iPad layout)? react-native-macos isn't an option before 0.86.
2. If S2 fails its pass criteria, is native feel on the iPhone still worth B's extra cost, or should the project take E?
3. How do self-hosters get the native apps: one shared build that asks for a server URL, or their own signed builds (`native-apps-auth.md` open question 1)? Sign-in no longer depends on the answer (§4.8).
4. Which design tool is the source of truth, Figma (which plan?) or Claude Design, given that the repo's tokens and specs stay final?
5. System fonts on each platform, or one brand typeface everywhere?
6. Watch scope: a glance and quick add only? Independent of the iPhone?
7. Apple platforms follow the system's regional settings. Should in-app overrides exist there too, as they must on the web?
8. With TypeScript clients on the iPhone, Mac and web, does the Swift backend of `apple-client-stack.md` §4.1 still beat TypeScript?
9. No over-the-air updates in v1, so every JavaScript fix waits for a TestFlight build (§4.8): acceptable?

**Not verified:**
- react-native-macos New Architecture status, and when 0.86 ships;
- Expo with Mac Catalyst;
- LocalAuthentication, `ASWebAuthenticationSession` and menu-bar commands in an RN iPad app running on a Mac;
- whether a macOS Dock web app is exempt from ITP's 7-day deletion, as Home Screen apps are. WebKit exempts the domains its client marks, such as a `standaloneApplicationURL` or persisted sites ([config](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/WebsiteData/Cocoa/WebsiteDataStoreCocoa.mm), [ITP store](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/Classifier/ResourceLoadStatisticsStore.cpp)), and Safari's own code isn't public;
- that Xcode Cloud archives an Expo app from a committed `ios/` with a post-clone script (S4);
- React Native Harness's image snapshots on a pinned simulator in CI, and how stable its baselines are (1.7.0; first UI release 2026-01);
- which `web.output` default a new SDK 58 Router project gets: the app config reference says `single`, the SDK 58 changelog says `server`;
- whether Storybook's MCP addon works with the `react-native-web-vite` framework;
- whether `temporal-polyfill` runs on Hermes;
- Copilot Money's web feature parity;
- browsers exposing the OS 12/24-hour override;
- Flutter time-zone handling;
- Compose web rendering and accessibility (secondary sources only);
- Compose and Flutter storage libraries;
- the exact current SF font license text.

## Sources

Accessed 2026-11-07.

- **Owner decisions:** [`PRODUCT.md`](../../PRODUCT.md) · [ADR 0001](../adr/0001-self-hosted-single-owner-instances.md) · [ADR 0002](../adr/0002-one-ui-codebase-native-watch.md) · [ADR 0003](../adr/0003-nixos-just-tailscale-infrastructure.md)
- **Apple, platform:** [watchOS apps](https://developer.apple.com/documentation/watchos-apps/building_a_watchos_app) · [independent watch apps](https://developer.apple.com/documentation/watchos-apps/creating-independent-watchos-apps) · [TN3135 low-level networking on watchOS](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos) · [watchOS 27](https://developer.apple.com/watchos/whats-new/) · [WatchConnectivity](https://developer.apple.com/documentation/watchconnectivity) · [WKWebView](https://developer.apple.com/documentation/webkit/wkwebview) · [JavaScriptCore](https://developer.apple.com/documentation/javascriptcore) · [WidgetKit](https://developer.apple.com/documentation/widgetkit) · [widget extension](https://developer.apple.com/documentation/widgetkit/creating-a-widget-extension) · [WidgetKit updates](https://developer.apple.com/documentation/updates/widgetkit) · [ActivityKit](https://developer.apple.com/documentation/activitykit) · [App Intents](https://developer.apple.com/documentation/appintents) · [App Intents updates](https://developer.apple.com/documentation/updates/appintents) · [SwiftUI updates](https://developer.apple.com/documentation/updates/swiftui) · [Liquid Glass newsroom](https://www.apple.com/newsroom/2025/06/apple-introduces-a-delightful-and-elegant-new-software-design/) · [SupportedPlatform](https://developer.apple.com/documentation/packagedescription/supportedplatform) · [Font.custom relativeTo](https://developer.apple.com/documentation/swiftui/font/custom%28_:size:relativeto:%29) · [ScaledMetric](https://developer.apple.com/documentation/swiftui/scaledmetric) · [iOS apps on Mac](https://developer.apple.com/documentation/apple-silicon/running-your-ios-apps-in-macos) · [adapting iOS code](https://developer.apple.com/documentation/apple-silicon/adapting-ios-code-to-run-in-the-macos-environment) · [Mac Catalyst](https://developer.apple.com/documentation/uikit/mac-catalyst) · [associated domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains) · [Associated Domains entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.associated-domains) · [Apple fonts](https://developer.apple.com/fonts/) · [App Intents runtime behavior](https://developer.apple.com/documentation/appintents/configuring-the-runtime-behavior-of-your-app-intents) · [first app intent](https://developer.apple.com/documentation/appintents/creating-your-first-app-intent) · [iCloud backup data](https://developer.apple.com/documentation/foundation/optimizing-your-app-s-data-for-icloud-backup) · [TestFlight on Apple silicon Macs](https://developer.apple.com/help/app-store-connect/test-a-beta-version/test-iphone-and-ipad-apps-on-macs-with-apple-silicon) · [Xcode Cloud dependencies](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud) · [hitches](https://developer.apple.com/documentation/xcode/understanding-hitches-in-your-app) · [XCTHitchMetric](https://developer.apple.com/documentation/xctest/xcthitchmetric) · [XCTApplicationLaunchMetric](https://developer.apple.com/documentation/xctest/xctapplicationlaunchmetric) · [WWDC19 423 app launch](https://developer.apple.com/videos/play/wwdc2019/423/)
- **Apple, localization:** [Locale.current](https://developer.apple.com/documentation/foundation/locale/current) · [hourCycle](https://developer.apple.com/documentation/foundation/locale/hourcycle-swift.property) · [firstDayOfWeek](https://developer.apple.com/documentation/foundation/locale/firstdayofweek) · [String Catalogs](https://developer.apple.com/documentation/xcode/localizing-and-varying-text-with-a-string-catalog)
- **Swift libraries:** [GRDB Package.swift](https://github.com/groue/GRDB.swift/blob/master/Package.swift) · [PowerSync Swift Package.swift](https://github.com/powersync-ja/powersync-swift/blob/main/Package.swift) · [SwiftLint](https://github.com/realm/SwiftLint) · [swift-snapshot-testing](https://github.com/pointfreeco/swift-snapshot-testing) · [its UIImage strategy](https://github.com/pointfreeco/swift-snapshot-testing/blob/main/Sources/SnapshotTesting/Snapshotting/UIImage.swift) · [#705 watchOS](https://github.com/pointfreeco/swift-snapshot-testing/issues/705) · [Skip](https://github.com/skiptools/skip) · [Swift Wasm getting started](https://www.swift.org/documentation/articles/wasm-getting-started.html) · [Swift for Wasm Feb 2026](https://forums.swift.org/t/swift-for-wasm-february-2026-updates/85046)
- **Expo and React Native:** [SDK 58](https://expo.dev/changelog/sdk-58) · [SDK 57](https://expo.dev/changelog/sdk-57) · [SDK 56](https://expo.dev/changelog/sdk-56) · [app config `web.output`](https://docs.expo.dev/versions/latest/config/app/) · [platform-specific modules](https://docs.expo.dev/router/advanced/platform-specific-modules/) · [Expo UI stable](https://expo.dev/blog/expo-ui-stable-sdk-56) · [Expo UI](https://docs.expo.dev/versions/latest/sdk/ui/) · [universal components](https://docs.expo.dev/versions/latest/sdk/ui/universal/) · [Expo UI SwiftUI guide](https://docs.expo.dev/guides/expo-ui-swift-ui/) · [expo-widgets](https://docs.expo.dev/versions/latest/sdk/widgets/) · [expo-glass-effect](https://docs.expo.dev/versions/latest/sdk/glass-effect/) · [native tabs](https://docs.expo.dev/router/advanced/native-tabs/) · [expo-sqlite](https://docs.expo.dev/versions/latest/sdk/sqlite/) · [additional platforms](https://docs.expo.dev/modules/additional-platform-support/) · [Expo MCP](https://docs.expo.dev/eas/ai/mcp/) · [EAS E2E](https://docs.expo.dev/eas/workflows/examples/e2e-tests/) · [RN blog](https://reactnative.dev/blog) · [RN Text](https://reactnative.dev/docs/text) · [RN text style props](https://reactnative.dev/docs/text-style-props) · [react-native-macos releases](https://github.com/microsoft/react-native-macos/releases) · [react-native-macos PRs](https://github.com/microsoft/react-native-macos/pulls) · [react-native-web discussion #2872](https://github.com/necolas/react-native-web/discussions/2872) · [react-strict-dom](https://www.npmjs.com/package/react-strict-dom) · [expo-apple-targets](https://github.com/EvanBacon/expo-apple-targets) · [react-native-watch-connectivity](https://www.npmjs.com/package/react-native-watch-connectivity) · [react-native-localize](https://github.com/zoontek/react-native-localize) · [Hermes Features](https://github.com/meta/hermes/blob/static_h/doc/Features.md) · [Hermes Intl](https://github.com/meta/hermes/blob/static_h/doc/IntlAPIs.md) · [Hermes build-apple-framework.sh](https://github.com/meta/hermes/blob/static_h/utils/build-apple-framework.sh) · [Hermes Android build](https://github.com/meta/hermes/blob/static_h/android/build.gradle.kts) · [RN hermes-utils.rb](https://github.com/facebook/react-native/blob/main/packages/react-native/sdks/hermes-engine/hermes-utils.rb) · [RN Hermes build script](https://github.com/facebook/react-native/blob/main/packages/react-native/sdks/hermes-engine/utils/build-apple-framework.sh) · [Tamagui](https://github.com/tamagui/tamagui) · [Unistyles](https://github.com/jpudysz/react-native-unistyles) · [NativeWind](https://github.com/nativewind/nativewind) · [gluestack-ui](https://github.com/gluestack/gluestack-ui) · [Restyle](https://github.com/Shopify/restyle) · [eslint-plugin-react-native](https://github.com/Intellicode/eslint-plugin-react-native) · [Maestro](https://github.com/mobile-dev-inc/Maestro/releases) · [Maestro assertScreenshot](https://docs.maestro.dev/reference/commands-available/assertscreenshot) · npm: [@storybook/react-native-web-vite](https://www.npmjs.com/package/@storybook/react-native-web-vite), [@powersync/react-native](https://www.npmjs.com/package/@powersync/react-native)
- **Expo native project, security, tests:** [CNG](https://docs.expo.dev/workflow/continuous-native-generation/) · [native upgrade helper](https://docs.expo.dev/bare/upgrade/) · [expo-doctor unsynced-fields check](https://github.com/expo/expo/blob/main/packages/expo-doctor/src/checks/AppConfigFieldsNotSyncedToNativeProjectsCheck.ts) · [local Expo modules](https://docs.expo.dev/modules/get-started/) · [@bacons/apple-targets npm](https://www.npmjs.com/package/@bacons/apple-targets) · [#147](https://github.com/EvanBacon/expo-apple-targets/issues/147) · [#148](https://github.com/EvanBacon/expo-apple-targets/pull/148) · [#171](https://github.com/EvanBacon/expo-apple-targets/issues/171) · [#196](https://github.com/EvanBacon/expo-apple-targets/issues/196) · [#210](https://github.com/EvanBacon/expo-apple-targets/pull/210) · [expo-app-intents](https://docs.expo.dev/versions/latest/sdk/app-intents/) · [expo-background-task](https://docs.expo.dev/versions/latest/sdk/background-task/) · [expo-file-system](https://docs.expo.dev/versions/latest/sdk/filesystem/) · [expo-secure-store](https://docs.expo.dev/versions/latest/sdk/securestore/) · [SecureStore.ts](https://github.com/expo/expo/blob/main/packages/expo-secure-store/src/SecureStore.ts) · [expo-local-authentication](https://docs.expo.dev/versions/latest/sdk/local-authentication/) · [expo-web-browser](https://docs.expo.dev/versions/latest/sdk/webbrowser/) · [expo-localization](https://docs.expo.dev/versions/latest/sdk/localization/) · [EAS Update code signing](https://docs.expo.dev/eas-update/code-signing/) · [Expo Updates protocol](https://docs.expo.dev/technical-specs/expo-updates-1/) · [Expo pricing](https://expo.dev/pricing) · [op-sqlite](https://github.com/OP-Engineering/op-sqlite) · [PowerSync encryption](https://docs.powersync.com/client-sdks/advanced/data-encryption) · [PowerSync platforms](https://docs.powersync.com/resources/supported-platforms) · [React Native Harness](https://github.com/callstackincubator/react-native-harness) · npm: [react-native-device-crypto](https://www.npmjs.com/package/react-native-device-crypto), [react-native-secure-sign](https://www.npmjs.com/package/react-native-secure-sign) · [pnpm resolution settings](https://pnpm.io/settings/dependency-resolution) · [pnpm build settings](https://pnpm.io/settings/build) · [CocoaPods trunk read-only](https://blog.cocoapods.org/CocoaPods-Specs-Repo/) · [RFC 10017](https://www.rfc-editor.org/rfc/rfc10017.html)
- **Flutter:** [release archive](https://docs.flutter.dev/release/archive) · [decoupling blog](https://flutter.dev/blog/decoupling-material-cupertino) · [#170310](https://github.com/flutter/flutter/issues/170310) · [macOS](https://docs.flutter.dev/platform-integration/macos/building) · [web FAQ](https://docs.flutter.dev/platform-integration/web/faq) · [renderers](https://docs.flutter.dev/platform-integration/web/renderers) · [web accessibility](https://docs.flutter.dev/ui/accessibility/web-accessibility) · [supported platforms](https://docs.flutter.dev/reference/supported-platforms) · [alwaysUse24HourFormat](https://api.flutter.dev/flutter/widgets/MediaQueryData/alwaysUse24HourFormat.html) · [i18n](https://docs.flutter.dev/ui/internationalization) · [testing](https://docs.flutter.dev/testing/overview)
- **Kotlin, Rust and wrappers:** [KMP/CMP stability](https://kotlinlang.org/docs/multiplatform/supported-platforms.html) · [Kotlin components stability](https://kotlinlang.org/docs/components-stability.html) · [Objective-C interop](https://kotlinlang.org/docs/native-objc-interop.html) · [CMP 1.12.0](https://github.com/JetBrains/compose-multiplatform/releases/tag/v1.12.0) · [Swift export](https://kotlinlang.org/docs/native-swift-export.html) · [Compose tests](https://kotlinlang.org/docs/multiplatform/compose-test.html) · [UniFFI](https://github.com/mozilla/uniffi-rs) · [uniffi-bindgen-react-native](https://github.com/jhugman/uniffi-bindgen-react-native) · [Crux](https://github.com/redbadger/crux) · [rustc platform support](https://doc.rust-lang.org/nightly/rustc/platform-support.html) · [Tauri 2.0](https://v2.tauri.app/blog/tauri-20/) · [Tauri biometric docs](https://v2.tauri.app/plugin/biometric/) · [Tauri biometric changelog](https://github.com/tauri-apps/plugins-workspace/blob/v2/plugins/biometric/CHANGELOG.md) · [Tauri biometric README](https://github.com/tauri-apps/plugins-workspace/blob/v2/plugins/biometric/README.md) · [Tauri releases](https://github.com/tauri-apps/tauri/releases) · [Capacitor releases](https://github.com/ionic-team/capacitor/releases) · [Capacitor native project management](https://capacitorjs.com/docs/cordova) · [Electron](https://www.npmjs.com/package/electron) · [Electron systemPreferences](https://github.com/electron/electron/blob/main/docs/api/system-preferences.md) · [Valdi](https://github.com/Snapchat/Valdi)
- **Web platform:** [React Aria quality](https://react-aria.adobe.com/quality) · [DatePicker](https://react-aria.adobe.com/DatePicker) · [NumberField](https://react-aria.adobe.com/NumberField) · [Base UI releases](https://base-ui.com/react/overview/releases) · [SQLite Wasm persistence](https://sqlite.org/wasm/doc/trunk/persistence.md) · [WebKit storage policy](https://webkit.org/blog/14403/updates-to-storage-policy/) · [Safari 17 features](https://webkit.org/blog/14445/webkit-features-in-safari-17-0/) · WebKit ITP exemptions: [WebsiteDataStoreCocoa.mm](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/UIProcess/WebsiteData/Cocoa/WebsiteDataStoreCocoa.mm), [ResourceLoadStatisticsStore.cpp](https://github.com/WebKit/WebKit/blob/main/Source/WebKit/NetworkProcess/Classifier/ResourceLoadStatisticsStore.cpp) · [web.dev INP](https://web.dev/articles/inp) · [MDN browser-compat-data](https://github.com/mdn/browser-compat-data) · [Temporal](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal) · [light-dark()](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/light-dark) · [getWeekInfo](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Locale/getWeekInfo) · [Tailwind theme](https://tailwindcss.com/docs/theme) · [stylelint-declaration-strict-value](https://www.npmjs.com/package/stylelint-declaration-strict-value) · [Playwright snapshots](https://playwright.dev/docs/test-snapshots) · [Storybook visual tests](https://storybook.js.org/docs/writing-tests/visual-testing) · [Storybook AI](https://storybook.js.org/docs/ai) · [@powersync/web](https://www.npmjs.com/package/@powersync/web) · npm registry for versions ([react](https://www.npmjs.com/package/react), [react-aria-components](https://www.npmjs.com/package/react-aria-components), [react-native](https://www.npmjs.com/package/react-native), [react-native-web](https://www.npmjs.com/package/react-native-web))
- **Design tokens and tools:** [DTCG 2025.10 final report](https://www.w3.org/community/reports/design-tokens/CG-FINAL-format-20251028/) · [Resolver module](https://www.designtokens.org/tr/2025.10/resolver/) · [Terrazzo CLI changelog](https://github.com/terrazzoapp/terrazzo/blob/main/packages/cli/CHANGELOG.md) · [Terrazzo Swift](https://terrazzo.app/docs/integrations/swift) · [Style Dictionary DTCG](https://styledictionary.com/info/dtcg/) · [formats](https://styledictionary.com/reference/hooks/formats/predefined/) · [transforms](https://styledictionary.com/reference/hooks/transforms/predefined/) · [releases](https://github.com/style-dictionary/style-dictionary/releases) · [#1590](https://github.com/style-dictionary/style-dictionary/issues/1590) · [Figma variables](https://help.figma.com/hc/en-us/articles/15339657135383-Guide-to-variables-in-Figma) · [Figma modes for variables](https://help.figma.com/hc/en-us/articles/15343816063383-Modes-for-variables) · [Code Connect](https://developers.figma.com/docs/code-connect/) · [Figma pricing](https://www.figma.com/pricing/) · [Claude Design launch](https://www.anthropic.com/news/claude-design-anthropic-labs) · [Claude Design support](https://support.claude.com/en/articles/14604416-get-started-with-claude-design)
- **Research on agents:** [SwiftEval](https://arxiv.org/abs/2505.24324) · [SWE-Bench Mobile](https://arxiv.org/abs/2602.09540)
- **Secondary:** [Software Mansion: RN on Mac Catalyst](https://docs.swmansion.com/react-native-audio-api/docs/other/maccatalyst) · [9to5Mac: Copilot Money web (sponsored)](https://9to5mac.com/2026/01/01/copilot-money-brings-clarity-to-your-finances-now-on-the-web/) · MacGeneration on SwiftUI for Mac, via `apple-client-stack.md`
