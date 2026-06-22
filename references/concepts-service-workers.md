# The MV3 Background Service Worker

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
In Manifest V3 the background page is replaced by an event-driven service worker registered through `background.service_worker`. It wakes to handle events, terminates when idle, has no DOM, and keeps no reliable in-memory state.

## Registration
Declared in `manifest.json`; extensions cannot register at runtime. Add `"type": "module"` to use ES module `import`/`export` (dynamic `import()` and import assertions are not supported); without it, use `importScripts()`.

```json
{
  "background": { "service_worker": "service-worker.js", "type": "module" }
}
```

The worker can only be updated by publishing a new version; remotely hosted code is prohibited in MV3, so the worker ships inside the package.

## Event-driven lifecycle
On install or update, events fire in order: the web `install` event, then `chrome.runtime.onInstalled` (fires on first install, on update to a new version, and when Chrome updates), then the web `activate` event. `chrome.runtime.onStartup` fires when the profile starts (no service worker events fire in that case).

```js
chrome.runtime.onInstalled.addListener((details) => {
  if (details.reason !== "install" && details.reason !== "update") return;
  chrome.contextMenus.create({ id: "sampleContextMenu", title: "Sample", contexts: ["selection"] });
});
```

## Termination and timers
Chrome terminates the worker when 30 seconds of inactivity elapse, a single request exceeds 5 minutes, or a `fetch()` response takes over 30 seconds. Events and API calls reset these timers (API-call resets since Chrome 110; WebSocket messages Chrome 116+; long-lived ports Chrome 114+; offscreen messages Chrome 109+; native messaging Chrome 105+).

## Global state is lost — use chrome.storage
Global variables do not survive shutdown. Persist with `chrome.storage` (recommended), `IndexedDB`, or `CacheStorage`. The Web Storage API is **not** available in extension service workers.

```js
async function increment() {
  const { count = 0 } = await chrome.storage.local.get("count");
  await chrome.storage.local.set({ count: count + 1 });
}
```

## Keep listeners at the top level
Declare event handlers in the global scope so they register synchronously when the worker starts; nesting a listener inside an async callback risks the worker shutting down before it attaches.

```js
// Better: register at top level, do async work separately
chrome.action.onClicked.addListener(handleActionClick);
chrome.storage.local.get(["badgeText"], ({ badgeText }) => {
  chrome.action.setBadgeText({ text: badgeText });
});
```

Where an API supports event filters, use them so the worker is not woken for irrelevant events.

## Use chrome.alarms instead of setTimeout/setInterval
`setTimeout()`/`setInterval()` do not survive termination and may never fire. Schedule with `chrome.alarms` (minimum period 30 seconds as of Chrome 120).

```js
chrome.alarms.create("refresh", { periodInMinutes: 1 });
chrome.alarms.onAlarm.addListener((alarm) => { if (alarm.name === "refresh") {/* work */} });
```

## No DOM — use offscreen documents
Service workers cannot access the DOM. Create a hidden offscreen document with `chrome.offscreen` (permission `"offscreen"`). `createDocument()` takes `url` (a bundled static HTML file), `reasons` (a `Reason` array), and `justification`. Only one offscreen document may exist at a time, and only `chrome.runtime` is available inside it.

```js
chrome.offscreen.createDocument({
  url: "off_screen.html", reasons: ["CLIPBOARD"], justification: "reason for needing the document",
});
```

Related: `chrome.offscreen.closeDocument()`, `chrome.offscreen.hasDocument()`.
