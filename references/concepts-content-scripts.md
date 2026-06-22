# Content Scripts

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Content scripts run JavaScript and CSS in the context of a web page, where they can read and modify the DOM. They execute in an isolated world and talk to the rest of the extension through message passing.

## Declaring content scripts statically
Use the `content_scripts` manifest key. Each entry needs `matches` and at least one of `js`/`css`. Optional: `run_at` (default `"document_idle"`), `exclude_matches`, `include_globs`/`exclude_globs`, `world` (`"ISOLATED"` default or `"MAIN"`), `all_frames`, `match_about_blank`, `match_origin_as_fallback`.

```json
{
  "content_scripts": [{
    "matches": ["https://*.nytimes.com/*"],
    "exclude_matches": ["*://*/*business*"],
    "css": ["my-styles.css"],
    "js": ["content-script.js"],
    "run_at": "document_idle"
  }]
}
```

## Registering dynamically
When targets are known only at runtime, register with `chrome.scripting` (Chrome 96+; needs `"scripting"` plus host access): `registerContentScripts`, `getRegisteredContentScripts`, `updateContentScripts`, `unregisterContentScripts`. The programmatic options use `runAt` (camelCase), unlike the manifest's `run_at`.

```js
chrome.scripting.registerContentScripts([{
  id: "session-script", js: ["content.js"], matches: ["*://example.com/*"],
  runAt: "document_start", persistAcrossSessions: false,
}]);
```

## Injecting on demand with executeScript
`chrome.scripting.executeScript` injects a file or function in response to an event; needs `"scripting"` plus host access, which can come from `activeTab`.

```js
chrome.action.onClicked.addListener((tab) => {
  chrome.scripting.executeScript({ target: { tabId: tab.id }, func: (c) => { document.body.style.backgroundColor = c; }, args: ["orange"] });
});
```

## ISOLATED world vs MAIN world
By default content scripts run in an **isolated world**: their JavaScript variables are not visible to the host page or other extensions' content scripts, though they share the page's DOM. `"MAIN"` runs in the page's own world (subject to the page CSP and visible to page scripts). Because an isolated script can't read page variables directly, exchange data with `window.postMessage`.

```js
// content script (isolated world) listening for a page message
window.addEventListener("message", (event) => {
  if (event.source === window && event.data.type === "FROM_PAGE") console.log(event.data.text);
});
```

## run_at timing
- `"document_start"` — after CSS is injected, before the DOM is constructed.
- `"document_end"` — after the DOM is complete, before subresources load.
- `"document_idle"` — preferred default; between `document_end` and `window.onload`.

```json
{ "content_scripts": [{ "matches": ["https://*/*"], "js": ["early.js"], "run_at": "document_start" }] }
```
The `<all_urls>` token may replace the pattern above to match all schemes the extension can access. <!-- allow-placeholder -->

## matches and match patterns
A match pattern has the form `scheme://host/path` and supports `*` wildcards. `exclude_matches` removes pages `matches` would include. Examples: `https://*.nytimes.com/*`, `*://example.com/*`, and the special `<all_urls>` token. <!-- allow-placeholder -->

## Injecting CSS
Declare static CSS via `css` (honors the same `matches`/`run_at`), or inject programmatically with `chrome.scripting.insertCSS`. Reference packaged resources with `chrome.runtime.getURL` (and list them under `web_accessible_resources`).

## APIs available in content scripts
Directly usable: `chrome.dom`, `chrome.i18n`, `chrome.storage`, and parts of `chrome.runtime` (`connect`, `getManifest`, `getURL`, `id`, `onConnect`, `onMessage`, `sendMessage`). Everything else goes through the service worker.

## Security
The isolated-world CSP is roughly `script-src 'self' 'wasm-unsafe-eval' 'inline-speculation-rules' chrome-extension://[id]/; object-src 'self';`. Never `eval()` untrusted data; prefer `JSON.parse()`; pass functions (not strings) to `setTimeout`.
