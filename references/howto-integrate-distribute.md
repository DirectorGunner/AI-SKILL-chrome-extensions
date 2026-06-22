# Integrate & Distribute

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Wiring an MV3 extension to Google services (OAuth via `chrome.identity`, Google Analytics 4 via the Measurement Protocol) and installing/distributing it, including loading an unpacked build during development.

## OAuth 2.0 with chrome.identity
The Identity API handles the OAuth flow. Methods: `chrome.identity.getAuthToken()` (Google token; pass `{interactive: true}` for consent), `chrome.identity.launchWebAuthFlow()` (generic / non-Google providers), `chrome.identity.removeCachedAuthToken()`, `chrome.identity.clearAllCachedAuthTokens()`. Requires the `identity` permission. Declare the `oauth2` block; the `key` field pins a stable extension ID (use a placeholder until you have the real key).

```json
{
  "permissions": ["identity"],
  "oauth2": {
    "client_id": "YOUR_CLIENT_ID.apps.googleusercontent.com",
    "scopes": ["https://www.googleapis.com/auth/userinfo.email"]
  },
  "key": "PUBLIC_KEY_HERE"
}
```

Procedure: register an OAuth client of type **Chrome Extension** in the Google API Console with the extension's **Item ID**; pin the ID by copying the public key from the Developer Dashboard **Package** tab into `key`; enable the target API. Then request the token and attach it as a bearer credential (`API_KEY` is a placeholder).

```js
chrome.identity.getAuthToken({ interactive: true }, (token) => {
  fetch("https://people.googleapis.com/v1/contactGroups/all?key=API_KEY", {
    headers: { Authorization: "Bearer " + token, "Content-Type": "application/json" },
  }).then((r) => r.json()).then(console.log);
});
```

## Google Analytics 4
MV3 forbids remote code, so you cannot load the analytics script; send events over the GA4 **Measurement Protocol** as HTTP requests (works from any context, including the service worker). Setup: add a **Web** data stream (record the **Measurement ID**, format `G-XXXXXXXXXX`), create a **Measurement Protocol API secret**, and add the `storage` permission. Persist a `clientId` in `chrome.storage.local` and `sessionData` in `chrome.storage.session`. POST to the endpoint `https://www.google-analytics.com/mp/collect` with `client_id` and an `events` array; include `session_id` and `engagement_time_msec` for Realtime. Validate via `https://www.google-analytics.com/debug/mp/collect`.

```js
fetch(`https://www.google-analytics.com/mp/collect?measurement_id=${MEASUREMENT_ID}&api_secret=${API_SECRET}`, {
  method: "POST",
  body: JSON.stringify({ client_id: await getOrCreateClientId(), events: [{ name: "button_clicked", params: { session_id, engagement_time_msec: 100 } }] }),
});
```

## Distribution and installation
The Chrome Web Store is the primary channel (publish there before using alternatives; verify at `chrome://extensions`).
- **Windows registry** — under `HKEY_LOCAL_MACHINE\Software\Google\Chrome\Extensions` (or `...\Wow6432Node\...` for 64-bit), add a key named with the extension ID and set `"update_url": "https://clients2.google.com/service/update2/crx"`.
- **Preferences file (macOS/Linux)** — a JSON file named after the extension ID under the browser's `External Extensions` directory, with `"external_update_url"` (Web Store), or `"external_crx"` + `"external_version"` (local file), and optional `"supported_locales"`.
- **Enterprise policies** — administrators deploy via policy.

## Loading an unpacked extension (development)
1. Open `chrome://extensions`.
2. Turn on the **Developer mode** toggle.
3. Click **Load unpacked** and select the extension directory (the folder with `manifest.json`).
4. Pin the action icon to the toolbar.
5. After editing code, click the refresh icon next to the extension to reload.

```json
{
  "name": "Hello Extensions", "description": "Base Level Extension", "version": "1.0",
  "manifest_version": 3,
  "action": { "default_popup": "hello.html", "default_icon": "hello_extensions.png" }
}
```
