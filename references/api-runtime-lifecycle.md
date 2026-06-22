# Runtime, Events & Lifecycle Timers

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
This cluster covers the core extension lifecycle surface: `chrome.runtime` (messaging, install/startup events, manifest and URL helpers), the shared `chrome.events` listener/declarative-rule machinery every event API builds on, and three scheduling/sensing APIs that pair naturally with a service worker — `chrome.alarms`, `chrome.idle`, and `chrome.power`. In MV3 these are called from the service worker or extension pages; most methods return a Promise.

## chrome.runtime
- **Purpose:** inspect the running extension (manifest, ID, URL resolution, platform info), respond to lifecycle moments, and pass messages between extension contexts or to native hosts.
- **Permission:** none for most members. `connectNative()`, `sendNativeMessage()`, and `onConnectNative` require `"nativeMessaging"`.
- **Key methods** (MV3 returns Promises; the connect helpers return synchronously):
  - `connect(extensionId?, connectInfo?)` → returns a `Port`
  - `connectNative(application)` → returns a `Port`
  - `getContexts(filter)` → resolves to `ExtensionContext[]`
  - `getManifest()` → returns the manifest `object` (synchronous)
  - `getPlatformInfo()` → resolves to `PlatformInfo`
  - `getURL(path)` → returns a `string` (synchronous)
  - `openOptionsPage()` → resolves when the options page opens
  - `reload()` → reloads the extension (synchronous)
  - `requestUpdateCheck()` → resolves to an update-check result `object`
  - `sendMessage(extensionId?, message, options?)` → resolves to the response
  - `sendNativeMessage(application, message)` → resolves to the host reply
  - `setUninstallURL(url)` → resolves when set
- **Key events:** `onConnect`, `onConnectExternal`, `onConnectNative`, `onInstalled` (details `{reason, previousVersion?, id?}`), `onMessage` (`(message, sender, sendResponse)` — return `true` to keep `sendResponse` valid for an async reply), `onMessageExternal`, `onStartup`, `onUpdateAvailable` (details `{version}`), `onUserScriptConnect`, `onUserScriptMessage`.
- **Key types:** `Port` (`name`, `onMessage`, `onDisconnect`, `sender?`, `postMessage(message)`, `disconnect()`); `MessageSender` (`id?`, `nativeApplication?`, `origin?`, `tab?`, `url?`, `frameId?`, `documentId?`, `documentLifecycle?`, `tlsChannelId?`); properties `runtime.id` and `runtime.lastError` (only meaningful inside a callback). Enums: `OnInstalledReason` = `"install"`, `"update"`, `"chrome_update"`, `"shared_module_update"`; `ContextType` (Chrome 114+) = `"TAB"`, `"POPUP"`, `"BACKGROUND"`, `"OFFSCREEN_DOCUMENT"`, `"SIDE_PANEL"`, `"DEVELOPER_TOOLS"`; `PlatformOs` = `"mac"`, `"win"`, `"android"`, `"cros"`, `"linux"`, `"openbsd"`; `PlatformArch` = `"arm"`, `"arm64"`, `"x86-32"`, `"x86-64"`, `"mips"`, `"mips64"`, `"riscv64"`; `RequestUpdateCheckStatus` = `"throttled"`, `"no_update"`, `"update_available"`.
- **MV3 notes:** there is no background page, so `getBackgroundPage()` is gone — use the service worker plus message passing. Reloading an unpacked extension (or calling `reload()`) re-fires `onInstalled` with reason `"update"`, so guard install-only work. Returning `true` from an `onMessage` listener keeps the channel open for a deferred `sendResponse`.

```js
// service-worker.js
chrome.runtime.onInstalled.addListener((details) => {
  if (details.reason === "install") {
    chrome.runtime.setUninstallURL("https://example.com/uninstall-survey");
  }
});

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === "ping") {
    queueMicrotask(() => sendResponse({ pong: true }));
    return true; // keep the channel open for the async sendResponse
  }
});
```

