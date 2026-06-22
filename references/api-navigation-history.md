# Navigation, History, Sessions & Search

> Manifest V3 reference. Original prose; identifiers preserved verbatim.

## Overview
These APIs observe navigation, read/edit history, bookmarks, and the reading list, surface top sites and closed sessions, inspect processes (dev only), and run default-provider searches. All are Promise-based in MV3.

## chrome.webNavigation
- **Permission:** `"webNavigation"` (no host permissions required).
- **Key methods** (Promises, Chrome 93+): `getFrame(details)` where `details` is `{ tabId?, frameId?, documentId?, processId? }`; `getAllFrames(details)` where `details` is `{ tabId }`.
- **Key events** (success order): `onBeforeNavigate` → `onCommitted` → `onDOMContentLoaded` → `onCompleted`; failures fire `onErrorOccurred`. Also `onCreatedNavigationTarget`, `onReferenceFragmentUpdated`, `onHistoryStateUpdated`, `onTabReplaced`. Detail objects include `tabId`, `frameId`, `parentFrameId`, `url`, `timeStamp`, `processId`, `documentId`, `documentLifecycle`, `frameType`, `parentDocumentId`. `onCommitted` adds `transitionType` and `transitionQualifiers`.
- **Key types:** event filters accept `{ url: [UrlFilter] }`. `TransitionType`: `"link"`, `"typed"`, `"auto_bookmark"`, `"auto_subframe"`, `"manual_subframe"`, `"generated"`, `"start_page"`, `"form_submit"`, `"reload"`, `"keyword"`, `"keyword_generated"`.

```js
chrome.webNavigation.onCommitted.addListener(
  (d) => console.log(d.url, d.transitionType),
  { url: [{ hostContains: "example.com" }] }
);
```

## chrome.sessions
- **Permission:** `"sessions"`.
- **Key methods** (Promises): `getRecentlyClosed(filter?)`, `getDevices(filter?)`, `restore(sessionId?)` (omit to restore most recent). Constant `MAX_SESSION_RESULTS` = `25`.
- **Key events:** `onChanged`.
- **Key types:** `Filter` (`maxResults`), `Session` (`lastModified`, `tab?`, `window?`), `Device` (`deviceName`, `sessions`).

```js
const closed = await chrome.sessions.getRecentlyClosed({ maxResults: 10 });
await chrome.sessions.restore();
```

## chrome.history
- **Permission:** `"history"`.
- **Key methods** (Promises): `search(query)` with fields `text`, `startTime?`, `endTime?`, `maxResults?` (default `100`); `getVisits(details)`, `addUrl(details)`, `deleteUrl(details)`, `deleteRange({ startTime, endTime })`, `deleteAll()`.
- **Key events:** `onVisited`, `onVisitRemoved`.
- **Key types:** `HistoryItem` (`id`, `url`, `title`, `lastVisitTime`, `visitCount`, `typedCount`). `VisitItem` (`id`, `visitId`, `visitTime`, `referringVisitId`, `transition`, `isLocal`).

```js
const items = await chrome.history.search({
  text: "", maxResults: 10, startTime: Date.now() - 7 * 24 * 60 * 60 * 1000,
});
```

## chrome.topSites
- **Permission:** `"topSites"`. **Method:** `get()` resolves to `MostVisitedURL[]` (`title`, `url`).

```js
const sites = await chrome.topSites.get();
```

## chrome.bookmarks
- **Permission:** `"bookmarks"`.
- **Key methods** (Promises): `create(bookmark)`, `get(idOrIdList)`, `getChildren(id)`, `getRecent(numberOfItems)`, `getTree()`, `getSubTree(id)`, `search(query)`, `move(id, destination)`, `update(id, changes)`, `remove(id)`, `removeTree(id)`.
- **Key events:** `onCreated`, `onRemoved`, `onChanged`, `onMoved`, `onChildrenReordered`, `onImportBegan`, `onImportEnded`.
- **Key types:** `BookmarkTreeNode` (`id`, `parentId`, `index`, `url`, `title`, `children`, `dateAdded`, `dateGroupModified`, `dateLastUsed`, `unmodifiable`, `folderType`, `syncing`). `CreateDetails` (`parentId`, `index`, `title`, `url`). Deprecated, unenforced constants `MAX_WRITE_OPERATIONS_PER_HOUR` = `1000000`, `MAX_SUSTAINED_WRITE_OPERATIONS_PER_MINUTE` = `1000000`.

```js
const node = await chrome.bookmarks.create({
  parentId: "1", title: "Docs", url: "https://example.com/",
});
```

## chrome.readingList
- **Permission:** `"readingList"`. Chrome 120+.
- **Key methods** (Promises): `addEntry(entry)` (`url`, `title`, `hasBeenRead`); `query(info)` (`url?`, `title?`, `hasBeenRead?`); `updateEntry(info)`; `removeEntry(info)` (`url`).
- **Key events:** `onEntryAdded`, `onEntryRemoved`, `onEntryUpdated` (each receives a `ReadingListEntry`).
- **Key types:** `ReadingListEntry` (`url`, `title`, `hasBeenRead`, `creationTime`, `lastUpdateTime`). Entries are keyed by full URL.

```js
await chrome.readingList.addEntry({ url: "https://example.com/", title: "Read me", hasBeenRead: false });
```

## chrome.processes
- **Permission:** `"processes"`. **Dev channel only** (not stable).
- **Key methods** (Promises): `getProcessIdForTab(tabId)`, `getProcessInfo(processIds, includeMemory)`, `terminate(processId)`.
- **Key events:** `onCreated`, `onUnresponsive`, `onUpdated`, `onUpdatedWithMemory`, `onExited`.
- **Key types:** `Process` (`id`, `osProcessId`, `type`, `cpu`, `network`, `privateMemory`, `jsMemoryAllocated`, `jsMemoryUsed`). `ProcessType`: `"browser"`, `"renderer"`, `"extension"`, `"gpu"`, `"utility"`, `"nacl"`, `"other"`.

## chrome.search
- **Permission:** `"search"`. Chrome 87+. **Method:** `query(queryInfo)` where `queryInfo` is `{ text (required), disposition?, tabId? }`. `Disposition`: `"CURRENT_TAB"`, `"NEW_TAB"`, `"NEW_WINDOW"`.

```js
await chrome.search.query({ text: "extensions", disposition: "NEW_TAB" });
```
