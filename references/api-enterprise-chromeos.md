# Enterprise & ChromeOS

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
These APIs are ChromeOS-only. The `chrome.enterprise.*` namespaces additionally require the extension to be force-installed by enterprise policy (and most require the user to be affiliated). Promise call styles arrived in Chrome 96+; callbacks remain as migration context.

## chrome.enterprise.deviceAttributes
- **Permission:** `"enterprise.deviceAttributes"`. ChromeOS, policy-installed only. Returns an empty string when unaffiliated or unset.
- **Key methods:** `getDeviceAnnotatedLocation()`, `getDeviceAssetId()`, `getDeviceHostname()` (from the `DeviceHostnameTemplate` policy), `getDeviceSerialNumber()`, `getDirectoryDeviceId()` — each resolves to a string.

```js
const serial = await chrome.enterprise.deviceAttributes.getDeviceSerialNumber();
```

## chrome.enterprise.hardwarePlatform
- **Permission:** `"enterprise.hardwarePlatform"`. Requires the `EnterpriseHardwarePlatformAPIEnabled` policy.
- **Key methods:** `getHardwarePlatformInfo()` resolves to a `HardwarePlatformInfo` (`manufacturer`, `model`).

## chrome.enterprise.login
- **Permission:** `"enterprise.login"`. Usable only within ChromeOS Managed Guest sessions.
- **Key methods:** `exitCurrentManagedGuestSession()` (Chrome 139+) ends the active Managed Guest session.

## chrome.enterprise.networkingAttributes
- **Permission:** `"enterprise.networkingAttributes"`. If unaffiliated or offline, `runtime.lastError` is set.
- **Key methods:** `getNetworkDetails()` resolves to a `NetworkDetails` (`ipv4?`, `ipv6?`, `macAddress`).

```js
try {
  const d = await chrome.enterprise.networkingAttributes.getNetworkDetails();
} catch { console.warn(chrome.runtime.lastError?.message); }
```

## chrome.enterprise.platformKeys
- **Permission:** `"enterprise.platformKeys"`.
- **Key methods:** `getTokens()` resolves to `Token[]`; `getCertificates(tokenId)`; `importCertificate(tokenId, certificate)`; `removeCertificate(tokenId, certificate)`; `challengeKey(options)` (Chrome 110+). Deprecated since Chrome 110: `challengeMachineKey(challenge, registerKey?)`, `challengeUserKey(challenge, registerKey)`.
- **Key types/enums:** `Token` (`id`, `subtleCrypto`, `softwareBackedSubtleCrypto` (Chrome 97+)); `ChallengeKeyOptions` (`challenge`, `scope`, `registerKey?`); `Algorithm` (`"RSA"`, `"ECDSA"`); `Scope` (`"USER"`, `"MACHINE"`).
- **MV3 notes:** use `challengeKey` in new code.

## chrome.fileBrowserHandler
- **Permission:** `"fileBrowserHandler"`. Foreground only (register the listener from a page). Declared with the `"file_browser_handlers"` manifest key (per-entry `id`, `default_title`, `file_filters` such as `"filesystem:*.jpg"`).
- **Key events:** `onExecute` → `(id, details)` where `details` is a `FileHandlerExecuteEventDetails` (`entries`, `tab_id?`).

## chrome.fileSystemProvider
- **Permission:** `"fileSystemProvider"`. Declared with `"file_system_provider_capabilities"` (`configurable`, `multiple_mounts`, `watchable` defaults `false`; `source` is `"file"`/`"device"`/`"network"`).
- **Key methods:** `mount(options)`, `unmount(options)`, `get(fileSystemId)`, `getAll()`, `notify(options)` (Chrome 45+).
- **Key events:** request events firing in the worker, each with `successCallback`/`errorCallback`: `onMountRequested`, `onUnmountRequested`, `onGetMetadataRequested`, `onReadDirectoryRequested`, `onOpenFileRequested`, `onCloseFileRequested`, `onReadFileRequested`, `onCreateFileRequested`, `onCreateDirectoryRequested`, `onDeleteEntryRequested`, `onCopyEntryRequested`, `onMoveEntryRequested`, `onTruncateRequested`, `onWriteFileRequested`, `onAbortRequested`, `onConfigureRequested`, `onAddWatcherRequested`, `onRemoveWatcherRequested`, `onGetActionsRequested`, `onExecuteActionRequested`.
- **Key enums:** `ProviderError` (`"OK"`, `"FAILED"`, `"IN_USE"`, `"EXISTS"`, `"NOT_FOUND"`, `"ACCESS_DENIED"`, `"NO_SPACE"`, `"INVALID_OPERATION"`, `"ABORT"`, `"IO"`, and more); `OpenFileMode` (`"READ"`, `"WRITE"`); `ChangeType` (`"CHANGED"`, `"DELETED"`).

