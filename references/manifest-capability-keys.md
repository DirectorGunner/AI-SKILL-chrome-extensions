# Manifest Capability Keys

> Manifest V3 reference. Original prose; identifiers preserved verbatim. MV2-only forms are labeled migration context.

## Overview
These keys wire up an extension's behavior: toolbar UI, background logic, injected scripts, permissions, security policy, and exposed surfaces.

## `action`
Defines the toolbar button (MV3's single key replacing MV2 `browser_action`/`page_action`). Sub-fields: `default_icon`, `default_title` (tooltip), `default_popup` (HTML path).

```json
{ "action": { "default_icon": { "16": "icon16.png" }, "default_title": "Click Me", "default_popup": "popup.html" } }
```

## `background`
Registers the service worker. Sub-fields: `service_worker` (path), `type` (`"module"` for ES module imports). Migration context: MV2's `scripts` array and `persistent` boolean are not used in MV3.

```json
{ "background": { "service_worker": "service-worker.js", "type": "module" } }
```

## `content_scripts`
Array of static injection entries; `matches` plus at least one of `js`/`css` required. Sub-fields: `matches`, `exclude_matches`, `include_globs`, `exclude_globs`, `css`, `js`, `all_frames` (default `false`), `match_about_blank` (default `false`), `match_origin_as_fallback` (default `false`), `run_at` (`document_start`/`document_end`/`document_idle`, default `document_idle`), `world` (`ISOLATED`/`MAIN`, default `ISOLATED`).

## `permissions`, `optional_permissions`, `host_permissions`, `optional_host_permissions`
`permissions` and `host_permissions` are install-time named/host access; the `optional_*` counterparts are requested at runtime via `chrome.permissions.request`. In MV3, host access lives in its own keys (migration context: MV2 mixed hosts into `permissions`).

```json
{
  "permissions": ["activeTab", "storage"],
  "optional_permissions": ["topSites"],
  "host_permissions": ["https://www.developer.chrome.com/*"],
  "optional_host_permissions": ["https://*/*", "http://*/*"]
}
```

## `content_security_policy`
Object (a string in MV2 — migration context) with `extension_pages` and `sandbox`. `extension_pages` cannot load remote code, use `'unsafe-eval'`, or run inline scripts; the minimum enforced policy is `script-src 'self' 'wasm-unsafe-eval'; object-src 'self';`.

```json
{ "content_security_policy": { "extension_pages": "script-src 'self'; object-src 'self';" } }
```

## `web_accessible_resources`
Array of objects (a flat string array in MV2 — migration context). Sub-fields: `resources` (paths, wildcards allowed), and `matches` (URL patterns) or `extension_ids` (at least one required), plus `use_dynamic_url`. Nothing is web-accessible by default.

```json
{ "web_accessible_resources": [{ "resources": ["test1.png"], "matches": ["https://example.com/*"] }] }
```

## `externally_connectable`
Controls who may message the extension. Sub-fields: `ids` (extension IDs; `"*"` allows all), `matches` (patterns), `accepts_tls_channel_id`.

```json
{ "externally_connectable": { "matches": ["https://*.google.com/*"] } }
```

## `declarative_net_request`
Object with `rule_resources`, an array of `{ id, enabled, path }` pointing at static ruleset JSON. Requires `"declarativeNetRequest"` (warns) or `"declarativeNetRequestWithHostAccess"`. Up to 100 static rulesets, 50 enabled at once.

```json
{ "declarative_net_request": { "rule_resources": [{ "id": "ruleset_1", "enabled": true, "path": "rules_1.json" }] } }
```

## `commands`
Maps a command name to `{ suggested_key, description, global }`. `suggested_key` is a string or object keyed by `default`/`mac`/`windows`/`chromeos`/`linux`. At most four suggested shortcuts; each must include `Ctrl` or `Alt`; `Ctrl+Alt` is disallowed; on macOS `Ctrl` maps to `Command` while `MacCtrl` targets Control; `global` (default `false`) is limited to `Ctrl+Shift+0..9` off ChromeOS. Reserved `_execute_action` triggers the action.

## `omnibox`
Object with `keyword` (address-bar keyword). Provide a 16x16 icon.

## `options_page` / `options_ui`
`options_page` is a path opened full-page in a new tab. `options_ui` is `{ page, open_in_tab }`; with `open_in_tab: false` the options embed inside `chrome://extensions`.

## `side_panel`
Object with `default_path`. Requires the `"sidePanel"` permission; controlled at runtime via `chrome.sidePanel`.

## `devtools_page`
Path to an HTML page loaded in the DevTools context; enables the `chrome.devtools.*` APIs.

## `chrome_url_overrides`
Object whose key is one of `newtab`, `bookmarks`, or `history` (a path). Only one page may be overridden; the new-tab page cannot be overridden in incognito.

## `chrome_settings_overrides`
Overrides browser settings (Windows/Mac). Sub-fields include `homepage`, `search_provider` (`name`, `keyword`, `search_url`, `is_default` (required), `prepopulated_id`, `favicon_url`, `encoding`, etc.), and `startup_pages`.

## `sandbox`
Object with `pages` (paths) that run in an isolated origin with no extension-API access; they communicate via `postMessage`. Pair with `content_security_policy.sandbox`.

## `storage`
Object with `managed_schema` (path to a JSON Schema) for enterprise managed storage, read via `storage.managed`.

## `oauth2`
Object with `client_id` (ends in `apps.googleusercontent.com`) and `scopes`. Requires the `"identity"` permission; used with `chrome.identity.getAuthToken()`.

```json
{ "permissions": ["identity"], "oauth2": { "client_id": "yourID.apps.googleusercontent.com", "scopes": ["https://www.googleapis.com/auth/contacts.readonly"] } }
```
