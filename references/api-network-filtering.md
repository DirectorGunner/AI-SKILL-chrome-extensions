# Network Filtering & Request APIs

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
How an MV3 extension shapes, observes, and configures network traffic: declarative rule-based filtering, observational request monitoring, page-state action enabling, proxy config, cookie access, and DNS lookup. In MV3, blocking the network is primarily a declarative task.

## chrome.declarativeNetRequest
- **Purpose:** block or modify network requests through declarative rules the browser evaluates internally, so the extension never sees request content.
- **Permission:** `"declarativeNetRequest"` (grants `block`, `allow`, `allowAllRequests` rules; shows an install warning) or `"declarativeNetRequestWithHostAccess"` (no warning but requires `host_permissions`). `"declarativeNetRequestFeedback"` enables `getMatchedRules()` and `onRuleMatchedDebug`.
- **Manifest key:** `"declarative_net_request"` with `"rule_resources"`, an array of `{ "id", "enabled", "path" }` objects pointing at static ruleset JSON files.
- **Rule shape:** `{ id, priority, action, condition }`. `action.type`: `block`, `allow`, `allowAllRequests`, `upgradeScheme`, `redirect`, `modifyHeaders`. `condition` includes `urlFilter`, `regexFilter`, `resourceTypes`, `initiatorDomains`/`excludedInitiatorDomains`, `requestDomains`/`excludedRequestDomains`, `requestMethods`, `domainType` (`firstParty`/`thirdParty`), `tabIds`/`excludedTabIds`.
- **Key methods** (Promises): `updateDynamicRules(options)`, `getDynamicRules(filter?)`, `updateSessionRules(options)`, `getSessionRules(filter?)`, `updateEnabledRulesets(options)`, `getEnabledRulesets()`, `updateStaticRules(options)`, `getDisabledRuleIds(options)`, `getAvailableStaticRuleCount()`, `getMatchedRules(filter?)`, `isRegexSupported(regexOptions)`, `testMatchOutcome(request)`, `setExtensionActionOptions(options)`.
- **Key event:** `onRuleMatchedDebug` (unpacked extensions only; requires the feedback permission).
- **Numeric limits (verbatim):** `GUARANTEED_MINIMUM_STATIC_RULES` = 30000; `MAX_NUMBER_OF_STATIC_RULESETS` = 100; `MAX_NUMBER_OF_ENABLED_STATIC_RULESETS` = 50; `MAX_NUMBER_OF_DYNAMIC_RULES` = 30000; `MAX_NUMBER_OF_UNSAFE_DYNAMIC_RULES` = 5000; `MAX_NUMBER_OF_SESSION_RULES` = 5000; `MAX_NUMBER_OF_REGEX_RULES` = 1000; `MAX_GETMATCHEDRULES_CALLS_PER_INTERVAL` = 20; `GETMATCHEDRULES_QUOTA_INTERVAL` = 10 minutes.

```js
await chrome.declarativeNetRequest.updateDynamicRules({
  removeRuleIds: [1],
  addRules: [{
    id: 2, priority: 1,
    action: { type: "block" },
    condition: { requestDomains: ["ads.example"], resourceTypes: ["script"] },
  }],
});
```

