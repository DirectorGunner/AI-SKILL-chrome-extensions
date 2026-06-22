# System & Hardware

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
Query host hardware (CPU, display, memory, storage), write system logs, print, scan documents, and toggle accessibility features. Several are ChromeOS-only; some require enterprise policy. `system.*` permission strings are namespaced (e.g. `"system.cpu"`).

## chrome.system.cpu
- **Permission:** `"system.cpu"`. **Method:** `getInfo()` resolves to a `CpuInfo` (`archName`, `modelName`, `numOfProcessors`, `processors` (array of `ProcessorInfo` with `usage` `CpuTime`: `user`, `kernel`, `idle`, `total`), `features`, `temperatures` (Chrome 60+, Celsius, ChromeOS)). Values in `CpuTime` are cumulative — sample twice and diff.

```js
const info = await chrome.system.cpu.getInfo();
console.log(info.archName, info.numOfProcessors);
```

## chrome.system.display
- **Permission:** `"system.display"` (configuration methods are ChromeOS only).
- **Key methods:** `getInfo(flags?)` resolves to `DisplayUnitInfo[]`; `getDisplayLayout()` (ChromeOS); `setDisplayProperties(id, info)` (ChromeOS); `setDisplayLayout(layouts)` (ChromeOS); `enableUnifiedDesktop(enabled)`; `overscanCalibrationStart/Adjust/Reset/Complete(id, ...)`; `setMirrorMode(info)` (ChromeOS); `showNativeTouchCalibration(id)` resolves to a boolean.
- **Key events:** `onDisplayChanged`.
- **Key types/enums:** `DisplayUnitInfo` (`id`, `name`, `isPrimary`, `isEnabled`, `bounds`, `workArea`, `dpiX`, `dpiY`, `rotation`, `displayZoomFactor`, `modes`, `overscan`, `hasTouchSupport`, `edid`); `Bounds` (`left`, `top`, `width`, `height`); `Insets` (`left`, `top`, `right`, `bottom`); `LayoutPosition` (`"top"`, `"right"`, `"bottom"`, `"left"`); `MirrorMode` (`"off"`, `"normal"`, `"mixed"`); `ActiveState` (`"active"`, `"inactive"`).

```js
const displays = await chrome.system.display.getInfo();
chrome.system.display.onDisplayChanged.addListener(() => {});
```

## chrome.system.memory
- **Permission:** `"system.memory"`. **Method:** `getInfo()` resolves to a `MemoryInfo` (`capacity` — total physical memory in bytes; `availableCapacity` — bytes).

```js
const mem = await chrome.system.memory.getInfo();
```

## chrome.system.storage
- **Permission:** `"system.storage"`.
- **Key methods:** `getInfo()` resolves to `StorageUnitInfo[]`; `ejectDevice(id)` resolves to an `EjectDeviceResultCode`; `getAvailableCapacity(id)` (Dev channel).
- **Key events:** `onAttached` → `(StorageUnitInfo)`; `onDetached` → `(id)`.
- **Key types/enums:** `StorageUnitInfo` (`id` — transient, valid only within one app run; `name`; `type`; `capacity` bytes); `StorageUnitType` (`"fixed"`, `"removable"`, `"unknown"`); `EjectDeviceResultCode` (`"success"`, `"in_use"`, `"no_such_device"`, `"failure"`).

```js
const units = await chrome.system.storage.getInfo();
```

## chrome.systemLog
- **Permission:** `"systemLog"`. ChromeOS only; Chrome 125+; requires enterprise policy. **Method:** `add(options)` where `options` is `{ message }`.

```js
await chrome.systemLog.add({ message: "Extension reached checkpoint A" });
```

## chrome.printing
- **Permission:** `"printing"`. ChromeOS only; Chrome 81+.
- **Key methods:** `getPrinters()` resolves to `Printer[]`; `getPrinterInfo(printerId)` resolves to a `GetPrinterInfoResponse`; `submitJob(request)` resolves to a `SubmitJobResponse` (user prompted unless allowlisted via the `PrintingAPIExtensionsAllowlist` policy); `cancelJob(jobId)`; `getJobStatus(jobId)` (Chrome 135+).
- **Key events:** `onJobStatusChanged` → `(jobId, status)`.
- **Key types/enums:** `Printer` (`id`, `name`, `description`, `uri`, `source`, `isDefault`); `SubmitJobRequest` (`job` is a `PrintJob`; content types `"application/pdf"`, `"image/png"`); `JobStatus` (`"PENDING"`, `"IN_PROGRESS"`, `"FAILED"`, `"CANCELED"`, `"PRINTED"`); `SubmitJobStatus` (`"OK"`, `"USER_REJECTED"`); `PrinterSource` (`"USER"`, `"POLICY"`).
- **Constants:** `MAX_GET_PRINTER_INFO_CALLS_PER_MINUTE` = 20; `MAX_SUBMIT_JOB_CALLS_PER_MINUTE` = 40.

