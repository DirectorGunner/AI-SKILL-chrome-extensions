# Chrome Extensions — Gotchas

Recurring Manifest V3 failure modes and what to do instead:

- The background is an **event-driven service worker that terminates when idle** — there are no
  persistent background pages. Do not keep state in globals; persist to `chrome.storage` (and
  register event listeners at the **top level** so a dormant worker can be woken).
- `chrome.scripting.executeScript` **replaces** `chrome.tabs.executeScript`; it takes a `target`
  plus `func` (with `args`) or `files`. A `func` is serialized, so pass data via `args`.
- Blocking `chrome.webRequest` is restricted in MV3 (policy-installed extensions only) — use
  `declarativeNetRequest` for blocking/redirect/modify; observational `webRequest` still works.
- **Remotely hosted code is disallowed** — all executable code must ship in the package.
- `setTimeout`/`setInterval` and long timers are unreliable in a service worker — use `chrome.alarms`.
- Many `chrome.*` methods **return Promises** in MV3 while old samples use callbacks.
- `permissions` and `host_permissions` are **separate** manifest keys.
- Content scripts run in an **isolated world** and cannot see page JavaScript variables; bridge with
  `window.postMessage`.
- Service workers have **no DOM** — use an **offscreen document** for DOM-dependent work.
- `manifest_version` must be `3`. Never present MV2 patterns (background pages, `browser_action`,
  blocking `webRequest`) as current.