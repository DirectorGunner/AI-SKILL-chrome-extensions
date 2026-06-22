# Media Capture & Speech

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Capture tab/screen/window media streams, save pages as MHTML, manage ChromeOS audio devices, synthesize speech, and implement a speech engine: `chrome.tabCapture`, `chrome.desktopCapture`, `chrome.pageCapture`, `chrome.audio`, `chrome.tts`, `chrome.ttsEngine`.

## chrome.tabCapture
- **Purpose:** capture the visible area of a tab as a media stream (audio and/or video).
- **Permission:** `"tabCapture"`.
- **Key methods:** `capture(options, callback)` → `(stream)` — foreground only (not the service worker), requires a prior user invocation of the extension (like `activeTab`); `getMediaStreamId(options)` resolves to a string id (Chrome 71+, service-worker friendly); `getCapturedTabs()` resolves to `CaptureInfo[]` (Chrome 116+).
- **Key events:** `onStatusChanged` → `(info)`.
- **Key types:** `CaptureOptions` (`audio?`, `audioConstraints?`, `video?`, `videoConstraints?`); `GetMediaStreamOptions` (`consumerTabId?`, `targetTabId?`); `CaptureInfo` (`tabId`, `status`, `fullscreen`); `TabCaptureState` (`"pending"`, `"active"`, `"stopped"`, `"error"`).
- **MV3 notes:** the typical MV3 pattern is `getMediaStreamId()` in the worker, then consume the id with `navigator.mediaDevices.getUserMedia()` in an offscreen document.

```js
// In a service worker (after a user gesture invokes the extension):
const streamId = await chrome.tabCapture.getMediaStreamId({ targetTabId });
// Pass streamId to an offscreen document, which calls getUserMedia with
// { mandatory: { chromeMediaSource: 'tab', chromeMediaSourceId: streamId } }.
```

## chrome.desktopCapture
- **Purpose:** prompt the user to choose a screen, window, or tab, returning a stream id for `getUserMedia()`.
- **Permission:** `"desktopCapture"`.
- **Key methods:** `chooseDesktopMedia(sources, targetTab?, callback)` returns a number request id; the callback receives `(streamId, options)` where `options.canRequestAudioTrack` is a boolean; `cancelChooseDesktopMedia(desktopMediaRequestId)`.
- **Key types/enums:** `DesktopCaptureSourceType` (`"screen"`, `"window"`, `"tab"`, `"audio"`); `SystemAudioPreferenceEnum` (Chrome 105+: `"include"`, `"exclude"`); `SelfCapturePreferenceEnum` (Chrome 107+).
- **MV3 notes:** consume the returned id by passing it to `getUserMedia()` with `chromeMediaSource: 'desktop'` (e.g. from an offscreen document).

```js
chrome.desktopCapture.chooseDesktopMedia(["screen", "window", "tab", "audio"], (streamId, options) => {
  if (!streamId) return; // user cancelled
  console.log("canRequestAudioTrack:", options.canRequestAudioTrack);
});
```

## chrome.pageCapture
- **Purpose:** save a tab's content as MHTML (RFC 2557).
- **Permission:** `"pageCapture"`.
- **Key methods:** `saveAsMHTML(details)` where `details` is `{ tabId }`, resolves to a `Blob` (Chrome 116+).
- **MV3 notes:** MHTML files can only load from the file system in the main frame.

```js
const mhtmlBlob = await chrome.pageCapture.saveAsMHTML({ tabId });
```

## chrome.audio
- **Purpose:** query and control audio devices and mute/level state. ChromeOS only (Chrome 59+).
- **Permission:** `"audio"`.
- **Key methods:** `getDevices(filter)` resolves to `AudioDeviceInfo[]`; `getMute(streamType)` resolves to a boolean; `setMute(streamType, isMuted)`; `setActiveDevices(ids)`; `setProperties(id, properties)`.
- **Key events:** `onDeviceListChanged`; `onLevelChanged` → `(LevelChangedEvent)`; `onMuteChanged` → `(MuteChangedEvent)`.
- **Key types/enums:** `StreamType` (`"INPUT"`, `"OUTPUT"`); `DeviceType` (`"HEADPHONE"`, `"MIC"`, `"USB"`, `"BLUETOOTH"`, `"HDMI"`, `"INTERNAL_SPEAKER"`, `"INTERNAL_MIC"`, `"FRONT_MIC"`, `"REAR_MIC"`, `"KEYBOARD_MIC"`, `"HOTWORD"`, `"LINEOUT"`, `"POST_MIX_LOOPBACK"`, `"POST_DSP_LOOPBACK"`, `"ALSA_LOOPBACK"`, `"OTHER"`); `AudioDeviceInfo` (`id`, `deviceName`, `deviceType`, `displayName`, `isActive`, `level`, `streamType`).

```js
const devices = await chrome.audio.getDevices({ streamTypes: ["INPUT"] });
```

## chrome.tts
- **Purpose:** play synthesized text-to-speech.
- **Permission:** `"tts"`.
- **Key methods:** `speak(utterance, options?)` (resolves immediately; use `options.onEvent` for progress); `stop()`; `pause()`; `resume()`; `isSpeaking()` resolves to a boolean; `getVoices()` resolves to `TtsVoice[]`.
- **Key events:** `onVoicesChanged` (Chrome 124+).
- **Key types:** `TtsOptions` (Chrome 77+: `rate` (0.1–10.0, default 1.0), `pitch` (0–2, default 1.0), `volume` (0–1, default 1.0), `lang`, `voiceName`, `enqueue`, `requiredEventTypes`, `desiredEventTypes`, `onEvent`); `EventType` (`"start"`, `"end"`, `"word"`, `"sentence"`, `"marker"`, `"interrupted"`, `"cancelled"`, `"error"`, `"pause"`, `"resume"`); `VoiceGender` (deprecated since Chrome 70).
- **MV3 notes:** callable from the service worker. `gender`/`VoiceGender` are deprecated.

```js
chrome.tts.speak("Hello, world.", { lang: "en-US", rate: 2.0 });
```

## chrome.ttsEngine
- **Purpose:** implement a TTS engine other extensions/Chrome can use.
- **Permission:** `"ttsEngine"`. Register voices via the `tts_engine` manifest key (`voices` entries with `voice_name`, `lang`, `event_types`).
- **Key methods:** `updateVoices(voices)` (Chrome 66+); `updateLanguage(status)` (Chrome 132+).
- **Key events:** `onSpeak` → `(utterance, options, sendTtsEvent)`; `onSpeakWithAudioStream` (Chrome 92+) → `(utterance, options, audioStreamOptions, sendTtsAudio, sendError)`; `onStop`; `onPause`; `onResume`; `onInstallLanguageRequest` (Chrome 131+); `onLanguageStatusRequest` (Chrome 132+); `onUninstallLanguageRequest` (Chrome 132+).
- **Key types:** `SpeakOptions` (`gender`, `lang`, `pitch`, `rate`, `voiceName`, `volume`); `AudioStreamOptions` (Chrome 92+: `bufferSize`, `sampleRate`).
- **MV3 notes:** register the background entry as `background.service_worker` in MV3 (older samples show MV2 `background.page` — migration context). The `tts_engine` key structure is unchanged.

```js
chrome.ttsEngine.onSpeak.addListener((utterance, options, sendTtsEvent) => {
  sendTtsEvent({ type: "start", charIndex: 0 });
  // synthesize utterance honoring options.rate/pitch/volume
  sendTtsEvent({ type: "end", charIndex: utterance.length });
});
chrome.ttsEngine.onStop.addListener(() => {});
```
