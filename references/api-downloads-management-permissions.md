# Downloads, Management & Permissions

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
`chrome.downloads` (start, monitor, and manipulate downloads), `chrome.management` (inspect/control installed extensions and apps), and `chrome.permissions` (request and relinquish optional permissions at runtime — central to the MV3 least-privilege model).

## chrome.downloads
- **Purpose:** initiate downloads, search download history, and manage the file lifecycle (pause, resume, cancel, erase, open, show).
- **Permission:** `"downloads"`. Capability permissions: `"downloads.open"` (for `open()`), `"downloads.ui"` (for `setUiOptions()`), `"downloads.shelf"` (for `setShelfEnabled()`).
- **Key methods** (Promises unless noted): `download(options)` resolves to a `number` `downloadId`; `search(query)` resolves to `DownloadItem[]`; `pause(downloadId)`; `resume(downloadId)`; `cancel(downloadId)`; `getFileIcon(downloadId, options?)` resolves to a URL string; `open(downloadId)` (requires `"downloads.open"`); `show(downloadId)` (returns nothing); `showDefaultFolder()` (returns nothing); `erase(query)` resolves to erased `downloadId`s; `removeFile(downloadId)`; `acceptDanger(downloadId)`; `setShelfEnabled(enabled)` (deprecated Chrome 117; requires `"downloads.shelf"`); `setUiOptions(options)` (requires `"downloads.ui"`).
- **Key events:** `onCreated` → `(DownloadItem)`; `onChanged` → `(DownloadDelta)`; `onErased` → `(downloadId)`; `onDeterminingFilename` → `(downloadItem, suggest)` (override the target filename before the file is written).
- **Key types:** `DownloadItem` (`id`, `url`, `finalUrl`, `filename`, `incognito`, `danger`, `mime`, `startTime`, `endTime`, `state`, `paused`, `canResume`, `error`, `bytesReceived`, `totalBytes`, `exists`, `byExtensionId`, `referrer`); `DownloadOptions` (`url`, `filename`, `conflictAction`, `saveAs`, `method`, `headers`, `body`); `DownloadQuery` (filters incl. `orderBy`, `limit`, `urlRegex`, `filenameRegex`, `state`, `danger`).
- **Enums:** `FilenameConflictAction` (`"uniquify"`, `"overwrite"`, `"prompt"`); `HttpMethod` (`"GET"`, `"POST"`); `State` (`"in_progress"`, `"interrupted"`, `"complete"`); `DangerType` (`"file"`, `"url"`, `"content"`, `"uncommon"`, `"host"`, `"unwanted"`, `"safe"`, `"accepted"`, and policy/scan variants); `InterruptReason` (`"FILE_FAILED"`, `"FILE_ACCESS_DENIED"`, `"FILE_NO_SPACE"`, `"NETWORK_FAILED"`, `"NETWORK_TIMEOUT"`, `"SERVER_FAILED"`, `"SERVER_FORBIDDEN"`, `"USER_CANCELED"`, `"CRASH"`, among others).
- **MV3 notes:** async methods return Promises; `show()`/`showDefaultFolder()`/`setShelfEnabled()` are synchronous. Register download listeners synchronously at the top level of the service worker. `onDeterminingFilename` must call `suggest` (sync, or async after returning `true`).

```js
const downloadId = await chrome.downloads.download({
  url, filename: "reports/report.pdf", conflictAction: "uniquify", saveAs: false,
});
chrome.downloads.onChanged.addListener(function listener(delta) {
  if (delta.id === downloadId && delta.state?.current === "complete") {
    chrome.downloads.onChanged.removeListener(listener);
  }
});
```

## chrome.management
- **Purpose:** enumerate and manage installed extensions/apps — read metadata, enable/disable, uninstall, query permission warnings, launch apps.
- **Permission:** `"management"`. Exceptions (no permission required): `getSelf()`, `getPermissionWarningsByManifest()`, `uninstallSelf()`.
- **Key methods** (Promises): `getAll()` resolves to `ExtensionInfo[]`; `get(id)`; `getSelf()`; `getPermissionWarningsById(id)` resolves to `string[]`; `getPermissionWarningsByManifest(manifestStr)`; `setEnabled(id, enabled)`; `uninstall(id, options?)`; `uninstallSelf(options?)`; `launchApp(id)`; `createAppShortcut(id)`; `setLaunchType(id, launchType)`; `generateAppForLink(url, title)`; `installReplacementWebApp()`.
- **Key events:** `onInstalled` → `(info)`; `onUninstalled` → `(id)`; `onEnabled` → `(info)`; `onDisabled` → `(info)`.
- **Key types/enums:** `ExtensionInfo` (`id`, `name`, `shortName`, `description`, `version`, `enabled`, `disabledReason`, `type`, `installType`, `permissions`, `hostPermissions`, `icons`, `mayDisable`, `updateUrl`, `offlineEnabled`); `IconInfo` (`size`, `url`); `ExtensionDisabledReason` (`"unknown"`, `"permissions_increase"`); `ExtensionInstallType` (`"admin"`, `"development"`, `"normal"`, `"sideload"`, `"other"`); `ExtensionType` (`"extension"`, `"hosted_app"`, `"packaged_app"`, `"legacy_packaged_app"`, `"theme"`, `"login_screen_extension"`); `LaunchType` (`"OPEN_AS_REGULAR_TAB"`, `"OPEN_AS_PINNED_TAB"`, `"OPEN_AS_WINDOW"`, `"OPEN_FULL_SCREEN"`).
- **MV3 notes:** register `management` listeners synchronously at the top level of the worker. `getSelf()`/`getPermissionWarningsByManifest()`/`uninstallSelf()` work without the `"management"` permission.

```js
const all = await chrome.management.getAll();
const enabledExtensions = all.filter((i) => i.enabled && i.type === "extension");
```

## chrome.permissions
- **Purpose:** request additional permissions at runtime and remove them when unused, so the extension holds only the access it actively uses.
- **Permission:** none. The permissions it grants/removes must be declared under `optional_permissions` and/or `optional_host_permissions`.
- **Manifest keys:** `optional_permissions` (named permission strings requestable at runtime); `optional_host_permissions` (host match patterns requestable at runtime, e.g. `"https://*/*"`).
- **Key methods** (Promises): `contains(permissions)` resolves to a boolean; `getAll()` resolves to a `Permissions` object; `request(permissions)` resolves to a boolean — **must be called from inside a user gesture** (e.g. a button click); `remove(permissions)` resolves to a boolean (rejects if removing required permissions); `addHostAccessRequest(request)` (Chrome 133+); `removeHostAccessRequest(request)` (Chrome 133+).
- **Key events:** `onAdded` → `(permissions)`; `onRemoved` → `(permissions)`.
- **Key types:** `Permissions` (`permissions` — named permissions array; `origins` — host permissions array; both optional).
- **MV3 notes:** these permissions cannot be made optional (cannot appear in `optional_permissions`): `"debugger"`, `"declarativeNetRequest"`, `"devtools"`, `"geolocation"`, `"mdns"`, `"proxy"`, `"tts"`, `"ttsEngine"`, `"wallpaper"`. Because `request()` needs a user gesture, call it from UI interaction, not a worker lifecycle event.

```js
document.querySelector("#enable").addEventListener("click", async () => {
  const granted = await chrome.permissions.request({
    permissions: ["downloads"],
    origins: ["https://example.com/*"],
  });
  console.log(granted ? "granted" : "denied");
});
```
