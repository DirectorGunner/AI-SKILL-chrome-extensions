# Manifest Specialized Keys

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Niche `manifest.json` keys for narrow cases: ChromeOS integration, text-to-speech engines, cross-origin isolation, shared code, origin trials, hardware requirements, and DevTools panels. Most are rarely needed; several are ChromeOS-only.

## `file_browser_handlers`
Register handlers the ChromeOS Files app can invoke on selected files (via the `fileBrowserHandler` API). Array of objects with `id`, `default_title`, and `file_filters` (patterns prefixed `filesystem:`). Requires the `fileBrowserHandler` permission. ChromeOS only.

```json
"file_browser_handlers": [{ "id": "upload", "default_title": "Save to Gallery", "file_filters": ["filesystem:*.jpg", "filesystem:*.png"] }]
```

## `file_handlers`
Declare which file types a ChromeOS extension can open (Chrome 120+). Array of objects: `action` (HTML path), `name`, `accept` (MIME type to extension array), `launch_type` (`"single-client"` or `"multiple-clients"`). ChromeOS only.

```json
"file_handlers": [{ "action": "/open_text.html", "name": "Plain text", "accept": { "text/plain": [".txt"] } }]
```

## `file_system_provider_capabilities`
Describe a virtual file system provided via the `fileSystemProvider` API. Object with `configurable`, `watchable`, `multiple_mounts` (booleans), and `source` (`"file"`/`"device"`/`"network"`, required). Requires the `fileSystemProvider` permission. ChromeOS only.

```json
"file_system_provider_capabilities": { "configurable": true, "watchable": false, "multiple_mounts": true, "source": "network" }
```

## `input_components`
Register an input method editor (IME) using the `input.ime` API. Array of objects: `name` (required), plus `id`, `language`, `layouts`, `input_view`, `options_page`. Requires the `input` permission. ChromeOS only.

```json
"input_components": [{ "name": "ToUpperIME", "id": "ToUpperIME", "language": "en", "layouts": ["us::eng"] }]
```

## `tts_engine`
Declare the voices a TTS engine extension provides (paired with the `ttsEngine` API/permission). Object with a `voices` array; each voice has `voice_name` (required), `lang`, and `event_types` (e.g. `"start"`, `"end"`, `"marker"`).

```json
"tts_engine": { "voices": [{ "voice_name": "Alice", "lang": "en-US", "event_types": ["start", "marker", "end"] }] }
```

## `cross_origin_embedder_policy`
Sets the `Cross-Origin-Embedder-Policy` (COEP) response header for the extension's origin (Chrome 93+). Object with a `value` string (e.g. `"require-corp"`).

```json
"cross_origin_embedder_policy": { "value": "require-corp" }
```

## `cross_origin_opener_policy`
Sets the `Cross-Origin-Opener-Policy` (COOP) response header. Object with a `value` string (e.g. `"same-origin"`). Combined with COEP, opts into cross-origin isolation.

```json
"cross_origin_opener_policy": { "value": "same-origin" }
```

## `shared_modules`
Two fields enabling resource/API sharing between extensions. `export` is an object with an `allowlist` array of extension IDs permitted to import. `import` is an array of objects, each `{ "id": "EXTENSION_ID", "minimum_version": "0.5" }` (the `minimum_version` is optional).

```json
"export": { "allowlist": ["aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"] },
"import": [{ "id": "cccccccccccccccccccccccccccccccc" }, { "id": "dddddddddddddddddddddddddddddddd", "minimum_version": "0.5" }]
```

## `trial_tokens`
Enroll the extension in Origin Trial / Deprecation Trial features. An array of opaque token strings (one per trial feature).

```json
"trial_tokens": ["AnlT7gRo...AAAG97Im9yaWdpbiI6..."]
```

## `requirements`
Declare technologies the extension needs so a host can warn unsupported users. Object keyed by requirement; `"3D"` takes a `features` array of `"webgl"` and/or `"css3d"`. (The former `"plugins"` requirement was deprecated in Chrome 45.)

```json
"requirements": { "3D": { "features": ["webgl"] } }
```

## `event_rules` (migration context)
Declaratively register `{ event, conditions, actions }` rules. The reference page is legacy and references `declarativeWebRequest`, which is not available in MV3; the supported MV3 path is `declarativeContent` (register its rules via the `chrome.declarativeContent` API at runtime).

## `devtools_page`
Path to an HTML page that runs in the DevTools context; declaring it enables the `chrome.devtools.*` APIs.

```json
"devtools_page": "devtools.html"
```
