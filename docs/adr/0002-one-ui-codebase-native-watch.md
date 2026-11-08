---
status: accepted
---

# One UI codebase for iOS, macOS and web; native watch app

The iPhone, Mac and web apps share one UI codebase, so each design-system component is written once and reused everywhere. The Apple Watch app is native SwiftUI, because cross-platform UI frameworks don't target watchOS. The maintainer chose shared code over native SwiftUI on iPhone and Mac, accepting a less native feel on the Mac in exchange for one implementation per component. ADR 0004 picks the framework.
