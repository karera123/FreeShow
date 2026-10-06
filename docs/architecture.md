# Architecture

How the source is split, what each part builds to, and how the app's windows relate to each other.

## Source roots

Everything is under `src/`:

| Root | What it is | Built by | Output |
| --- | --- | --- | --- |
| `src/electron/` | Electron main process | `tsc` (`config/typescript/tsconfig.electron.json`) | `build/electron/`, entry `build/electron/index.js` |
| `src/frontend/` | Svelte 3 renderer | Vite (`vite.config.mjs`) | `public/build/bundle.js` and `bundle.css` |
| `src/server/` | Four Svelte web apps: `remote`, `stage`, `controller`, `output_stream` | Vite (`config/building/vite.config.servers.mjs`) | `build/electron/<id>/` |
| `src/types/` | Types shared by all of the above | - | - |

Both the main process and the frontend import from `src/types/`, and `src/types/` imports some types back from `src/electron/`. Code shared between the web apps is in `src/server/common/`.

## Dev versus production loading

- In dev, Vite serves the frontend with `public/` as its root on `127.0.0.1:3000`, and the main window loads `http://localhost:3000`.
- In production, the frontend is bundled as a single IIFE and the main window loads `public/index.html` from disk.
- `isProd` in `src/electron/index.ts` decides which. It is true when `NODE_ENV` is `production` or when the executable is not the dev Electron binary.

`scripts/preBuild.js` clears `build/` and `public/build/`, generates the `tsconfig.*.prod.json` files (source maps off) and copies the pdf.js worker into `public/assets/`.

## Windows

The main window and every output window load the same frontend bundle.

- `src/frontend/utils/startup.ts` sets the `currentWindow` store: `null` for the main window, `"output"` for an output window, `"pdf"` for the PDF export window.
- `src/frontend/App.svelte` switches on that store between `MainLayout.svelte` (the editor UI) and `MainOutput.svelte` (what the audience sees).
- Output windows do not own state. The main window pushes state to them over the `OUTPUT` IPC channel, and an output window asks for it on startup with `REQUEST_DATA_MAIN`.
- The Electron side of output windows (creation, displays, capture) is in `src/electron/output/`.

## Where feature areas live

- `src/frontend/components/` - UI, grouped by area (`drawer`, `edit`, `show`, `slide`, `stage`, `output`, `settings`, `timeline`, ...). Non-UI logic shared across components is in `src/frontend/components/helpers/`.
- `src/frontend/converters/` - importers for other presentation software and Bible formats.
- `src/electron/contentProviders/` - Planning Center, OnStage and similar integrations behind a shared base.
- `src/electron/cloud/` - sync and backup to cloud storage.
- `src/electron/ndi`, `omt`, `blackmagic`, `capture`, `streaming` - video output paths that depend on native modules. Several are optional or platform-specific.
- `src/electron/ai/` and `src/frontend/ai/` - speech-to-text and scripture detection.
