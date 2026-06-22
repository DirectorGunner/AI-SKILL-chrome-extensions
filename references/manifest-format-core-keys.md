# Manifest Format & Core Keys

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
`manifest.json` is a JSON document (UTF-8) at the extension's root declaring metadata and capabilities. Comments are not permitted in a production manifest, and `manifest_version` must be `3`.

## `manifest_version`
Required integer; only `3` is accepted (the Chrome Web Store no longer accepts MV2).

```json
{ "manifest_version": 3 }
```

## `name`
Required string identifying the extension in the install dialog, `chrome://extensions`, and the Web Store. Up to 75 characters; localizable.

## `short_name`
Optional abbreviated name (up to 12 characters) used where space is limited; falls back to a truncated `name`.

## `version`
Required string: one to four dot-separated integers (e.g. `"1.0"`, `"3.1.2.4567"`). Each integer is `0`–`65535`; a non-zero integer must not begin with `0` (so `032` is invalid); the value must not be entirely zero (`0` and `0.0.0.0` invalid, `0.1.0.0` valid). Compared left to right.

## `version_name`
Optional human-readable label shown instead of `version` (e.g. `"1.0 beta"`).

## `description`
Optional plain-text summary, up to 132 characters, no HTML; localizable.

## `default_locale`
String naming a subdirectory under `_locales` (e.g. `"en"`, `"pt_BR"`). Required when a `_locales` directory exists, and must be absent otherwise.

## `icons`
Object mapping pixel-size strings to file paths. Provide at least `128` (install + Web Store); `48` shows on `chrome://extensions`; `16` is the page favicon. PNG recommended.

```json
{ "icons": { "16": "icon16.png", "48": "icon48.png", "128": "icon128.png" } }
```

## `minimum_chrome_version`
String matching part of a Chrome version (e.g. `"126"`). Older browsers show "Not compatible" instead of the install button.

## `key`
A single-line base64-encoded public key that pins the extension's ID during local development (from the Developer Dashboard Package tab, "View public key"). A stable ID matters for origin-restricted servers, cross-extension/site messaging, and `web_accessible_resources`.

## `homepage_url`
URL for the extension's home page; defaults to the Web Store listing.

## `update_url`
HTTPS URL of an XML update manifest (Omaha format) for self-hosted extensions; required when hosting outside the Web Store.

## `incognito`
Enum `"spanning"` (default), `"split"`, or `"not_allowed"`. `"spanning"` shares one process and flags incognito events; `"split"` uses a separate incognito process with memory-only cookies; `"not_allowed"` disables the extension in incognito (Chrome 47+). `storage.sync`/`storage.local` stay shared regardless.

```json
{ "incognito": "spanning" }
```