## chrome.events
- **Purpose:** the shared shape behind every dispatched event — both ordinary listeners and declarative rule sets.
- **Permission:** none.
- **Key methods:** on each `Event` object — `addListener(callback)`, `removeListener(callback)`, `hasListener(callback)`, `hasListeners()`. For declarative APIs: `addRules(rules, callback?)`, `removeRules(ruleIdentifiers?, callback?)`, `getRules(ruleIdentifiers?, callback)`.
- **Key types:** `Rule` = `{id?, priority? (default 100), conditions[], actions[], tags?}`. `UrlFilter` matches with `hostEquals`/`hostPrefix`/`hostSuffix`/`hostContains`, `pathEquals`/`pathPrefix`/`pathSuffix`/`pathContains`, `queryEquals`/`queryPrefix`/`querySuffix`/`queryContains`, `urlEquals`/`urlPrefix`/`urlSuffix`/`urlContains`/`urlMatches`, `originAndPathMatches`, `schemes`, `ports` (number or array, e.g. `[80, 443, [1000, 1200]]`), and `cidrBlocks` (Chrome 123+).
- **MV3 notes:** declarative rules persist across sessions, so register them inside `runtime.onInstalled` and clear stale rules first. Passing a filter as the second argument to `addListener` lets the browser skip waking the service worker for irrelevant events.

```js
// Filtered listener: only wake for one host.
chrome.webNavigation.onCommitted.addListener(
  (details) => console.log("committed:", details.url),
  { url: [{ hostSuffix: "example.com" }] }
);
```

## chrome.alarms
- **Purpose:** run code at a future moment or on a repeating schedule — the MV3-friendly replacement for `setTimeout`/`setInterval` in a service worker.
- **Permission:** `"alarms"`.
- **Key methods** (Promises): `create(name?, alarmInfo)`; `get(name?)` → resolves to an `Alarm` or `undefined`; `getAll()` → resolves to `Alarm[]`; `clear(name?)` → resolves to a `boolean`; `clearAll()` → resolves to a `boolean`.
- **Key events:** `onAlarm` → `(alarm)`.
- **Key types:** `Alarm` = `{name, scheduledTime (ms past epoch), periodInMinutes?, persistAcrossSessions}`; `AlarmCreateInfo` = `{when? (ms past epoch), delayInMinutes?, periodInMinutes?, persistAcrossSessions?}`.
- **MV3 notes:** in a packed/production extension alarms fire at most once every 30 seconds; values for `delayInMinutes`/`periodInMinutes` under `0.5` minutes log a warning and are clamped. Unpacked extensions bypass the floor. Alarms do not wake a sleeping device — a missed alarm fires once on wake.

```js
chrome.runtime.onInstalled.addListener(async () => {
  await chrome.alarms.create("refresh", { delayInMinutes: 1, periodInMinutes: 5 });
});

chrome.alarms.onAlarm.addListener((alarm) => {
  if (alarm.name === "refresh") console.log("fired at", alarm.scheduledTime);
});
```

## chrome.idle
- **Purpose:** detect when the machine transitions between active use, idleness, and a locked screen.
- **Permission:** `"idle"`.
- **Key methods:** `queryState(detectionIntervalInSeconds)` → resolves to an `IdleState`; `setDetectionInterval(intervalInSeconds)` (default `60` seconds); `getAutoLockDelay()` → resolves to a number (ChromeOS only, Chrome 73+).
- **Key events:** `onStateChanged` → `(newState)`.
- **Key types:** `IdleState` = `"active"`, `"idle"`, `"locked"`.
- **MV3 notes:** `"locked"` is only reported where the OS exposes lock/screensaver state. Registering `onStateChanged` from the service worker lets idle transitions wake it.

```js
chrome.idle.setDetectionInterval(30);
chrome.idle.onStateChanged.addListener((newState) => console.log("idle ->", newState));
const state = await chrome.idle.queryState(15);
```

## chrome.power
- **Purpose:** temporarily override the OS power-management policy to keep the system (or display) awake.
- **Permission:** `"power"`.
- **Key methods:** `requestKeepAwake(level)` (a new call replaces this extension's prior request); `releaseKeepAwake()`; `reportActivity()` (Chrome 113+, ChromeOS only) → resolves when reported.
- **Key types:** `Level` = `"system"` (block sleep on inactivity; display may still turn off) or `"display"` (keep the display on and the system awake). `"display"` takes precedence over `"system"`.
- **MV3 notes:** `requestKeepAwake`/`releaseKeepAwake` are fire-and-forget. Always pair a request with a release; another extension's higher-precedence level can override yours.

```js
chrome.power.requestKeepAwake("display"); // keep screen on during a long task
chrome.power.releaseKeepAwake();
```
