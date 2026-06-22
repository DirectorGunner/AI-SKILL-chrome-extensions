# Internationalization (i18n)

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
`chrome.i18n` localizes extension UI and metadata using `_locales/` message catalogs and the `default_locale` manifest key, plus runtime helpers for message lookup, UI language, accepted languages, and language detection.

## chrome.i18n
- **Purpose:** retrieve localized strings, the browser UI and accepted languages, and detect the language of supplied text.
- **Permission:** none.
- **Key methods:**
  - `getMessage(messageName, substitutions?, options?)` returns a localized string synchronously (empty string if the message is missing). `messageName` is a key from `messages.json`; `substitutions` allows up to 9 replacement values; `options` accepts `escapeLt` (boolean) to escape less-than characters in the result.
  - `getUILanguage()` returns the browser UI language synchronously (e.g. `"en-US"`).
  - `getAcceptLanguages()` resolves to `LanguageCode[]`.
  - `detectLanguage(text)` resolves to a detection result (`isReliable`, `languages` — array of `{ language, percentage }`); uses CLD (Compact Language Detector).
- **Key types:** `LanguageCode` — an ISO language code string (`"en"`, `"fr"`); unknown is `"und"`.
- **Manifest & catalog:**
  - `default_locale` — required when the extension includes a `/_locales` directory (e.g. `"default_locale": "en"`).
  - `_locales/` layout: one subdirectory per locale, each with a `messages.json` (e.g. `_locales/en/messages.json`).
  - `messages.json` entry: keyed by message name with `message` (string), `description` (optional), and `placeholders` (optional; each has `content` and optional `example`).
- **Predefined messages:** `@@extension_id`, `@@ui_locale`, `@@bidi_dir` (`"ltr"`/`"rtl"`), `@@bidi_reversed_dir`, `@@bidi_start_edge` (`"left"`/`"right"`), `@@bidi_end_edge`. Reference predefined/named messages in CSS and the manifest with the `__MSG_name__` form (e.g. `__MSG_@@extension_id__`); in JavaScript use `getMessage()`.
- **MV3 notes:** `getMessage` and `getUILanguage` are synchronous; `getAcceptLanguages` and `detectLanguage` return Promises. No permission needed.

```js
// _locales/en/messages.json:
// { "greeting": { "message": "Hello, $USER$",
//     "placeholders": { "user": { "content": "$1", "example": "Ada" } } } }
const text = chrome.i18n.getMessage("greeting", ["Ada"]);
const ui = chrome.i18n.getUILanguage();
const accepted = await chrome.i18n.getAcceptLanguages();
const detection = await chrome.i18n.detectLanguage("Bonjour le monde");
```
