---
status: accepted
---

# Swift where iOS needs it, in a committed `ios/` project

The Expo app (ADR 0004) stays TypeScript except where iOS needs native code. Swift covers the watch app and its complications, since watchOS runs no JavaScript, and on the iPhone: a `Security` module (the sign-in sheet, Keychain items, native token refresh, the app lock and a biometry-bound Secure Enclave key for step-up), a `WatchRelay` module (ADR 0006), a notification service extension that decrypts alerts and renders their text, and, after the MVP, an App Intents extension for Apple Pay tap and cash capture and iOS widgets. iOS runs these pieces when JavaScript may not be running, and the libraries that would avoid Swift fall short: none signs with the Secure Enclave, `expo-web-browser` uses the auth-session initializer that the iOS 27.3 SDK deprecates, `react-native-watch-connectivity` only bridges to JavaScript, and `expo-app-intents` is alpha. The `ios/` project comes from one `expo prebuild`, then is committed and owned in Xcode, because Continuous Native Generation can't build the watch today: its target tooling trails the current SDK and can't link Swift packages into a watch target.

## Consequences

- Each Expo SDK upgrade's native changes are applied by hand, and the app config's native fields stop syncing. Continuous Native Generation returns once it builds the watch on the SDK in use (spike SP-03 (UI-S4)).
- No EAS: GitHub-hosted macOS runners build pull requests, and Xcode Cloud archives releases from the committed project (ADR 0019).
- The refresh token never enters JavaScript, which sees only access tokens, and stays in the app's own Keychain access group, since no extension calls the API.
- Each local database keeps one writer: the App Intents extension, when it comes, appends captures to a handoff file in the app group, which the JavaScript sync client drains.
