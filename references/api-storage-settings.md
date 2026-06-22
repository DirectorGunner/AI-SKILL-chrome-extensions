# Storage & Browser Settings

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Persist extension data (`chrome.storage`) and read or override per-site and browser-wide preferences for content permissions, privacy features, fonts, and stored browsing data. All methods return a Promise in MV3.

## chrome.storage
- **Purpose:** persist key-value data, usable from every extension context including service workers and content scripts.
- **Permission:** `"storage"` (add `"unlimitedStorage"` to lift the `local` quota).
- **Areas:** `local` (cleared on uninstall), `sync` (synced across signed-in browsers; behaves like `local` when sync is off), `session` (in-memory; cleared on reload/restart; Chrome 102+), `managed` (read-only enterprise policy).
- **Key methods** (on each `StorageArea`, all return a Promise): `get(keys?)`, `set(items)`, `remove(keys)`, `clear()`, `getBytesInUse(keys?)`, `getKeys()`, `setAccessLevel({ level })`.
- **Key event:** `chrome.storage.onChanged(changes, areaName)`; each area also exposes its own `onChanged`.
- **Key types:** `StorageChange` `{ oldValue?, newValue? }`; `AccessLevel` = `"TRUSTED_CONTEXTS"`, `"TRUSTED_AND_UNTRUSTED_CONTEXTS"`.
- **Quota constants (verbatim):** `local.QUOTA_BYTES` = 10,485,760; `sync.QUOTA_BYTES` = 102,400; `sync.QUOTA_BYTES_PER_ITEM` = 8,192; `sync.MAX_ITEMS` = 512; `sync.MAX_WRITE_OPERATIONS_PER_HOUR` = 1,800; `sync.MAX_WRITE_OPERATIONS_PER_MINUTE` = 120; `session.QUOTA_BYTES` = 10,485,760.
- **MV3 notes:** service workers have full access; prefer `session` for worker state. `session` is hidden from content scripts unless you call `setAccessLevel({ level: "TRUSTED_AND_UNTRUSTED_CONTEXTS" })`.

```js
await chrome.storage.local.set({ counter: 1 });
const { counter } = await chrome.storage.local.get(["counter"]);
chrome.storage.onChanged.addListener((changes, areaName) => {
  for (const [key, { oldValue, newValue }] of Object.entries(changes)) {
    console.log(areaName, key, oldValue, newValue);
  }
});
```

## chrome.contentSettings
- **Permission:** `"contentSettings"`.
- **Key methods** (per `ContentSetting`, return a Promise): `get(details)` (`primaryUrl` required), `set(details)` (`primaryPattern` + `setting` required), `clear(details)`, `getResourceIdentifiers()`.
- **Setting types:** `cookies`, `images`, `javascript`, `location`, `popups`, `notifications`, `camera`, `microphone`, `automaticDownloads`, `clipboard`, `autoVerify`; deprecated `fullscreen`/`mouselock` (always `allow`), `plugins`/`unsandboxedPlugins` (always `block`).
- **Key types:** `Scope` = `"regular"`, `"incognito_session_only"`. Primary patterns take precedence over secondary.

```js
await chrome.contentSettings.javascript.set({
  primaryPattern: "https://*.example.com/*", setting: "block",
});
```

## chrome.privacy
- **Permission:** `"privacy"`.
- **Property groups (each a `ChromeSetting`):** `privacy.network` (`networkPredictionEnabled`, `webRTCIPHandlingPolicy`); `privacy.services` (`safeBrowsingEnabled`, `autofillAddressEnabled`, `autofillCreditCardEnabled`, `passwordSavingEnabled`, `searchSuggestEnabled`); `privacy.websites` (`thirdPartyCookiesAllowed`, `hyperlinkAuditingEnabled`, `referrersEnabled`, `doNotTrackEnabled`, `protectedContentEnabled`, `adMeasurementEnabled`, `fledgeEnabled`, `topicsEnabled`, `relatedWebsiteSetsEnabled`).
- **`ChromeSetting`:** `get(details)`, `set(details)`, `clear(details)`, event `onChange`.
- **Enums:** `IPHandlingPolicy` = `"default"`, `"default_public_and_private_interfaces"`, `"default_public_interface_only"`, `"disable_non_proxied_udp"`.
- **MV3 gotchas:** `adMeasurementEnabled`, `fledgeEnabled`, `topicsEnabled`, `relatedWebsiteSetsEnabled` may only be set to `false` (setting `true` throws). `thirdPartyCookiesAllowed` cannot be enabled in Incognito.

```js
await chrome.privacy.services.safeBrowsingEnabled.set({ value: true });
```

## chrome.fontSettings
- **Permission:** `"fontSettings"`.
- **Key methods** (return a Promise, Chrome 96+): `getFont(details)`, `setFont(details)`, `clearFont(details)`, `getFontList()`; size trios `get/set/clearDefaultFontSize`, `…DefaultFixedFontSize`, `…MinimumFontSize`.
- **Key types:** `GenericFamily` = `"standard"`, `"sansserif"`, `"serif"`, `"fixed"`, `"cursive"`, `"fantasy"`, `"math"`; `ScriptCode` (ISO 15924, `"Zyyy"` = global default).
- **Events:** `onFontChanged`, `onDefaultFontSizeChanged`, `onDefaultFixedFontSizeChanged`, `onMinimumFontSizeChanged`.

```js
await chrome.fontSettings.setFont({ genericFamily: "serif", script: "Latn", fontId: "Georgia" });
```

## chrome.browsingData
- **Permission:** `"browsingData"`.
- **Key methods** (return a Promise): `remove(options, dataToRemove)`, plus targeted `removeCache`, `removeCacheStorage`, `removeCookies`, `removeDownloads`, `removeFileSystems`, `removeFormData`, `removeHistory`, `removeIndexedDB`, `removeLocalStorage`, `removeServiceWorkers`, `removeWebSQL`, and `settings()`.
- **Key types:** `RemovalOptions` `{ since?, originTypes?, origins?, excludeOrigins? }` (`since` is ms since epoch, default 0); `DataTypeSet` booleans include `cache`, `cacheStorage`, `cookies`, `downloads`, `fileSystems`, `formData`, `history`, `indexedDB`, `localStorage`, `passwords`, `serviceWorkers`, `webSQL`.

```js
await chrome.browsingData.remove(
  { since: Date.now() - 60 * 60 * 1000 },
  { cache: true, cookies: true }
);
```
