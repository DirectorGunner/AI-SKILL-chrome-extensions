# Security & Privacy

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
The MV3 security model: ship all code in the package, lock down `content_security_policy`, request minimal permissions, and handle user data carefully.

## No remotely hosted code
Remotely hosted code (RHC) is anything the browser executes that is loaded from outside the extension's own files (data such as JSON or CSS does not count). MV3 requires all executable code to ship inside the package; dynamically loading scripts or WASM from external URLs is prohibited. To find violations, search the bundled output (including dependencies) for `"http://"`/`"https://"`. Remediation: bundle remote dependencies locally at build time, tree-shake, or delete unused code; reference bundled resources with `chrome.runtime.getURL()`. Narrow exceptions: the User Scripts API (developer mode), `chrome.debugger` (shows a warning banner), and sandboxed iframes.

## Content Security Policy
In MV3 `content_security_policy` is an object with two keys:
- `extension_pages` — governs `chrome-extension://` contexts (HTML pages and the service worker), e.g. `"default-src 'self'"`, or add `'wasm-unsafe-eval'` to permit WebAssembly.
- `sandbox` — applies only to sandboxed extension pages (more permissive).

```json
{ "content_security_policy": { "extension_pages": "default-src 'self'" } }
```

For `extension_pages`, the `script-src`, `object-src`, and `worker-src` directives allow only `'self'`, `'none'`, and `'wasm-unsafe-eval'` (unpacked extensions may also use `http://localhost` and `http://127.0.0.1`, any port).

## Web-accessible resources
`web_accessible_resources` makes extension files detectable by web pages, so keep it minimal. The MV3 form pairs `resources` with `matches`:

```json
"web_accessible_resources": [{ "resources": ["test1.png"], "matches": ["https://example.com/*"] }]
```

## Safe handling of user data and DOM
Treat content scripts as untrusted: assume their messages could be attacker-crafted, validate senders, and never send sensitive data to content scripts. Avoid `document.write()` and `innerHTML`; build the DOM explicitly (`createElement` + `innerText`).

## Privacy and minimal permissions
Request only the permissions core features need. `activeTab` grants temporary access to the active tab only when the extension is invoked — a less invasive alternative to broad host permissions like `<all_urls>` — and shows no warning. Mark non-essential features as optional so you can explain why a permission matters when the user enables it. Extension storage is not encrypted, so keep sensitive data on secure HTTPS servers and off the client where possible. Honor incognito by checking the `incognito` property on `tabs.Tab`/`windows.Window` and not retaining data from those sessions. <!-- allow-placeholder -->
