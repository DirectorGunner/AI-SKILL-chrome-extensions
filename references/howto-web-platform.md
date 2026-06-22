# Web Platform How-Tos

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Using web-platform features from an MV3 extension: geolocation, screen capture, WebSockets, WebHID, WebUSB, and origin trials — including workarounds because the service worker has no DOM.

## Geolocation
Use `navigator.geolocation` `getCurrentPosition()` with `"geolocation"` in `permissions`. Popups, the side panel, and content scripts have a document context and can call it directly (content scripts still get the normal consent prompt). The service worker has no `document`, so bundle an offscreen document: request `"offscreen"`, create it with `chrome.offscreen.createDocument` (reason `Reason.GEOLOCATION`), call `getCurrentPosition` inside, message the result back, then `chrome.offscreen.closeDocument`.

## Screen capture
Capture with `navigator.mediaDevices.getDisplayMedia`, which prompts the user. To record a specific tab, call `chrome.tabCapture.getMediaStreamId` after a user gesture (Chrome 116+), hand the id to an offscreen document, and there call `getUserMedia` with `chromeMediaSource: "tab"`. For background recording across navigations, run capture in an offscreen document created with the `'USER_MEDIA'` or `'DISPLAY_MEDIA'` reason.

```js
const stream = await navigator.mediaDevices.getDisplayMedia({ audio: true, video: true });
```

## WebSockets (keeping the service worker alive)
Extension service workers support `WebSocket` from Chrome 116 (`"minimum_chrome_version": "116"`). Because the worker shuts down after 30 seconds of inactivity, send a keepalive (e.g. every 20 seconds, leaving a 10-second margin); receiving WebSocket activity also resets the idle timer.

```js
function keepAlive() {
  const id = setInterval(() => { if (webSocket) webSocket.send("keepalive"); else clearInterval(id); }, 20 * 1000);
}
```

## WebHID
Use `navigator.hid` (Chrome 117+); no manifest permission/key is required (the browser shows its own prompt). `navigator.hid.requestDevice()` needs a user gesture and a page context — call it from a popup/extension page, then message the worker, which enumerates with `navigator.hid.getDevices()` and opens each `HIDDevice`. The worker stays alive while a device is connected.

```js
// popup.js
await navigator.hid.requestDevice({ filters: [{ vendorId: 0x1234, productId: 0x5678 }] });
chrome.runtime.sendMessage("newDevice");
// service-worker.js
chrome.runtime.onMessage.addListener(async (m) => {
  if (m === "newDevice") for (const d of await navigator.hid.getDevices()) await d.open();
});
```

## WebUSB
Use `navigator.usb` (Chrome 118+); no manifest permission/key required. As with WebHID, `navigator.usb.requestDevice()` cannot run in the worker — call it from a page, message the worker, and enumerate with `navigator.usb.getDevices()`, opening each `USBDevice`.

## Origin trials
Register the extension on the Chrome Origin Trials page using a stable extension ID (origin `chrome-extension://abcdefghijklmnopqrstuvwxyz`), then add the issued token(s) to the `trial_tokens` manifest key (an array). Some features also require an API permission. Confirm activation in the DevTools Application panel. Manifest tokens do not activate trials in content-script-injected pages; for those, create a third-party-match token and inject it as a meta tag.

```js
const otMeta = document.createElement("meta");
otMeta.httpEquiv = "origin-trial";
otMeta.content = "TOKEN_GOES_HERE";
document.head.append(otMeta);
```
