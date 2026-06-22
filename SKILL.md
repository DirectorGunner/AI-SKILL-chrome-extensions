---
name: chrome-extensions
description: >-
  Use when building, scaffolding, or debugging Chrome, Chromium, or Edge Manifest V3 browser
  extensions: service workers, content scripts, popups, options pages, side panels, omnibox,
  context menus, message passing, permissions, host_permissions, native messaging, loading unpacked
  extensions, MV2-to-MV3 migration, manifest.json with manifest_version 3, or chrome.* / browser.*
  APIs such as action, tabs, scripting, storage, runtime, alarms, permissions, declarativeNetRequest,
  webRequest, identity, sidePanel, commands, cookies, notifications, and devtools. Do not use for
  Firefox/Safari-specific WebExtension behavior, ordinary web-page automation, or Chrome Web Store
  listing/review/publishing beyond manifest keys.
covers:
  - permissions
  - optional-permissions
  - service worker
  - alarms
  - oninstalled
  - onmessage
  - offscreen
  - scripting
  - executescript
  - main
  - sendmessage
  - activetab
---

# Chrome Extensions (Manifest V3) API Reference

Faithful, task-routed reference for building and debugging **Manifest V3** Chrome/Chromium
extensions — the `chrome.*` APIs, the manifest format, and the core concepts. Manifest V3 only;
Manifest V2 appears only as labeled migration context.

## When to use this

Use this skill whenever the task involves a browser extension and any of: a `manifest.json` with
`"manifest_version": 3`; a background **service worker**, **content scripts**, popup, options page,
**side panel**, **omnibox**, or **context menus**; any `chrome.*`/`browser.*` extension API; message
passing or native messaging; declaring `permissions`/`host_permissions`; an MV2→MV3 migration; or
loading an unpacked extension. It fires even when "Chrome extension" is never said.

**Do not use when** the task is: Firefox/Safari WebExtension specifics or the `webextension-polyfill`;
general web or Node.js development unrelated to extensions; npm package authoring; Puppeteer/Playwright
automation of normal web pages (driving an *extension* in tests is in scope — see the test reference);
Chrome DevTools usage as an end user; or the Web Store listing/review/publishing workflow beyond
manifest keys. Those belong to another skill or a generic answer.

## Task router

Load only the reference the task needs.

