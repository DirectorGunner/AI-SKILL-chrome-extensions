# Declaring Permissions & Match Patterns

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
How an MV3 extension requests capabilities and host access through manifest keys, requests host access at runtime, and how those declarations map to user-facing warnings.

## Permission manifest keys
MV3 splits capability into named API permissions and host (origin) access, each with an install-time key and an optional (runtime) counterpart:
- `permissions` — known permission strings (e.g. `"storage"`, `"contextMenus"`), granted at install time.
- `optional_permissions` — named permissions granted by the user at runtime.
- `host_permissions` — match patterns for host access, granted at install time.
- `optional_host_permissions` — host patterns granted at runtime.

In MV3 host access lives in its own keys rather than being mixed into `permissions` as in MV2 (migration context).

```json
{
  "permissions": ["activeTab", "contextMenus", "storage"],
  "optional_permissions": ["topSites"],
  "host_permissions": ["https://www.developer.chrome.com/*"],
  "optional_host_permissions": ["https://*/*", "http://*/*"],
  "manifest_version": 3
}
```

## Requesting permissions at runtime
Request optional permissions/hosts with `chrome.permissions.request`, which resolves to a boolean. It **must be called from inside a user gesture** (e.g. a click handler). Anything requested must already be declared in the manifest's optional keys. Related methods (Promises): `contains`, `getAll`, `remove`, plus events `onAdded`/`onRemoved`.

```js
document.querySelector("#go").addEventListener("click", () => {
  chrome.permissions.request({ permissions: ["tabs"], origins: ["https://www.google.com/"] }, (granted) => {
    if (granted) doSomething(); else doSomethingElse();
  });
});
```

The argument is a `Permissions` object with optional `permissions` (named strings) and `origins` (host patterns) arrays.

## The `activeTab` permission
`activeTab` grants temporary access to the currently active tab without a broad-host warning. When active, the extension can run `scripting.insertCSS()`/`scripting.executeScript()` (with `"scripting"` also declared), read tab metadata (URL, title, favicon), and use `webRequest` for the tab's main-frame origin. Access is granted only after an explicit user action — invoking the `action`, a `contextMenus` item, a `commands` shortcut, or an `omnibox` suggestion — and is revoked when the user navigates to a different origin or closes the tab (same-origin navigation keeps it).

```js
chrome.action.onClicked.addListener((tab) => {
  if (!tab.url.includes("chrome://")) {
    chrome.scripting.executeScript({ target: { tabId: tab.id }, func: () => { document.body.style.backgroundColor = "red"; } });
  }
});
```

## Host permission match patterns
Patterns have the form `scheme://host/path`:
- **Scheme** — `http`, `https`, `file`, or `*` (the `*` scheme matches only `http`/`https`).
- **Host** — a literal host (`www.example.com`), a subdomain wildcard (`*.example.com`), or a bare `*`. A wildcard in the host must be first and followed by a `.` or `/`.
- **Path** — required but ignored for host permissions; may use `*`.

Special cases: the `<all_urls>` token matches any URL with a permitted scheme; `file:///` needs three slashes and a manual user grant; use `http://localhost/*` or `http://127.0.0.1/*` for local; Chrome does not support patterns for top-level domains. <!-- allow-placeholder -->

```json
{ "host_permissions": ["https://*/*", "https://*.google.com/foo*bar", "file:///foo*", "http://localhost/*", "*://mail.google.com/*"] }
```

## How permission warnings work
Warnings derive from declared `permissions` and `host_permissions`. `activeTab` shows no install warning (temporary access only). Broad host patterns can subsume narrower warnings (the `"tabs"` warning is not shown if the extension also requests `<all_urls>`). Move intrusive capabilities into `optional_permissions`/`optional_host_permissions` to reduce install friction; on update, adding a warning-triggering permission may disable the extension until the user re-accepts. <!-- allow-placeholder -->
