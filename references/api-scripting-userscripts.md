# Scripting & User Scripts

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
`chrome.scripting` injects JavaScript and CSS into pages at runtime and manages dynamically registered content scripts. `chrome.userScripts` runs user-supplied (arbitrary) code in a dedicated, CSP-exempt execution world with its own messaging. Both replace MV2 patterns such as `chrome.tabs.executeScript`.

## chrome.scripting
- **Purpose:** inject scripts/styles on demand and register/update/unregister content scripts at runtime, instead of declaring them statically in the manifest.
- **Permission:** `"scripting"` plus host access via `"host_permissions"` or the `"activeTab"` permission. Example: `"permissions": ["scripting", "activeTab"]`.
- **Key methods** (all Promise-based; no callback variants):
  - `executeScript(injection)` → resolves to `InjectionResult[]`
  - `insertCSS(injection)` → resolves when applied
  - `removeCSS(injection)` (Chrome 90+)
  - `registerContentScripts(scripts)` (Chrome 96+)
  - `getRegisteredContentScripts(filter?)` → resolves to `RegisteredContentScript[]` (Chrome 96+)
  - `unregisterContentScripts(filter?)` (Chrome 96+)
  - `updateContentScripts(scripts)` (Chrome 96+)
- **Key types:**
  - `ScriptInjection`: `target` (required), `func` (Chrome 92+), `files`, `args` (Chrome 92+), `world` (Chrome 95+), `injectImmediately` (Chrome 102+). Exactly one of `func` or `files`. `args` is valid only with `func` and must be JSON-serializable.
  - `InjectionTarget`: `tabId` (required), `frameIds`, `allFrames`, `documentIds` (Chrome 106+). `frameIds` and `allFrames` are mutually exclusive.
  - `CSSInjection`: `target` (required), `css`, `files`, `origin`. Exactly one of `css` or `files`.
  - `StyleOrigin`: `"AUTHOR"` (default), `"USER"`.
  - `RegisteredContentScript`: `id` (required, must not start with `_`), `matches`, `excludeMatches`, `css`, `js`, `allFrames`, `matchOriginAsFallback` (Chrome 119+), `runAt`, `persistAcrossSessions` (default `true`), `world` (Chrome 102+).
  - `ContentScriptFilter`: `ids`.
  - `ExecutionWorld`: `"ISOLATED"` (default), `"MAIN"`.
  - `InjectionResult`: `result`, `frameId` (Chrome 90+), `documentId` (Chrome 106+).
- **MV3 notes:** MV3-only (Chrome 88+). A `func` is serialized then deserialized in the target, losing its closure — pass data through `args`. Injected files/content must satisfy the extension CSP. Each target needs host permission; `"activeTab"` grants it temporarily. `removeCSS` requires `css`/`files`/`origin` to match the original `insertCSS` call.

```js
// Inject a function with arguments into the active tab's top frame.
async function highlight(tabId, color) {
  const [res] = await chrome.scripting.executeScript({
    target: { tabId },
    func: (c) => { document.body.style.outline = `4px solid ${c}`; return document.title; },
    args: [color],
    world: "ISOLATED",
  });
  console.log("title:", res.result, "frame:", res.frameId);
}
```

## chrome.userScripts
- **Purpose:** register and run arbitrary user-provided scripts in the User Scripts world, configure that world (CSP, messaging), and (Chrome 135+) execute scripts imperatively.
- **Permission:** `"userScripts"` plus host permissions (e.g. `"*://example.com/*"`). MV3; minimum Chrome 120. Before Chrome 138 the user must enable Developer Mode; Chrome 138+ exposes an "Allow User Scripts" toggle on the extension's details page.
- **Key methods** (all Promise-based): `register(scripts)`; `getScripts(filter?)`; `update(scripts)`; `unregister(filter?)`; `configureWorld(properties)`; `getWorldConfigurations()` (Chrome 133+); `resetWorldConfiguration(worldId?)` (Chrome 133+); `execute(injection)` (Chrome 135+).
- **Key events:** messaging is received on the extension side via `runtime.onUserScriptMessage` and `runtime.onUserScriptConnect` (not the standard `onMessage`/`onConnect`).
- **Key types:**
  - `RegisteredUserScript`: `id` (must not start with `_`), `matches` (required), `excludeMatches`, `includeGlobs`, `excludeGlobs`, `js` (required), `allFrames` (default `false`), `runAt` (default `document_idle`), `world` (default `"USER_SCRIPT"`), `worldId` (Chrome 133+).
  - `ScriptSource`: `code` or `file`.
  - `WorldProperties`: `worldId` (Chrome 133+), `csp` (default is the `ISOLATED` world CSP), `messaging` (default `false`).
  - `ExecutionWorld`: `"MAIN"`, `"USER_SCRIPT"`.
  - `RunAt`: `document_start`, `document_end`, `document_idle`.
- **MV3 notes:** detect availability by calling `getScripts` inside a `try/catch`; it throws when the permission/toggle is off. The `USER_SCRIPT` world is exempt from the page CSP. To message between user scripts and the extension you must first `configureWorld({ messaging: true })`. User scripts are cleared on extension update — re-register from a `runtime.onInstalled` handler (reason `"update"`).

```js
await chrome.userScripts.configureWorld({ messaging: true });
await chrome.userScripts.register([{
  id: "greeter",
  matches: ["*://example.com/*"],
  js: [{ code: "chrome.runtime.sendMessage({ hello: location.host });" }],
  runAt: "document_idle",
  world: "USER_SCRIPT",
}]);
chrome.runtime.onUserScriptMessage.addListener((msg, sender) => {
  console.log("from user script:", msg, sender.url);
});
```
