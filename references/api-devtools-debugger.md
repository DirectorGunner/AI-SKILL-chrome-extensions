# DevTools Extension APIs & chrome.debugger

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
The `chrome.devtools.*` APIs (`inspectedWindow`, `network`, `panels`, `performance`, `recorder`) extend the Chrome DevTools UI and are available **only** on a DevTools page declared with the manifest key `devtools_page`; they are NOT available in the service worker. `chrome.debugger` is separate: it runs in the service worker / extension pages, needs the `"debugger"` permission, and speaks the Chrome DevTools Protocol (CDP).

## chrome.devtools.inspectedWindow
- **Purpose:** interact with the inspected window — get its tab id, evaluate code in its context, reload it, or list its resources.
- **Permission:** none (requires manifest key `devtools_page`).
- **Key properties:** `tabId` (the inspected tab's id; usable with `chrome.tabs.*`).
- **Key methods:** `eval(expression, options?, callback?)` runs JS in the inspected page's main frame and returns JSON-compliant results (`options`: `frameURL`, `scriptExecutionContext` (Chrome 107+), `useContentScriptContext`; `callback` receives `(result, exceptionInfo)`); `getResources(callback)` → `(resources)`; `reload(reloadOptions?)` (`ignoreCache`, `injectedScript`, `userAgent`).
- **Key events:** `onResourceAdded` → `(resource)`; `onResourceContentCommitted` → `(resource, content)`.
- **Key types:** `Resource` (`url`; `getContent(callback)` → `(content, encoding)`; `setContent(content, commit, callback?)`).
- **MV3 notes:** lives on the `devtools_page`, not the service worker. `eval` is limited to JSON-serializable return values.

```js
chrome.devtools.inspectedWindow.eval("jQuery.fn.jquery", (result, isException) => {
  if (isException) console.log("the page is not using jQuery");
  else console.log("The page is using jQuery v" + result);
});
```

## chrome.devtools.network
- **Purpose:** read the network requests shown in the DevTools Network panel, in HTTP Archive (HAR) format.
- **Permission:** none (requires `devtools_page`).
- **Key methods:** `getHAR(callback)` → `(harLog)`.
- **Key events:** `onRequestFinished` → `(request)`; `onNavigated` → `(url)`.
- **Key types:** `Request` adds `getContent(callback)` → `(content, encoding)` (`encoding` is empty when not encoded; otherwise names the encoding, currently `base64`).
- **MV3 notes:** `harLog` follows HAR v1.2. Available only on the `devtools_page`.

```js
chrome.devtools.network.onRequestFinished.addListener((request) => {
  console.log(request.request.url, request.response.status);
});
```

## chrome.devtools.panels
- **Purpose:** integrate into the DevTools window — create panels and sidebars, open resources.
- **Permission:** none (requires `devtools_page`).
- **Key properties:** `elements` (`ElementsPanel`), `sources` (`SourcesPanel`), `themeName` (`"default"`, `"dark"`).
- **Key methods:** `create(title, iconPath, pagePath, callback?)`; `openResource(url, lineNumber, columnNumber?, callback?)` (`columnNumber` Chrome 114+); `setOpenResourceHandler(callback?)` → `(resource, lineNumber)`; `setThemeChangeHandler(callback?)` (Chrome 99+).
- **Key types:** `ExtensionPanel` (events `onShown`, `onHidden`, `onSearch`; `createStatusBarButton(iconPath, tooltipText, disabled)`); `ExtensionSidebarPane` (`setPage(path)`, `setObject(jsonObject, rootTitle?, callback?)`, `setExpression(expression, rootTitle?, callback?)`, `setHeight(height)`); `Button` (event `onClicked`; `update(iconPath?, tooltipText?, disabled?)`); `ElementsPanel`/`SourcesPanel` (`onSelectionChanged`; `createSidebarPane(title, callback?)`).
- **MV3 notes:** read `themeName` or register `setThemeChangeHandler` to match the active light/dark theme.

```js
chrome.devtools.panels.create("My Panel", "icon.png", "Panel.html", (panel) => {
  // panel is an ExtensionPanel
});
chrome.devtools.panels.elements.createSidebarPane("Props", (sidebar) => {
  sidebar.setPage("Sidebar.html");
});
```

## chrome.devtools.performance
- **Purpose:** observe Performance-panel recording status.
- **Permission:** none (requires `devtools_page`). Chrome 129+.
- **Key events:** `onProfilingStarted` → `()`; `onProfilingStopped` → `()`.

```js
chrome.devtools.performance.onProfilingStarted.addListener(() => {});
chrome.devtools.performance.onProfilingStopped.addListener(() => {});
```

## chrome.devtools.recorder
- **Purpose:** customize the Recorder panel — extend export and replay.
- **Permission:** none (requires `devtools_page`).
- **Key methods:** `registerRecorderExtensionPlugin(plugin, name, mediaType)`; `createView(title, pagePath)` → `RecorderView` (Chrome 112+).
- **Key types:** `RecorderExtensionPlugin` (`stringify(recording)`, `stringifyStep(step)`, `replay(recording)` (Chrome 112+)); `RecorderView` (`show()`; events `onShown`, `onHidden`).
- **MV3 notes:** export customization Chrome 105+; replay/`createView` Chrome 112+.

```js
class MyPlugin {
  stringify(recording) { return Promise.resolve(JSON.stringify(recording)); }
  stringifyStep(step) { return Promise.resolve(JSON.stringify(step)); }
}
chrome.devtools.recorder.registerRecorderExtensionPlugin(new MyPlugin(), "MyPlugin", "application/json");
```

## chrome.debugger
- **Purpose:** an alternate transport for Chrome's remote debugging protocol — attach to tabs to instrument network, debug JS, mutate the DOM/CSS, and issue CDP commands.
- **Permission:** `"debugger"`.
- **Key methods** (Promises, Chrome 96+): `attach(target, requiredVersion)`; `detach(target)`; `sendCommand(target, method, commandParams?)` resolves to the CDP response body; `getTargets()` resolves to an array of `TargetInfo`.
- **Key events:** `onEvent` → `(source, method, params?)`; `onDetach` → `(source, reason)`.
- **Key types:** `Debuggee` (`tabId?`, `extensionId?`, `targetId?`); `DebuggerSession` (Chrome 125+, adds `sessionId?`); `TargetInfo` (`id`, `type`, `title`, `url`, `attached`, `tabId?`, `extensionId?`, `faviconUrl?`); `TargetInfoType` (`"page"`, `"background_page"`, `"worker"`, `"other"`); `DetachReason` (`"target_closed"`, `"canceled_by_user"`).
- **MV3 notes:** runs from the service worker / extension pages (not a DevTools page). Flat sessions (Chrome 125+) route to child targets via `sessionId`. While attached, Chrome shows an active-debugging banner to the user.

```js
const target = { tabId: someTabId };
await chrome.debugger.attach(target, "1.3");
await chrome.debugger.sendCommand(target, "Network.enable");
chrome.debugger.onEvent.addListener((source, method, params) => {
  if (method === "Network.responseReceived") console.log(params.response.url);
});
```
