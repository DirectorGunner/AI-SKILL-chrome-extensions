# UI & DevTools How-Tos

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Recipes for accessible extension UI, page favicons, localizing UI strings, notifications, and extending Chrome DevTools.

## Accessibility (a11y)
Prefer standard HTML controls (`button`, `checkbox`, radio, text input, `select`, link) — they are keyboard-focusable, scale with zoom, and work with screen readers. For custom widgets, add WAI-ARIA attributes: `role` (e.g. `role="toolbar"`, `role="button"`), `aria-activedescendant` to point at the focused child, `tabindex="0"` to make an element focusable in document order, and `tabindex="-1"` for programmatic-only focus. For example, a toolbar container carries `role="toolbar"`, `tabindex="0"`, and `aria-activedescendant`, holding several `img` children each with `role="button"`, a descriptive `alt` (e.g. `cut`/`copy`/`paste`), and an `id`; arrow-key handlers move `aria-activedescendant` between the child ids. Keep a visible focus outline, survive 200% zoom, keep text out of images (use web fonts), check color contrast, and give images descriptive `alt` text (`alt=""` for decorative ones).

## Favicons
Request the `"favicon"` permission, then read a page's favicon from the reserved `_favicon/` path. Build the URL with `runtime.getURL("/_favicon/")` and set the `pageUrl` and `size` search params; the resolved form is `chrome-extension://EXTENSION_ID/_favicon/?pageUrl=EXAMPLE_URL&size=FAV_SIZE`. From a content script, also expose `"_favicon/*"` under `web_accessible_resources`.

```js
function faviconURL(u) {
  const url = new URL(chrome.runtime.getURL("/_favicon/"));
  url.searchParams.set("pageUrl", u);
  url.searchParams.set("size", "32");
  return url.toString();
}
```

## Localizing the UI (message formats)
Declare a `default_locale` and place per-locale `messages.json` under `_locales/` (e.g. `_locales/en/messages.json`); retrieve strings with `chrome.i18n` `getMessage()`. Each entry needs a name and a `message`; `description`, `placeholders`, and `example` are optional. Message names allow `A-Z`, `a-z`, `0-9`, underscore, and `@`, are case-insensitive, and must not begin with `@@` (reserved). Placeholders use `$placeholder_name$`; a literal dollar sign is `$$`; a placeholder's `content` can reference substitution args as `$1`, `$2`.

```json
{ "hello": { "message": "Hello, $USER$", "placeholders": { "user": { "content": "$1", "example": "Cira" } } } }
```

See [api-i18n.md](api-i18n.md) for the full `chrome.i18n` surface.

## Notifications
Request the `"notifications"` permission and create with `chrome.notifications.create(id, options)`. `options` is a `notifications.NotificationOptions` whose `type` (`notifications.TemplateType`) is `basic`, `image`, `list`, or `progress`. Respond with events like `notifications.onButtonClicked`, registered in the service worker.

```js
chrome.notifications.create("notify-id", {
  type: "progress", title: "Primary Title", message: "Primary message", iconUrl: "icon.png", progress: 42,
});
chrome.notifications.onButtonClicked.addListener(replyBtnClick);
```

## Extending DevTools
Add the `devtools_page` manifest key pointing at a local HTML file; that page loads when DevTools opens and is the only context with `chrome.devtools.inspectedWindow`, `chrome.devtools.network`, `chrome.devtools.panels`, `chrome.devtools.recorder`, and `chrome.devtools.performance`. Use `chrome.devtools.panels.create` to add a panel and `panels.elements.createSidebarPane` (then `setPage`/`setObject`/`setExpression`) for an Elements sidebar. Run code in the inspected page with `chrome.devtools.inspectedWindow.eval`, or inject via `chrome.scripting.executeScript` targeting `chrome.devtools.inspectedWindow.tabId`.

```js
chrome.devtools.panels.create("My Panel", "MyPanelIcon.png", "Panel.html", (panel) => {
  // code invoked on panel creation
});
```
