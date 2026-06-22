# Tabs, Tab Groups & Windows

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
These APIs manage the browser's tab, tab-group, and window surfaces from a service worker or extension page (not content scripts). All methods are Promise-based in MV3; most tab/window operations need no permission, but reading sensitive metadata does.

## chrome.tabs
- **Purpose:** create, query, move, update, group, and message tabs; capture screenshots, detect language, and control zoom.
- **Permission:** mostly none (creating, reloading, navigating tabs needs no permission). The `"tabs"` permission unlocks the sensitive properties `url`, `pendingUrl`, `title`, `favIconUrl`. `"activeTab"` grants a temporary host grant on user invocation. Host permissions (e.g. `"https://*/*"`) are needed for `captureVisibleTab()` and for querying by `url`.
- **Key methods** (all return Promises):
  - `query(queryInfo)` — match by `active`, `currentWindow`, `lastFocusedWindow`, `url`, `pinned`, `groupId`, `windowId`, `status`, etc.
  - `create(createProperties)` — `url`, `active`, `index`, `windowId`, `pinned`, `openerTabId`.
  - `update(tabId?, updateProperties)` — `url`, `active`, `highlighted`, `pinned`, `muted`, `openerTabId`, `autoDiscardable`.
  - `sendMessage(tabId, message, options?)` — message that tab's content scripts.
  - `get(tabId)`, `getCurrent()` (resolves to `undefined` outside a tab), `remove(tabIds)`, `move(tabIds, moveProperties)`, `duplicate(tabId)`, `reload(tabId?, reloadProperties?)`.
  - `group(options)`, `ungroup(tabIds)` — grouping lives here, not in `tabGroups`.
  - `highlight(highlightInfo)`, `discard(tabId?)`, `goBack(tabId?)`, `goForward(tabId?)`, `detectLanguage(tabId?)`, `captureVisibleTab(windowId?, options?)`, `connect(tabId, connectInfo?)`, `getZoom`/`setZoom`/`getZoomSettings`/`setZoomSettings`.
- **Key events:** `onCreated`, `onUpdated`, `onActivated`, `onRemoved`, `onMoved`, `onAttached`, `onDetached`, `onHighlighted`, `onReplaced`, `onZoomChange`.
- **Key types:** `Tab` (`id`, `windowId`, `index`, `url`, `title`, `favIconUrl`, `status`, `active`, `highlighted`, `pinned`, `incognito`, `mutedInfo`, `audible`, `discarded`, `autoDiscardable`, `groupId`, `openerTabId`, `pendingUrl`, `sessionId`, `width`, `height`, `frozen`, `lastAccessed`). `MutedInfo` (`muted`, `reason` = `"user"`/`"capture"`/`"extension"`, `extensionId`). Enums: `TabStatus` = `"unloaded"`/`"loading"`/`"complete"`; `ZoomSettingsMode` = `"automatic"`/`"manual"`/`"disabled"`; `ZoomSettingsScope` = `"per-origin"`/`"per-tab"`.
- **MV3 notes:** callable from the service worker and extension pages, never content scripts. Promise-only (no callbacks). Operations may fail while the user drags a tab; retry with backoff.

```js
// Query the active tab, then message its content script.
const [tab] = await chrome.tabs.query({ active: true, lastFocusedWindow: true });
const reply = await chrome.tabs.sendMessage(tab.id, { action: "GET_INFO" });
console.log(reply);
```

## chrome.tabGroups
- **Purpose:** inspect, recolor, rename, collapse, and reposition tab groups. Adding/removing tabs is done via `chrome.tabs.group`/`ungroup`. Chrome 89+.
- **Permission:** `"tabGroups"`.
- **Key methods** (Promises): `get(groupId)`, `query(queryInfo)`, `move(groupId, moveProperties)`, `update(groupId, updateProperties)`. `MoveProperties`: `index` (use `-1` for end), `windowId`. `UpdateProperties`: `collapsed`, `color`, `title`.
- **Key events:** `onCreated`, `onMoved`, `onRemoved`, `onUpdated`.
- **Key types:** `TabGroup` (`id`, `windowId`, `collapsed`, `color`, `title`, `shared`). `Color` enum: `"grey"`, `"blue"`, `"red"`, `"yellow"`, `"green"`, `"pink"`, `"purple"`, `"cyan"`, `"orange"`. Constant `TAB_GROUP_ID_NONE` = `-1`.
- **MV3 notes:** requires `"tabs"` alongside `"tabGroups"` to also act on member tabs.

```js
const groupId = await chrome.tabs.group({ tabIds: [tabA, tabB] });
await chrome.tabGroups.update(groupId, { title: "Research", color: "blue" });
```

## chrome.windows
- **Purpose:** create, query, focus, resize, and close browser windows.
- **Permission:** none for basic operations; `"tabs"` is required to read `url`, `pendingUrl`, `title`, or `favIconUrl` on contained `Tab` objects.
- **Key methods** (Promises, Chrome 88+): `get(windowId, queryOptions?)`, `getCurrent(queryOptions?)`, `getLastFocused(queryOptions?)`, `getAll(queryOptions?)`, `create(createData?)`, `update(windowId, updateInfo)`, `remove(windowId)`. `QueryOptions`: `populate`, `windowTypes` (default `['normal', 'popup']`). `CreateData`: `url`, `tabId`, `left`, `top`, `width`, `height`, `focused`, `incognito`, `type`, `state`, `setSelfAsOpener` (Chrome 64+). `UpdateInfo`: `left`, `top`, `width`, `height`, `focused`, `drawAttention`, `state`.
- **Key events:** `onCreated`, `onRemoved`, `onFocusChanged`, `onBoundsChanged` (Chrome 86+).
- **Key types:** `Window` (`id`, `focused`, `alwaysOnTop`, `incognito`, `type`, `state`, `width`, `height`, `left`, `top`, `tabs`, `sessionId`). Enums: `WindowType` = `"normal"`/`"popup"`/`"panel"`/`"app"`/`"devtools"`; `WindowState` = `"normal"`/`"minimized"`/`"maximized"`/`"fullscreen"`; `CreateType` = `"normal"`/`"popup"`/`"panel"`. Constants `WINDOW_ID_CURRENT` = `-2`, `WINDOW_ID_NONE` = `-1`.
- **MV3 notes:** the "current window" is the one running the code, not necessarily the focused one; from a service worker it may be absent. `"panel"`/`"app"` window types are deprecated.

```js
const win = await chrome.windows.create({
  url: "https://example.com/", type: "popup", width: 800, height: 600, focused: true,
});
const all = await chrome.windows.getAll({ populate: true, windowTypes: ["normal", "popup"] });
for (const w of all) console.log(w.id, w.tabs.length);
```
