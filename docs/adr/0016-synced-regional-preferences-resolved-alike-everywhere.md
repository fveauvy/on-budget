---
status: accepted
---

# Synced regional preferences, resolved alike everywhere

A person's regional preferences (region, hour cycle and first day of the week, each automatic or explicit) are synced data, and an explicit value wins on every device. One resolver, in TypeScript, turns them into a resolved locale tag (such as `fr-FR-u-hc-h23-fw-mon`) and the settings every formatter and date picker uses; the glance and push payloads carry that tag to the Swift targets, whose Foundation formatters take it, and nothing formats from the raw system or browser locale. The UI language is the exception: on Apple devices it follows the system's per-app language setting, which Apple says not to override in code, and only the web app has its own language choice. Why: browsers can't see the system's regional overrides (with English as the language and France as the region, Safari still prints US dates, times and amounts), and a person should see the same formats on the iPhone, the watch and the web. The maintainer accepted these default language and region rules, which `PRODUCT.md` had left undecided.

## Consequences

- On the web, automatic means the values the person's Apple apps last reported, or a choice made once at first run.
- Formats come from named fields, never preset styles, and the web uses React Aria date and time fields, because `<input type=date|time>` follows the browser.
- A Swift resolver comes only when a Swift target must resolve preferences on its own; until then, golden vectors check that Foundation renders the resolved tag as the shared formatter does (SP-17).
- Hermes needs every FormatJS polyfill and its time-zone data at start-up, and in rare measured cases the iPhone app still formats differently from its own watch app.
- Currency belongs to the household, through its base currency, not to a person's preferences.
