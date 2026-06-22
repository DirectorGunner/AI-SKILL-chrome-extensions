# Chrome Extensions (MV3) reference index

Start here, then open only the file the SKILL.md task-router points to. All references are Manifest V3, original prose with identifiers preserved verbatim.

## chrome.* API reference (all 83 APIs)

- [api-runtime-lifecycle.md](api-runtime-lifecycle.md) — runtime, events, alarms, idle, power.
- [api-runtime-misc.md](api-runtime-misc.md) — offscreen, gcm, instanceID, extension, types, extensionTypes, dom.
- [api-scripting-userscripts.md](api-scripting-userscripts.md) — scripting, userScripts (inject code/CSS, dynamic content scripts).
- [api-tabs-windows.md](api-tabs-windows.md) — tabs, tabGroups, windows.
- [api-navigation-history.md](api-navigation-history.md) — webNavigation, sessions, history, topSites, bookmarks, readingList, processes, search.
- [api-network-filtering.md](api-network-filtering.md) — declarativeNetRequest, webRequest, declarativeContent, proxy, cookies, dns.
- [api-storage-settings.md](api-storage-settings.md) — storage, contentSettings, privacy, fontSettings, browsingData.
- [api-ui-surfaces.md](api-ui-surfaces.md) — action, contextMenus, sidePanel, omnibox, notifications, commands.
- [api-devtools-debugger.md](api-devtools-debugger.md) — devtools.inspectedWindow/network/panels/performance/recorder, debugger.
- [api-identity-auth.md](api-identity-auth.md) — identity, loginState, certificateProvider, platformKeys, webAuthenticationProxy.
- [api-media-capture.md](api-media-capture.md) — tabCapture, desktopCapture, pageCapture, audio, tts, ttsEngine.
- [api-downloads-management-permissions.md](api-downloads-management-permissions.md) — downloads, management, permissions.
- [api-system-hardware.md](api-system-hardware.md) — system.cpu/display/memory/storage, systemLog, printing, printerProvider, printingMetrics, documentScan, accessibilityFeatures.
- [api-enterprise-chromeos.md](api-enterprise-chromeos.md) — enterprise.*, fileBrowserHandler, fileSystemProvider, input.ime, wallpaper, vpnProvider.
- [api-i18n.md](api-i18n.md) — i18n localization.

## Core concepts

- [concepts-architecture.md](concepts-architecture.md) — components and how they fit together; start here for "how an extension works".
- [concepts-service-workers.md](concepts-service-workers.md) — the MV3 background service worker lifecycle and constraints.
- [concepts-content-scripts.md](concepts-content-scripts.md) — isolated/MAIN world, injection, run_at, match patterns.
- [concepts-messaging.md](concepts-messaging.md) — one-time messages, ports, external and native messaging.
- [concepts-permissions.md](concepts-permissions.md) — permissions vs host_permissions, optional permissions, activeTab, match patterns.
- [concepts-storage-state.md](concepts-storage-state.md) — storage areas, worker state, cookies, cross-origin isolation.
- [concepts-lifecycle-updates.md](concepts-lifecycle-updates.md) — install/update events and update rollout.
- [concepts-security-privacy.md](concepts-security-privacy.md) — no remote code, CSP, web_accessible_resources, privacy.
- [concepts-mv2-to-mv3-migration.md](concepts-mv2-to-mv3-migration.md) — OLD-to-NEW migration checklist.

## Manifest reference

- [manifest-format-core-keys.md](manifest-format-core-keys.md) — file format and identity keys.
- [manifest-capability-keys.md](manifest-capability-keys.md) — action, background, content_scripts, permissions, CSP, and other functional keys.
- [manifest-specialized-keys.md](manifest-specialized-keys.md) — niche/ChromeOS/TTS/origin-trial/devtools keys.
- [permissions-list.md](permissions-list.md) — the named permission strings.

## How-to guides

- [howto-ui-and-devtools.md](howto-ui-and-devtools.md) — a11y, favicons, localization, notifications, extending DevTools.
- [howto-web-platform.md](howto-web-platform.md) — geolocation, screen capture, WebSockets, WebHID, WebUSB, origin trials.
- [howto-integrate-distribute.md](howto-integrate-distribute.md) — OAuth, Google Analytics 4, distribution, load unpacked.
- [howto-test-debug.md](howto-test-debug.md) — inspecting the worker/popup/content scripts, Jest, Puppeteer, Selenium.
