# UI Surfaces (action, contextMenus, sidePanel, omnibox, notifications, commands)

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
The user-facing entry points an extension presents in Chrome: the toolbar icon, right-click menus, the side panel, the address-bar keyword, system notifications, and keyboard shortcuts. Methods are Promise-based in MV3.

## chrome.action
- **Purpose:** control the extension's toolbar icon, badge, tooltip, and popup.
- **Permission:** none — declaring the `action` manifest key is sufficient.
- **Key methods** (return a Promise unless noted): `setBadgeText({text?, tabId?})`, `getBadgeText(details)`, `setBadgeBackgroundColor({color, tabId?})`, `setBadgeTextColor({color, tabId?})`, `setIcon({path?|imageData?, tabId?})`, `setTitle({title, tabId?})`, `setPopup({popup, tabId?})`, `openPopup(options?)`, `enable(tabId?)`, `disable(tabId?)`, `getUserSettings()`.
- **Key events:** `onClicked` (`(tab)`); `onUserSettingsChanged` (Chrome 130+).
- **Key types:** `UserSettings` (`{isOnToolbar}`), `OpenPopupOptions` (`{windowId?}`).
- **Manifest key:** `action` (`default_icon`, `default_title`, `default_popup`).
- **MV3 notes:** MV3 merged the MV2 `browser_action` and `page_action` into a single `action`. `onClicked` does **not** fire when a popup is configured. Use `OffscreenCanvas` for dynamic icons inside a service worker.

```js
chrome.action.setBadgeBackgroundColor({ color: "#1a73e8" });
chrome.action.setBadgeText({ text: "3" });
chrome.action.onClicked.addListener((tab) => { /* only runs if no popup is set */ });
```

## chrome.contextMenus
- **Permission:** `"contextMenus"`.
- **Key methods:** `create(createProperties, callback?)` returns the item id; `update(id, updateProperties)`; `remove(menuItemId)`; `removeAll()` (Promises since Chrome 123).
- **Key events:** `onClicked` (`(info, tab?)` where `info` is `OnClickData`).
- **Key types:** `ContextType` (`all`, `page`, `frame`, `selection`, `link`, `editable`, `image`, `video`, `audio`, `launcher`, `action`, `tab`); `ItemType` (`normal`, `checkbox`, `radio`, `separator`); `OnClickData` (`menuItemId`, `pageUrl`, `linkUrl`, `selectionText`, `checked`, `wasChecked`, `editable`). Constant `ACTION_MENU_TOP_LEVEL_LIMIT` = `6`.
- **MV3 notes:** the `onclick` property in `createProperties` is unavailable in service workers — register the `onClicked` event instead. Create menus from inside `runtime.onInstalled`.

```js
chrome.runtime.onInstalled.addListener(() => {
  chrome.contextMenus.create({ id: "lookup", title: "Look up", contexts: ["selection"] });
});
chrome.contextMenus.onClicked.addListener((info, tab) => {
  if (info.menuItemId === "lookup") console.log(info.selectionText);
});
```

## chrome.sidePanel
- **Permission:** `"sidePanel"`. Chrome 114+.
- **Key methods:** `setOptions(options)`, `getOptions(options)`, `setPanelBehavior(behavior)`, `getPanelBehavior()`, `open(options)`.
- **Key types:** `PanelOptions` (`{enabled?, path?, tabId?}`), `PanelBehavior` (`{openPanelOnActionClick?}`), `OpenOptions` (`{tabId?, windowId?}`).
- **Manifest key:** `side_panel` (`default_path`).
- **MV3 notes:** `open()` may only be called in response to a user gesture.

```js
chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true });
chrome.sidePanel.setOptions({ tabId, path: "panel.html", enabled: true });
```

## chrome.omnibox
- **Permission:** none. **Manifest key:** `omnibox` (`keyword`).
- **Key methods:** `setDefaultSuggestion(suggestion)`.
- **Key events:** `onInputStarted`, `onInputChanged` (`(text, suggest)` — call `suggest([SuggestResult])`), `onInputEntered` (`(text, disposition)`), `onInputCancelled`, `onDeleteSuggestion`.
- **Key types:** `SuggestResult` (`{content, description, deletable?}`), `OnInputEnteredDisposition` = `"currentTab"`, `"newForegroundTab"`, `"newBackgroundTab"`. Suggestion `description` accepts styled markup with the tag names `url`, `match`, and `dim`.

```js
chrome.omnibox.onInputChanged.addListener((text, suggest) => {
  suggest([{ content: text, description: "Search for <match>" + text + "</match>" }]); // allow-placeholder: verbatim omnibox markup
});
chrome.omnibox.onInputEntered.addListener((text, disposition) => {
  if (disposition === "currentTab") chrome.tabs.update({ url: "https://example.com/?q=" + text });
});
```

## chrome.notifications
- **Permission:** `"notifications"`.
- **Key methods:** `create(notificationId?, options)` resolves to the id; `update(notificationId, options)`; `clear(notificationId)`; `getAll()`; `getPermissionLevel()`.
- **Key events:** `onClicked`, `onButtonClicked` (`(notificationId, buttonIndex)`), `onClosed` (`(notificationId, byUser)`), `onPermissionLevelChanged`.
- **Key types:** `TemplateType` = `"basic"`, `"image"`, `"list"`, `"progress"`; `PermissionLevel` = `"granted"`, `"denied"`; `NotificationOptions` (required `type`, `title`, `message`, `iconUrl`; optional `buttons` (max 2), `priority` (-2 to 2), `requireInteraction`, `silent`, `progress` (0 to 100)).

```js
chrome.notifications.create("done", {
  type: "basic", iconUrl: "icon.png", title: "Finished", message: "Task complete", priority: 2,
});
```

## chrome.commands
- **Permission:** none (manifest declaration only).
- **Key methods:** `getAll()` resolves to `Command[]`.
- **Key events:** `onCommand` (`(command, tab?)`) — does **not** fire for `_execute_action`.
- **Key types:** `Command` (`{name?, description?, shortcut?}`).
- **Manifest key:** `commands`, where each entry has `suggested_key` (`default`, `windows`, `mac`, `chromeos`, `linux`), `description`, optional `global`. Reserved `_execute_action` triggers the toolbar action.
- **MV3 notes:** `_execute_action` replaces MV2 `_execute_browser_action`/`_execute_page_action`. At most four suggested shortcuts; shortcuts must include `Ctrl` or `Alt`; `Ctrl+Alt` is prohibited; global commands are limited to `Ctrl+Shift+[0..9]`.

```js
// manifest.json: "commands": { "run-foo": { "suggested_key": { "default": "Ctrl+Shift+Y" }, "description": "Run foo" } }
chrome.commands.onCommand.addListener((command, tab) => {
  if (command === "run-foo") console.log("shortcut fired");
});
```
