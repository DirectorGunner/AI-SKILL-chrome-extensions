# Test & Debug

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Inspecting a running MV3 extension (service worker, popup, content scripts) from `chrome://extensions`, and driving automated tests with Jest, Puppeteer, and Selenium.

## Debugging a loaded extension
Load the extension unpacked first (see [howto-integrate-distribute.md](howto-integrate-distribute.md)), then debug from `chrome://extensions`.

- **Service worker** — click the blue **Inspect views** link (the service worker entry) to open DevTools; `console.log` output appears there. A red **Errors** button surfaces registration/runtime errors (**Clear all** resets it). Control the worker lifecycle via the **Service Workers** pane under the **Application** panel.
- **Popup** — right-click the open popup → **Inspect** for its own DevTools.
- **Content scripts** — errors appear in the host page's console; use the console context dropdown (next to **top**) to select your extension.

After any code change, reload the extension; ensure required APIs are declared as permissions.

## Unit testing with Jest
Test logic that does not touch `chrome` with plain Jest; mock the APIs otherwise. Register a setup file in `jest.config.js`, provide a global stub, and override per test with `jest.spyOn(...).mockResolvedValue(...)`.

```js
// jest.config.js
module.exports = { setupFiles: ["<rootDir>/mock-extension-apis.js"] }; // allow-placeholder
```

```js
// mock-extension-apis.js
global.chrome = { tabs: { query: async () => { throw new Error("Unimplemented."); } } };
// a test
test("getActiveTabId returns active tab ID", async () => {
  jest.spyOn(chrome.tabs, "query").mockResolvedValue([{ id: 3, active: true, currentWindow: true }]);
  expect(await getActiveTabId()).toBe(3);
});
```

## End-to-end testing
Supported drivers: Puppeteer, Playwright, Selenium (via `ChromeOptions` / ChromeDriver), and WebDriverIO. Old headless mode cannot load extensions — run with `--headless=new`. Fix the extension ID (via `key`) for allow-listing, and reach extension pages by URL (`chrome-extension://EXTENSION_ID/index.html`). Open the popup with `action.openPopup()` or by navigating to its URL.

## Driving an extension with Puppeteer
Install (`npm install puppeteer jest`), launch Chrome with the extension loaded, and grab the service worker target.

```js
browser = await puppeteer.launch({ headless: false, pipe: true, enableExtensions: [EXTENSION_PATH] });
const workerTarget = await browser.waitForTarget(
  (target) => target.type() === "service_worker" && target.url().endsWith("service-worker.js")
);
const worker = await workerTarget.worker();
```

```js
test("popup renders correctly", async () => {
  const page = await browser.newPage();
  await page.goto("https://example.com");
  await worker.evaluate("chrome.action.openPopup();");
  const popupTarget = await browser.waitForTarget((t) => t.type() === "page" && t.url().endsWith("popup.html"));
  const popup = await popupTarget.asPage();
  expect((await (await page.$("ul")).$$("li")).length).toBe(1);
});
```

Caveat: under Selenium/ChromeDriver the service worker may not terminate automatically as it would in normal usage.