## chrome.webRequest
- **Purpose:** observe and analyze web traffic, and (where permitted) intercept requests in flight.
- **Permission:** `"webRequest"` plus `host_permissions`. `"webRequestBlocking"` enables synchronous blocking handlers, but **in Manifest V3 it is restricted to policy-installed extensions** — the MV3 path for blocking is `declarativeNetRequest`. Non-blocking (observational) webRequest remains fully available in MV3. `"webRequestAuthProvider"` is required for `onAuthRequired`.
- **Key events** (lifecycle order): `onBeforeRequest`, `onBeforeSendHeaders`, `onSendHeaders`, `onHeadersReceived`, `onAuthRequired`, `onBeforeRedirect`, `onResponseStarted`, `onCompleted`, `onErrorOccurred`.
- **Registration:** `addListener(callback, filter, opt_extraInfoSpec)`. `filter` (`RequestFilter`) has `urls`, `types`, `tabId`, `windowId`. `opt_extraInfoSpec`: `"blocking"`, `"asyncBlocking"`, `"requestHeaders"`, `"responseHeaders"`, `"extraHeaders"`, `"requestBody"`.
- **Key types:** `ResourceType` (`main_frame`, `sub_frame`, `stylesheet`, `script`, `image`, `font`, `xmlhttprequest`, `ping`, `media`, `websocket`, `webbundle`, `other`); `BlockingResponse` (`cancel`, `redirectUrl`, `requestHeaders`, `responseHeaders`, `authCredentials`).

```js
chrome.webRequest.onCompleted.addListener(
  (details) => console.log(details.url, details.statusCode),
  { urls: ["https://*.example.com/*"] }
);
```

## chrome.declarativeContent
- **Purpose:** enable the toolbar action based on a page's URL or matching CSS selector, without host permissions or injecting a content script.
- **Permission:** `"declarativeContent"`.
- **Rules:** `{ conditions, actions }`. Condition `PageStateMatcher` matches `pageUrl` (a UrlFilter), `css` (selectors), and `isBookmarked` (requires `"bookmarks"`). Actions: `ShowAction` (Chrome 97+), `SetIcon`, `RequestContentScript`.
- **Key event:** `onPageChanged` with `addRules()`, `removeRules()`, `getRules()`.

```js
chrome.declarativeContent.onPageChanged.addRules([{
  conditions: [new chrome.declarativeContent.PageStateMatcher({
    pageUrl: { hostSuffix: ".example.com" }, css: ["input[type='password']"],
  })],
  actions: [new chrome.declarativeContent.ShowAction()],
}]);
```

## chrome.proxy
- **Permission:** `"proxy"`. `chrome.proxy.settings` is a `ChromeSetting` exposing `get()`, `set()`, `clear()`. `ProxyConfig.mode`: `"direct"`, `"auto_detect"`, `"pac_script"`, `"fixed_servers"`, `"system"`. `ProxyServer` has `host` (required), `scheme` (`http`, `https`, `quic`, `socks4`, `socks5`; default `http`), `port`. **Event:** `onProxyError`.

```js
chrome.proxy.settings.set({
  value: { mode: "fixed_servers", rules: { singleProxy: { scheme: "socks5", host: "127.0.0.1", port: 1080 } } },
  scope: "regular",
}, () => {});
```

## chrome.cookies
- **Purpose:** query, modify, and monitor HTTP cookies across cookie stores.
- **Permission:** `"cookies"` plus `host_permissions` for each domain accessed.
- **Key methods** (Promises, Chrome 88+): `get(details)` resolves to a `Cookie` or `undefined`; `getAll(details)` resolves to `Cookie[]`; `set(details)`; `remove(details)`; `getAllCookieStores()`.
- **Key types:** `Cookie` (`name`, `value`, `domain`, `hostOnly`, `path`, `secure`, `httpOnly`, `sameSite`, `session`, `expirationDate?`, `storeId`, `partitionKey?`); `SameSiteStatus` = `"no_restriction"`, `"lax"`, `"strict"`, `"unspecified"`; `CookieStore` (`id`, `tabIds`).
- **Key event:** `onChanged` (`{cookie, removed, cause}`).

```js
await chrome.cookies.set({
  url: "https://example.com", name: "user_pref", value: "dark", secure: true, sameSite: "lax",
});
```

## chrome.dns
- **Availability:** Dev channel only. **Permission:** `"dns"`. **Method:** `resolve(hostname)` resolves to `ResolveCallbackResolveInfo` (`resultCode` — zero means success; `address?`). Omit the scheme and trailing slash from the hostname.

```js
const record = await chrome.dns.resolve("example.com");
console.log(record.resultCode, record.address);
```
