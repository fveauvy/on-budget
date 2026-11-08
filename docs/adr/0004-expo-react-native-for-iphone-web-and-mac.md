---
status: accepted
---

# Expo (React Native) for the iPhone, web and Mac apps

ADR 0002 gives the iPhone, Mac and web apps one UI codebase and leaves its framework open. That codebase is one Expo (React Native) app: UIKit views, native tabs and Expo UI's SwiftUI controls on the iPhone, and react-native-web in browsers, exported as a single-page app that each instance serves as an installable web app. It is the only option that meets all three constraints: one codebase, a native feel on the iPhone, where the daily sessions happen, and a watch app with iOS extensions that builds and reaches TestFlight (ADR 0005). The Mac starts as that web app added to the Dock from Safari. It switches to the iPad app on the Mac only if spike SP-08 (UI-S3) shows Safari deleting a Dock app's offline data, and moves to react-native-macos once Expo supports macOS again. The maintainer confirmed this choice.

## Considered Options

- One web app in native shells (Capacitor) scored higher overall and shares more code, but on the iPhone it is a WebView without system controls. If SP-18 (UI-S2) fails hard, the framework choice goes back to the maintainer, with Capacitor as the option to put to them.
- SwiftUI apps beside a separate React web app feel the most native, but they are two UI codebases, which ADR 0002 rules out.
- Flutter and Compose Multiplatform draw their own pixels, on the iPhone and on the web.

## Consequences

- Apple's changes reach the apps only through Expo, which ships a new SDK every few months.
- Expo UI's web versions are experimental, so each system control gets its own web implementation in the same component (`.ios.tsx` and `.web.tsx`).
- No over-the-air updates: `expo-updates` stays off and every JavaScript fix ships as a signed build, since an update channel would let its key holder change the code of every installed app that holds bank data.
- Hermes ships only a partial `Intl`, so the apps load FormatJS polyfills at start-up (ADR 0016).