```js
const printers = await chrome.printing.getPrinters();
chrome.printing.onJobStatusChanged.addListener((jobId, status) => console.log(jobId, status));
```

## chrome.printerProvider
- **Permission:** `"printerProvider"`. Event-only API (Chrome 44+).
- **Key events** (each listener gets a `resultCallback`): `onGetPrintersRequested` → `(resultCallback)`; `onGetCapabilityRequested` → `(printerId, resultCallback)`; `onGetUsbPrinterInfoRequested` → `(device, resultCallback)` (Chrome 45+); `onPrintRequested` → `(printJob, resultCallback)`.
- **Key types/enums:** `PrinterInfo` (`id`, `name`, `description?`); `PrintJob` (`printerId`, `ticket`, `contentType` (`"application/pdf"` or `"image/pwg-raster"`), `document`, `title`); `PrintError` (`"OK"`, `"FAILED"`, `"INVALID_TICKET"`, `"INVALID_DATA"`).

```js
chrome.printerProvider.onGetPrintersRequested.addListener((resultCallback) => {
  resultCallback([{ id: "printer-1", name: "My Virtual Printer" }]);
});
```

## chrome.printingMetrics
- **Permission:** `"printingMetrics"`. ChromeOS only; Chrome 79+ (`getPrintJobs` Chrome 96+); requires enterprise policy.
- **Key methods:** `getPrintJobs()` resolves to `PrintJobInfo[]`. **Events:** `onPrintJobFinished` → `(jobInfo)`.
- **Key types/enums:** `PrintJobInfo` (`id`, `title`, `creationTime` (ms past epoch), `completionTime`, `numberOfPages`, `printer`, `settings`, `source`, `status`); `ColorMode` (`"BLACK_AND_WHITE"`, `"COLOR"`); `DuplexMode` (`"ONE_SIDED"`, `"TWO_SIDED_LONG_EDGE"`, `"TWO_SIDED_SHORT_EDGE"`); `PrintJobStatus` (`"PRINTED"`, `"CANCELED"`, `"FAILED"`).

```js
const jobs = await chrome.printingMetrics.getPrintJobs();
```

## chrome.documentScan
- **Permission:** `"documentScan"`. ChromeOS only. `scan` since Chrome 44; session methods since Chrome 125.
- **Key methods:** `scan(options)` resolves to `ScanResults` (`dataUrls`, `mimeType`); session methods (Chrome 125+): `getScannerList(filter)`, `openScanner(scannerId)`, `getOptionGroups(scannerHandle)`, `setOptions(scannerHandle, options)`, `startScan(scannerHandle, options)`, `readScanData(job)`, `cancelScan(job)`, `closeScanner(scannerHandle)`.
- **Key enums:** `OperationResult` (`SUCCESS`, `UNSUPPORTED`, `CANCELLED`, `DEVICE_BUSY`, `ADF_JAMMED`, `ADF_EMPTY`, `COVER_OPEN`, `IO_ERROR`, `ACCESS_DENIED`, `EOF`, and more); `OptionType` (`BOOL`, `INT`, `FIXED`, `STRING`, `BUTTON`, `GROUP`); `OptionUnit` (`UNITLESS`, `PIXEL`, `MM`, `DPI`, `PERCENT`).

```js
const result = await chrome.documentScan.scan({ mimeTypes: ["image/png"] });
console.log(result.dataUrls.length, result.mimeType);
```

## chrome.accessibilityFeatures
- **Permissions:** `"accessibilityFeatures.read"` to read; `"accessibilityFeatures.modify"` to change (modify does not imply read).
- **Surface:** each feature is a `ChromeSetting` with `get(details)`, `set(details)`, `clear(details)` (Promises). Properties (ChromeOS unless noted): `spokenFeedback`, `largeCursor`, `stickyKeys`, `highContrast`, `screenMagnifier`, `autoclick`, `virtualKeyboard`, `caretHighlight`, `cursorHighlight`, `cursorColor`, `dockedMagnifier`, `focusHighlight`, `selectToSpeak`, `switchAccess`, `dictation`, `animationPolicy` (value one of `"allowed"`, `"once"`, `"none"`).

```js
const state = await chrome.accessibilityFeatures.spokenFeedback.get({});
await chrome.accessibilityFeatures.highContrast.set({ value: true }); // needs modify
```
