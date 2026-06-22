# Offscreen, Messaging Services & Shared Type Namespaces

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
This cluster gathers the remaining runtime-adjacent APIs: `chrome.offscreen` (DOM access for a service worker), the cloud-messaging pair `chrome.gcm` and `chrome.instanceID`, the partly-legacy `chrome.extension` utilities, and three declaration-only namespaces — `chrome.types`, `chrome.extensionTypes`, and `chrome.dom`.

## chrome.offscreen
- **Purpose:** create a hidden document so a service worker can use DOM-dependent web APIs (clipboard, parsing, audio, WebRTC) that the worker lacks.
- **Permission:** `"offscreen"`.
- **Key methods** (Promises): `createDocument(parameters)` → resolves after the document's initial load; `closeDocument()`; `hasDocument()` → resolves to a `boolean`.
- **Key types:** `CreateParameters` = `{url (relative static HTML bundled with the extension), reasons, justification}`. `Reason` = `TESTING`, `AUDIO_PLAYBACK`, `IFRAME_SCRIPTING`, `DOM_SCRAPING`, `BLOBS`, `DOM_PARSER`, `USER_MEDIA`, `DISPLAY_MEDIA`, `WEB_RTC`, `CLIPBOARD`, `LOCAL_STORAGE`, `WORKERS`, `BATTERY_STATUS`, `MATCH_MEDIA`, `GEOLOCATION`.
- **MV3 notes:** available Chrome 109+. Only one offscreen document may be open at a time — call `hasDocument()` first. An `AUDIO_PLAYBACK` document is auto-closed after 30 seconds with no audio playing. Inside the offscreen page, `chrome.runtime` is the only extensions API available, so route work via messaging.

```js
async function ensureOffscreen() {
  if (await chrome.offscreen.hasDocument()) return;
  await chrome.offscreen.createDocument({
    url: "offscreen.html",
    reasons: ["CLIPBOARD"],
    justification: "Write to the clipboard from the service worker",
  });
}
```

## chrome.gcm
- **Purpose:** send and receive push messages through Firebase Cloud Messaging (FCM).
- **Permission:** `"gcm"`.
- **Key methods** (Promises): `register(senderIds)` accepts 1–100 sender IDs and resolves to a registration ID; `send(message)` resolves to a message ID; `unregister()`. The `send` message object carries `data` (keys may not start with `goog.`, `google`, or `collapse_key`), `destinationId`, `messageId`, and optional `timeToLive` (seconds in the range 0–2,419,200, default 86,400).
- **Key events:** `onMessage` (`{data, from?, collapseKey?}`), `onMessagesDeleted`, `onSendError` (`{errorMessage, messageId?, details?}`).
- **Constants:** `MAX_MESSAGE_SIZE` = 4096 bytes (sum of all key/value pairs).
- **MV3 notes:** all methods are Promise-based. Register listeners at the top level so an incoming push can wake the worker.

```js
const registrationId = await chrome.gcm.register(["SENDER_ID_PLACEHOLDER"]);
chrome.gcm.onMessage.addListener((message) => console.log("push:", message.data));
```

## chrome.instanceID
- **Purpose:** obtain a stable per-instance identifier and scoped tokens (used with FCM).
- **Permission:** `"gcm"`.
- **Key methods** (Promises): `getID()`; `getCreationTime()`; `getToken(getTokenParams)` (params `authorizedEntity`, `scope`, and a deprecated `options` ignored since Chrome 89); `deleteToken(deleteTokenParams)` (params `authorizedEntity`, `scope`); `deleteID()`.
- **Key events:** `onTokenRefresh` — fires when all granted tokens need to be re-fetched.
- **MV3 notes:** `deleteID()` resets the instance ID and invalidates every derived token, so re-acquire tokens afterward; treat `onTokenRefresh` as the cue to call `getToken` again.

```js
const id = await chrome.instanceID.getID();
const token = await chrome.instanceID.getToken({
  authorizedEntity: "PROJECT_ID_PLACEHOLDER",
  scope: "GCM",
});
```

