# Extension Lifecycle & Updates

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
How an MV3 extension is installed, started, updated, and uninstalled, and how the service worker reacts to each phase.

## Install and startup events
On install/update, the worker's web `install` event fires, then `chrome.runtime.onInstalled`, then the web `activate` event. `chrome.runtime.onInstalled` fires on first install, on update to a new version, and when Chrome updates. Its `details` object has:
- `details.reason` — an `OnInstalledReason`: `"install"`, `"update"`, `"chrome_update"`, or `"shared_module_update"`.
- `details.previousVersion` — present only when `reason` is `"update"`.
- `details.id` — the imported shared-module extension id; present only for `"shared_module_update"`.

```js
chrome.runtime.onInstalled.addListener((details) => {
  if (details.reason !== "install" && details.reason !== "update") return;
  chrome.contextMenus.create({ id: "sampleContextMenu", title: "Sample", contexts: ["selection"] });
});
```

`chrome.runtime.onStartup` fires when a profile that has the extension first starts up (not for incognito, even in `"split"` mode); no service worker events are invoked on profile start.

## How updates roll out
Chrome checks for updates on startup and every few hours. An update installs only while the extension is **idle** — the service worker is not running and no extension pages (side panel, popup, options) are open. Active content scripts do not block idleness; disabled extensions still receive updates. A worker kept constantly busy may never idle, deferring the update until restart. Developers can force an update at `chrome://extensions` (Developer mode → Update).

## Detecting and requesting updates
`chrome.runtime.onUpdateAvailable` fires when an update is available but deferred because the app is running; `details.version` is the available version. `chrome.runtime.requestUpdateCheck()` requests an immediate check and resolves to a status — a `RequestUpdateCheckStatus`: `"throttled"`, `"no_update"`, or `"update_available"` (frequent calls are throttled; the extension still must idle before installing). `chrome.runtime.reload()` reloads the extension (not supported in kiosk mode). The `minimum_chrome_version` manifest key blocks installation on incompatible Chrome versions.

## Service worker lifecycle during updates
The worker loads on demand and unloads when dormant; global variables do not survive shutdown — persist to `chrome.storage`, `IndexedDB`, or `CacheStorage`. Termination triggers: 30 seconds of inactivity, a single request over 5 minutes, or a `fetch()` response over 30 seconds.

## Uninstall hook
`chrome.runtime.setUninstallURL(url)` sets a URL opened after uninstall (cleanup, analytics, survey). The `url` must use `http:`/`https:` (max 1023 characters); an empty string opens no tab. Returns a Promise.