```js
chrome.fileSystemProvider.onReadDirectoryRequested.addListener((options, onSuccess, onError) => {
  onSuccess([{ isDirectory: false, name: "hello.txt", size: 5 }], false /* hasMore */);
});
await chrome.fileSystemProvider.mount({ fileSystemId: "demo-fs", displayName: "Demo Drive" });
```

## chrome.input.ime
- **Permission:** `"input"`. Declared with the `"input_components"` manifest key.
- **Key methods:** `clearComposition`, `commitText`, `deleteSurroundingText`, `hideInputView`, `keyEventHandled(requestId, response)`, `sendKeyEvents`, `setCandidates`, `setCandidateWindowProperties`, `setComposition`, `setCursorPosition`, `setMenuItems`, `updateMenuItems`, `setAssistiveWindowProperties` (Chrome 85+), `setAssistiveWindowButtonHighlighted` (Chrome 86+).
- **Key events:** `onActivate`, `onDeactivated`, `onFocus` → `(context)`, `onBlur` → `(contextID)`, `onKeyEvent` → `(engineID, keyData, requestId)` (return a boolean to consume, or call `keyEventHandled` later), `onCandidateClicked`, `onMenuItemActivated`, `onReset`, `onSurroundingTextChanged`, `onInputContextUpdate`.
- **Key enums:** `InputContextType` (`"text"`, `"search"`, `"url"`, `"email"`, `"number"`, `"password"`, `"tel"`, `"null"`); `KeyboardEventType` (`"keyup"`, `"keydown"`); `ScreenType` (`"normal"`, `"login"`, `"lock"`, `"secondary-login"`); `MouseButton` (`"left"`, `"middle"`, `"right"`).

```js
let ctx = 0;
chrome.input.ime.onFocus.addListener((context) => { ctx = context.contextID; });
chrome.input.ime.onKeyEvent.addListener((engineID, keyData, requestId) => {
  if (keyData.type === "keydown" && keyData.key === "a") {
    chrome.input.ime.commitText({ contextID: ctx, text: "あ" });
    return true;
  }
  return false;
});
```

## chrome.wallpaper
- **Permission:** `"wallpaper"`. **Method:** `setWallpaper(details)` where `details` has `url?` or `data?` (JPEG/PNG), `layout` (`WallpaperLayout`: `"STRETCH"`, `"CENTER"`, `"CENTER_CROPPED"`), `filename`, `thumbnail?` (generates a 128x60 thumbnail). Provide exactly one of `url`/`data`.

```js
await chrome.wallpaper.setWallpaper({ url: "https://example.com/bg.jpg", layout: "CENTER_CROPPED", filename: "bg" });
```

## chrome.vpnProvider
- **Permission:** `"vpnProvider"`.
- **Key methods:** `createConfig(name)` resolves to a config `id`; `destroyConfig(id)`; `notifyConnectionStateChanged(state)`; `sendPacket(data)`; `setParameters(parameters)`.
- **Key events:** `onConfigCreated`, `onConfigRemoved`, `onPacketReceived` → `(data)`, `onPlatformMessage` → `(id, message, error)`, `onUIEvent`.
- **Key enums:** `VpnConnectionState` (`"connected"`, `"failure"`); `PlatformMessage` (`"connected"`, `"disconnected"`, `"error"`, `"linkDown"`, `"linkUp"`, `"linkChanged"`, `"suspend"`, `"resume"`); `UIEvent` (`"showAddDialog"`, `"showConfigureDialog"`).

```js
chrome.vpnProvider.onPlatformMessage.addListener(async (id, message) => {
  if (message === "connected") {
    await chrome.vpnProvider.setParameters({ address: "10.10.10.2/24", dnsServers: ["10.10.10.1"], mtu: "1500", exclusionList: [], inclusionList: ["0.0.0.0/0"] });
    chrome.vpnProvider.notifyConnectionStateChanged("connected");
  }
});
```
