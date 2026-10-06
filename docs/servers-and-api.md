# Companion servers and external API

How the web apps for phones and tablets reach the app, and how the automation API is wired.

## Companion web servers

`src/electron/servers.ts` runs one express + socket.io server per web app in `src/server/`:

| Server | Port | Purpose |
| --- | --- | --- |
| `REMOTE` | 5510 | Remote control of shows and slides |
| `STAGE` | 5511 | Stage display for people on stage |
| `CONTROLLER` | 5512 | Simple next/previous controller |
| `OUTPUT_STREAM` | 5513 | Stream of the output to a browser |

Each server serves its built web app from `build/electron/<id>/` and publishes its port with Bonjour.

The main process is only a relay. A socket message from a web app is forwarded to the main window over the IPC channel with the same name as the server, and the frontend answers from:

- `src/frontend/utils/remoteTalk.ts`
- `src/frontend/utils/stageTalk.ts`
- `src/frontend/utils/controllerTalk.ts`

The web apps have no direct access to app state. Anything a web app shows has to be sent to it by the frontend, and a new feature there usually needs a change on both sides: the handler in the `*Talk.ts` file and the receiving code in `src/server/<id>/`.

The `remote` and `stage` apps keep their own small set of stores (`src/server/<id>/util/stores.ts`), separate from `src/frontend/stores.ts`.

## External automation API

`src/electron/utils/api.ts` serves the public API over WebSocket (default port 5505), REST (5506) and OSC, with optional password auth.

The server side only transports requests. It passes each one to the renderer with `ToMain.API_TRIGGER2`, and the action runs there:

- `API_ACTIONS` in `src/frontend/components/actions/api.ts` maps every action id to its function, and the `API_*` types in the same file define the payloads.
- `triggerAction()` in that file is the entry point.

The same `API_ACTIONS` table backs the in-app actions in `src/frontend/components/actions/actions.ts`, so adding an entry there makes it available both to the API and inside the app.
