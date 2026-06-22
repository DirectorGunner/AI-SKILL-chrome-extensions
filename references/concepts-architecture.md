# Extension Architecture & Getting Started

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
A Manifest V3 extension is a package of web files (HTML, CSS, JavaScript) plus a `manifest.json` that declares its components — a background service worker, content scripts, and UI surfaces — which run in separate contexts and talk to each other through the messaging APIs.

## The extension package
An extension is a directory built with the same web technologies as web apps (HTML, CSS, JavaScript), plus the Chrome Extension APIs (the `chrome.*` namespace). `manifest.json` is the only required file with a fixed name, and it must live in the extension's root directory.

## The manifest
`manifest.json` declares metadata and wires up every component.

```json
{
  "manifest_version": 3,
  "name": "Awesome Test Extension",
  "version": "1.0",
  "background": { "service_worker": "service-worker.js" },
  "action": { "default_popup": "popup.html" },
  "content_scripts": [
    { "matches": ["https://*.nytimes.com/*"], "js": ["content-script.js"] }
  ],
  "permissions": ["storage"],
  "host_permissions": ["https://*.nytimes.com/*"]
}
```

## Components and where they run
- **Service worker (background)** — registered via `background.service_worker`. Handles browser events; has no DOM (combine with an offscreen document for DOM work). See [concepts-service-workers.md](concepts-service-workers.md).
- **Content scripts** — run JavaScript in a web page's context, in an **isolated world** (their variables are not visible to the page or other extensions), yet share the page's DOM. A content script may call only a limited set of APIs directly: `dom`, `i18n`, `storage`, and parts of `runtime` (`connect`, `sendMessage`, `onMessage`, `onConnect`, `getManifest`, `getURL`, `id`). See [concepts-content-scripts.md](concepts-content-scripts.md).
- **UI surfaces** — the `action` toolbar icon/popup, the `sidePanel`, and the options page; these are ordinary extension HTML pages with full `chrome.*` access per their permissions.

## How the pieces communicate
Because components run in different contexts, they coordinate with message passing — one-time requests (`chrome.runtime.sendMessage`, `chrome.tabs.sendMessage`, `chrome.runtime.onMessage`) and long-lived connections (`chrome.runtime.connect`/`chrome.tabs.connect` opening a `runtime.Port`, accepted by `chrome.runtime.onConnect`). External senders use `chrome.runtime.onMessageExternal`/`onConnectExternal`, gated by the `externally_connectable` manifest key. Full patterns in [concepts-messaging.md](concepts-messaging.md).

```js
// content script asks the service worker to fetch a status
const response = await chrome.runtime.sendMessage({ greeting: "hello" });
```

## Network request handling
The `declarativeNetRequest` API intercepts, blocks, or modifies network requests declaratively (see [api-network-filtering.md](api-network-filtering.md)).
