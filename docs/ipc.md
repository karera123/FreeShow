# IPC

How the renderer and the Electron main process talk to each other, and what to change when adding a call.

## Transport

The renderer has no Node access. `src/electron/preload.ts` exposes `window.api` with `send`, `receive` and `removeListener`, and nothing else reaches the main process.

Channel names are in `src/types/Channels.ts`: `MAIN`, `OUTPUT`, `EXPORT`, `CLOUD`, `NDI`, `OMT`, `BLACKMAGIC`, `AUDIO`, plus one per companion web server (`REMOTE`, `STAGE`, `CONTROLLER`, `OUTPUT_STREAM`). The main-process listeners are registered at the bottom of `src/electron/index.ts`.

In dev, the preload logs every message to the console, except a list of noisy ones in `filteredChannelsData`.

## The typed MAIN channel

Nearly everything goes through `MAIN`. Each message is `{ channel, data }`, where `channel` is an id from one of two enums:

| File | Contents |
| --- | --- |
| `src/types/IPC/Main.ts` | `Main` enum: requests from the renderer to the main process. `MainSendPayloads` types what is sent, `MainReturnPayloads` types the reply. |
| `src/types/IPC/ToMain.ts` | `ToMain` enum: pushes and requests from the main process to the renderer, with its own payload maps. |
| `src/electron/IPC/responsesMain.ts` | One handler per `Main` id. Whatever the handler returns is sent back as the reply. |
| `src/frontend/IPC/responsesMain.ts` | Handlers for `ToMain` ids, and for `Main` replies that arrive without a pending request (for example the stores sent at startup). |

Helpers:

- `src/frontend/IPC/main.ts`: `sendMain` (fire and forget), `requestMain` (awaitable), `receiveMain` and `receiveToMain` (subscribe).
- `src/electron/IPC/main.ts`: the mirror, `sendToMain`, `requestToMain` and `sendMain`.

`requestMain` and `requestToMain` match a reply to its request by a generated listener id. Both time out after 15 seconds and then resolve with `undefined` (renderer) or `null` (main) instead of rejecting, so callers must handle an empty result.

## Adding a main-process call

1. Add the id to the `Main` enum in `src/types/IPC/Main.ts`, with its entry in `MainSendPayloads` and, if it returns something, `MainReturnPayloads`.
2. Add the handler to `mainResponses` in `src/electron/IPC/responsesMain.ts`.
3. Call it from the frontend with `requestMain(Main.X, data)` or `sendMain(Main.X, data)`.

For a push from the main process, do the same with `ToMain`: the enum and payloads in `src/types/IPC/ToMain.ts`, a handler in `src/frontend/IPC/responsesMain.ts`, and `sendToMain(ToMain.X, data)` on the Electron side.

## Other channels

Channels other than `MAIN` are untyped. They use `send(CHANNEL, ["ID"], data)` and `receive(CHANNEL, { ID: handler })` from `src/frontend/utils/request.ts`. The receivers are collected in `src/frontend/utils/receivers.ts`.
