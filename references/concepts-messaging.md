# Message Passing

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Extensions pass messages between content scripts and the service worker, between extensions, and between web pages and extensions, using one-time requests, long-lived ports, and native messaging.

## One-time requests
Use `chrome.runtime.sendMessage` to message the extension from a content script (or vice versa), and `chrome.tabs.sendMessage` to message a content script. Both return Promises resolving to the recipient's response. Receive with `chrome.runtime.onMessage`; the listener gets `(message, sender, sendResponse)`.

```js
const response = await chrome.runtime.sendMessage({ greeting: "hello" });
```

### Returning true to keep the channel open (gotcha)
`sendResponse` is synchronous by default — once the listener returns, the channel closes. To respond asynchronously, **return `true`** from the listener so Chrome keeps the channel open until `sendResponse` runs. Forgetting this is a common bug: the response never arrives.

```js
chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message !== "get-status") return;
  fetch("https://example.com").then((r) => sendResponse({ statusCode: r.status }));
  return true; // keep the channel open for the async sendResponse
});
```

Alternatives: an `async` listener (always returns a Promise), or returning a Promise (Chrome 148+). Messages are JSON-serialized (`undefined` becomes `null`); maximum message size is 64 MiB.

## Long-lived connections (ports)
`chrome.runtime.connect` (from a content script) or `chrome.tabs.connect` (from the extension) opens a `Port`; `chrome.runtime.onConnect` accepts it. A `Port` provides `postMessage`, `onMessage`, `onDisconnect`, and `disconnect`. Pass a `name` to distinguish channels.

```js
const port = chrome.runtime.connect({ name: "knockknock" });
port.onMessage.addListener((msg) => { if (msg.question) port.postMessage({ answer: "Madame" }); });
port.postMessage({ joke: "Knock knock" });
// service worker
chrome.runtime.onConnect.addListener((port) => {
  if (port.name !== "knockknock") return;
  port.onMessage.addListener((msg) => { if (msg.joke) port.postMessage({ question: "Who's there?" }); });
});
```

`onDisconnect` fires when the other end has no `onConnect` listeners, the tab unloads/navigates, or `port.disconnect` is called.

## Cross-extension messaging
Listen with `chrome.runtime.onMessageExternal` / `chrome.runtime.onConnectExternal`; send by passing the target extension ID as the first argument to `chrome.runtime.sendMessage` / `chrome.runtime.connect`. The `sender` carries `sender.id`, `sender.tab`, `sender.url`, `sender.frameId`.

## Messages from web pages (externally_connectable)
A web page can message an extension when the extension allowlists the page's origin via the `externally_connectable` manifest key. Extensions cannot send messages *to* web pages this way.

```json
{ "externally_connectable": { "matches": ["https://*.example.com/*"] } }
```

## Security
Treat content scripts as less trustworthy than the service worker: validate input, never `eval()` a response or assign it to `innerHTML` — use `JSON.parse` and `innerText`.

## Native messaging
Exchange messages with a native app on the user's machine. Requires the `"nativeMessaging"` permission; **not** available in content scripts (route through the service worker). Connection-based: `chrome.runtime.connectNative('com.my_company.my_application')`. Connectionless: `chrome.runtime.sendNativeMessage('com.my_company.my_application', {text: 'Hello'}, cb)`.

```js
const port = chrome.runtime.connectNative("com.my_company.my_application");
port.onMessage.addListener((msg) => console.log("Received", msg));
port.postMessage({ text: "Hello, my_application" });
```

### Native host manifest
The native app ships a JSON manifest: `name` (lowercase alphanumerics, underscores, dots; no leading/trailing dot), `description`, `path` (host binary), `type` must be `"stdio"`, and `allowed_origins` (extension IDs allowed to connect; no wildcards).

```json
{
  "name": "com.my_company.my_application",
  "description": "My Application",
  "path": "C:\\Program Files\\My Application\\chrome_native_messaging_host.exe",
  "type": "stdio",
  "allowed_origins": ["chrome-extension://knldjmfmopnpolahpmmgbagdohdnhkik/"]
}
```

Install locations: Windows registers a key under `HKEY_LOCAL_MACHINE\SOFTWARE\Google\Chrome\NativeMessagingHosts\com.my_company.my_application` (or `HKEY_CURRENT_USER`); macOS uses `/Library/Google/Chrome/NativeMessagingHosts/` or `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/`; Linux uses `/etc/opt/chrome/native-messaging-hosts/` or `~/.config/google-chrome/NativeMessagingHosts/`.

### Protocol
Each message is JSON-serialized, UTF-8 encoded, and preceded by a 32-bit message length in native byte order. Size limits: native host to Chrome max 1 MB; Chrome to native host max 64 MiB. Chrome passes the caller origin (`chrome-extension://[extension-id]`) as the host's first argument and, on Windows, a `--parent-window` argument set to the decimal window handle (0 for service workers). On Windows the host's stdio must use `O_BINARY`, not the default `O_TEXT`.