## chrome.extension
- **Purpose:** a grab-bag of helpers usable from any extension page; in MV3 most of its messaging surface moved to `chrome.runtime`.
- **Permission:** none.
- **Key methods** (current in MV3): `getURL(path)` (prefer `chrome.runtime.getURL`); `getViews(fetchProperties?)` → returns `Window[]`; `isAllowedIncognitoAccess()` (Chrome 99+); `isAllowedFileSchemeAccess()` (Chrome 99+); `setUpdateUrlData(data)`.
- **Key types:** `ViewType` = `"tab"`, `"popup"` (Chrome 44+); property `inIncognitoContext`.
- **Migration context only** (deprecated MV2-era — do not use in new MV3 code): `getBackgroundPage()`, `getExtensionTabs()`, `sendRequest()` (use `chrome.runtime.sendMessage()`), `onRequest` (use `chrome.runtime.onMessage`), `onRequestExternal` (use `chrome.runtime.onMessageExternal`), and `lastError` (use `chrome.runtime.lastError`).

```js
const allowed = await chrome.extension.isAllowedIncognitoAccess();
console.log("incognito allowed:", allowed, "in incognito:", chrome.extension.inIncognitoContext);
```

## chrome.types
- **Purpose:** a declarations-only namespace; its central contribution is the `ChromeSetting` type that other APIs (such as `chrome.privacy` and `chrome.accessibilityFeatures`) expose to read and control a browser setting.
- **Permission:** none.
- **Key methods** (on a `ChromeSetting` object, Promise-based, Chrome 96+): `get(details)`; `set(details)`; `clear(details)`.
- **Key events:** `onChange` → `(details {value, levelOfControl, incognitoSpecific?})`.
- **Key types:** `ChromeSettingScope` = `"regular"`, `"regular_only"`, `"incognito_persistent"`, `"incognito_session_only"`; `LevelOfControl` = `"not_controllable"`, `"controlled_by_other_extensions"`, `"controllable_by_this_extension"`, `"controlled_by_this_extension"`.
- **MV3 notes:** `chrome.types` has no callable members of its own — you reach a `ChromeSetting` through another API. The most recently installed controlling extension wins; check `levelOfControl` before assuming a `set` will stick.

```js
const setting = chrome.privacy.network.networkPredictionEnabled; // a ChromeSetting
const details = await setting.get({});
if (details.levelOfControl === "controllable_by_this_extension") {
  await setting.set({ value: false, scope: "regular" });
}
```

## chrome.extensionTypes
- **Purpose:** a types-only namespace holding common enums and option shapes shared by several APIs (no methods, no events).
- **Permission:** none.
- **Key types / enums:** `ImageFormat` = `"jpeg"`, `"png"`; `ImageDetails` = `{format? (default "jpeg"), quality? (JPEG only)}`; `CSSOrigin` = `"author"`, `"user"`; `RunAt` = `"document_start"`, `"document_end"`, `"document_idle"` (default); `DocumentLifecycle` (Chrome 106+) = `"prerender"`, `"active"`, `"cached"`, `"pending_deletion"`; `FrameType` (Chrome 106+) = `"outermost_frame"`, `"fenced_frame"`, `"sub_frame"`; `ExecutionWorld` (Chrome 111+) = `"ISOLATED"`, `"MAIN"`, `"USER_SCRIPT"`; `InjectDetails` (`code` or `file`, plus `frameId`, `allFrames`, `matchAboutBlank`, `runAt`, `cssOrigin`); `ColorArray` (Chrome 139+) = `[number, number, number, number]`.
- **MV3 notes:** nothing to import or call — these names appear as the parameter/return types of methods on APIs like `chrome.scripting`, `chrome.tabs`, and `chrome.userScripts`.

## chrome.dom
- **Purpose:** special DOM helpers available to content scripts beyond the standard web platform.
- **Permission:** none.
- **Key methods:** `openOrClosedShadowRoot(element)` → returns the element's `ShadowRoot` (or `null`) — notably it can reach a *closed* shadow root that page script could not.
- **MV3 notes:** added Chrome 88+. Meant for the content-script context where there is a real DOM.

```js
// content script
const host = document.querySelector("#widget");
const root = chrome.dom.openOrClosedShadowRoot(host);
if (root) console.log("shadow children:", root.childElementCount);
```