| If the task is about... | Read |
| --- | --- |
| chrome.runtime, events, alarms, idle, power, onInstalled/onStartup | references/api-runtime-lifecycle.md |
| offscreen documents, gcm/instanceID push, chrome.extension, types/extensionTypes/dom | references/api-runtime-misc.md |
| injecting scripts/CSS, chrome.scripting.executeScript, dynamic content scripts, userScripts | references/api-scripting-userscripts.md |
| tabs, tab groups, windows (query/create/update/sendMessage) | references/api-tabs-windows.md |
| webNavigation, sessions, history, bookmarks, readingList, topSites, processes, search | references/api-navigation-history.md |
| blocking/redirecting requests, declarativeNetRequest, webRequest, proxy, cookies, dns | references/api-network-filtering.md |
| chrome.storage areas/quotas, contentSettings, privacy, fontSettings, browsingData | references/api-storage-settings.md |
| toolbar action/badge, contextMenus, sidePanel, omnibox, notifications, commands shortcuts | references/api-ui-surfaces.md |
| DevTools panels/inspectedWindow/network, chrome.debugger (CDP) | references/api-devtools-debugger.md |
| OAuth tokens, chrome.identity, loginState, certificate/platform keys, WebAuthn proxy | references/api-identity-auth.md |
| capturing tab/screen media, MHTML, audio devices, text-to-speech | references/api-media-capture.md |
| downloads, managing installed extensions, runtime permission requests | references/api-downloads-management-permissions.md |
| system info (cpu/display/memory/storage), printing, scanning, accessibility features | references/api-system-hardware.md |
| enterprise/ChromeOS APIs, file system provider, IME, wallpaper, VPN | references/api-enterprise-chromeos.md |
| localizing strings, getMessage, default_locale, _locales | references/api-i18n.md |
| how the components fit together / getting started | references/concepts-architecture.md |
| service worker lifecycle, why state is lost, alarms vs setTimeout, offscreen | references/concepts-service-workers.md |
| content-script isolated vs MAIN world, run_at, match patterns | references/concepts-content-scripts.md |
| sendMessage/onMessage, ports, external messaging, native messaging | references/concepts-messaging.md |
| which permission/host_permission to declare, activeTab, match-pattern syntax | references/concepts-permissions.md |
| choosing a storage area, persisting state, cookies, cross-origin isolation | references/concepts-storage-state.md |
| install/update events, update rollout, setUninstallURL | references/concepts-lifecycle-updates.md |
| CSP, no remotely hosted code, web_accessible_resources, privacy | references/concepts-security-privacy.md |
| converting an MV2 extension to MV3 | references/concepts-mv2-to-mv3-migration.md |
| manifest_version/name/version/icons/default_locale and other identity keys | references/manifest-format-core-keys.md |
| action/background/content_scripts/permissions/CSP/side_panel/oauth2 keys | references/manifest-capability-keys.md |
| file_handlers/input_components/tts_engine/COEP-COOP/shared_modules/trial_tokens/devtools_page | references/manifest-specialized-keys.md |
| confirming a permission string exists and is spelled correctly | references/permissions-list.md |
| accessibility, favicons, localization formats, notifications UI, extending DevTools | references/howto-ui-and-devtools.md |
| geolocation, screen capture, WebSockets, WebHID, WebUSB, origin trials | references/howto-web-platform.md |
| OAuth setup, Google Analytics 4, packaging/self-hosting and loading unpacked (not Web Store review/listing) | references/howto-integrate-distribute.md |
| debugging the worker/popup/content scripts, Jest, Puppeteer, Selenium | references/howto-test-debug.md |

Start from references/INDEX.md if the right file is not obvious.

## Runtime workflow

1. Load this SKILL.md and use the task-router to pick the reference for the API, concept, or
   manifest key in play; read only the files you need.
2. Confirm the required `permissions` and `host_permissions` for every API used (cross-check against
   references/permissions-list.md), and that the manifest is MV3-correct (`"manifest_version": 3`).
3. Apply MV3-correct patterns; if a sample uses an MV2 pattern, flag it and convert it
   (references/concepts-mv2-to-mv3-migration.md).
4. Report any detail you cannot verify against a reference rather than guessing; never invent an API
   name, signature, permission string, or manifest key.
5. Before claiming the skill covers an API, confirm it appears in references/topics.json or a
   reference file (the verifier enforces the full 83-API coverage list).

## Gotchas

Recurring failure modes and what to do instead live in the sibling [GOTCHA.md](GOTCHA.md).

## References

All depth lives in `references/` (start at references/INDEX.md; metadata in references/topics.json).
Put routing here in SKILL.md; open a reference only when the router points to it. The references are
organized by topic, not one file per source.

## Official documentation

Includes original content written for this skill. The official documentation is at https://developer.chrome.com/docs/extensions — consult it to verify anything uncertain, conflicting, or version-specific.

## Verification notes

Ground-truth checks for this package (run from the repository root):

```powershell
python work/verify_chrome_extensions.py .agents/skills/chrome-extensions
python "$env:DEVROOT\SKILLS\skills\skill-drafting\scripts\validate_skill_package.py" "$env:DEVROOT\SKILLS\skills\chrome-extensions"
```

`work/verify_chrome_extensions.py` asserts: every reference is listed in references/INDEX.md and has
exactly one references/topics.json entry; every reference has an `## Overview` and a `## See also`
whose `references/*.md` links resolve; a curated preserve-glossary of verbatim terms appears in the
references; and all 83 `chrome.*` APIs are covered. `validate_skill_package.py` checks the package
structure and that no unresolved placeholder text remains. When extending the skill, re-run both
and confirm every API name, signature, permission string, and manifest key still matches the
official docs exactly (MV3-correct; MV2 only as labeled migration context).
