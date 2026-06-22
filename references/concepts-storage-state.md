# Storage, State & Cross-Origin Isolation

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Choosing among the `chrome.storage` areas, persisting service-worker state, reading cookies, and opting into cross-origin isolation for powerful web features.

## The chrome.storage areas
`chrome.storage` exposes four areas behind one async API; all require the `"storage"` permission.

- **local** — stored locally, cleared on uninstall, persists across sessions, not synced. Exposed to content scripts by default. `QUOTA_BYTES` = 10485760 (10 MB; 5 MB in Chrome 113 and earlier); raise it with `"unlimitedStorage"`.
- **sync** — syncs across the user's signed-in browsers (stored locally while offline). Quotas: `QUOTA_BYTES` = 102400, `QUOTA_BYTES_PER_ITEM` = 8192, `MAX_ITEMS` = 512, `MAX_WRITE_OPERATIONS_PER_MINUTE` = 120, `MAX_WRITE_OPERATIONS_PER_HOUR` = 1800.
- **session** — in memory while the extension is loaded; cleared on disable/reload/update and browser restart; not synced. Not exposed to content scripts by default. `QUOTA_BYTES` = 10485760 (10 MB; 1 MB in Chrome 111 and earlier).
- **managed** — set by enterprise policy; read-only for the extension.

Change content-script exposure with `setAccessLevel()` on the area.

```js
await chrome.storage.local.set({ key: value });
const result = await chrome.storage.local.get(["key"]);
chrome.storage.onChanged.addListener((changes, namespace) => {
  for (const [key, { oldValue, newValue }] of Object.entries(changes)) console.log(namespace, key, oldValue, newValue);
});
```

## Persisting service-worker state
An MV3 service worker is ephemeral, so in-memory variables are not durable. Inside the worker, `IndexedDB` and `Cache Storage` are available, but the Web Storage APIs (`Local Storage`, `Session Storage`) are not. Use `chrome.storage.local` for data that should survive restarts and `chrome.storage.session` for in-memory state to discard when the worker/browser stops. Extension storage is not cleared when the user clears browsing data.

## Cookies access
To read/write site cookies, declare the `"cookies"` permission plus `host_permissions` for the hosts involved.

```json
{ "permissions": ["cookies"], "host_permissions": ["*://*.google.com/"] }
```

```js
const cookie = await chrome.cookies.get({ url: "https://www.google.com", name: "sessionId" });
const all = await chrome.cookies.getAll({ domain: "google.com" });
```

Cookies set by `chrome-extension://` pages use `SameSite=Lax` and cannot use the `Secure` attribute.

## Cross-origin isolation
Some powerful features (e.g. `SharedArrayBuffer`) require a cross-origin-isolated document. Opt in with two manifest keys, each an object with a `value`: `cross_origin_embedder_policy` = `"require-corp"` and `cross_origin_opener_policy` = `"same-origin"`.

```json
{
  "cross_origin_embedder_policy": { "value": "require-corp" },
  "cross_origin_opener_policy": { "value": "same-origin" }
}
```

Isolation is not fully implemented for service/shared workers, and web-accessible subframes embedded on regular pages are not considered cross-origin isolated.
