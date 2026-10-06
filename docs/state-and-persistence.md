# State and persistence

Where app state lives in the renderer, how it gets to disk and back, and how undo/redo fits in.

## Stores

`src/frontend/stores.ts` holds every global as a Svelte `writable`. There is no other state layer. The file is split into two halves:

- **TEMPORARY VARIABLES** - session state such as the active show, selection, open popup and playback status. Lost on exit.
- **SAVED VARIABLES** - user data and settings. Written to disk.

## Saving

1. `save()` in `src/frontend/utils/save.ts` collects the saved stores into one object and sends `Main.SAVE`.
2. `src/electron/data/save.ts` writes each part to its store file.

## Loading

At startup the main process sends each store's content over `MAIN`. The receivers in `src/frontend/IPC/responsesMain.ts` put it into the Svelte stores; settings go through `updateSettings` and `updateSyncedSettings` in `src/frontend/utils/updateSettings.ts`.

## Adding a persisted store

A new saved store has to appear in all of these, or it will silently not round-trip:

- the store itself in `src/frontend/stores.ts`
- the save object in `src/frontend/utils/save.ts`
- the load path in `src/frontend/utils/updateSettings.ts` or the matching receiver in `src/frontend/IPC/responsesMain.ts`
- a default in `src/electron/data/defaults.ts`

## Store files

`src/electron/data/store.ts` defines the JSON store files in `storeFilesData`. Each has a `portable` flag:

- `portable: true` (synced settings, themes, projects, stage layouts, overlays, templates, events) is stored in the user-chosen data folder, and moves with it when the user changes location.
- `portable: false` (device settings, caches, history, media metadata, usage, error log, credentials) stays in the app data directory.

Removed default keys are dropped on load: defaults replace the keys of a stored object.

The `FS_MOCK_STORE_PATH` environment variable redirects all stores to another directory. The Playwright test uses it.

## Shows

Shows are not in a store file. Each show is its own `.show` file in the data folder (`src/electron/utils/shows.ts`).

- The `shows` store holds only a trimmed index (`TrimmedShows`: name, category, timestamps).
- Full show content is loaded on demand into `showsCache` by `loadShows()` in `src/frontend/components/helpers/setShow.ts`.
- Read and modify show content through the `_show()` helper in `src/frontend/components/helpers/shows.ts` (for example `_show().slides([id]).set({ key, value })`). It defaults to the active show.

A show must be in `showsCache` before it can be read, so code that touches a show that may not be open has to load it first.

## Undo and redo

User-visible changes go through `history()` in `src/frontend/components/helpers/history.ts`, not through direct store writes. A history entry (`History` in `src/types/History.ts`) carries an id, the old and new data and a location, and the handlers that apply or revert it are in `historyActions.ts` and `historyHelpers.ts` next to it.

A change written straight to a store works but cannot be undone.
