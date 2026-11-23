# useBus

```ts
const bus = appState().useBus();     // workerState().useBus() in the worker
const off = bus.on('rates:updated', (data) => { ... });   // returns an unsubscribe function
bus.once('report:ready', (data) => { ... });
bus.emit('rates:updated', { base: 'USD' });
bus.broadcast({ verb: 'update', room: 'tots', data: [tot] });
```

`useBus()` always exists. Which bus you get depends on `config/bus.ts`:

| | No bus config (server only) | Bus configured |
|---|---|---|
| `bus.name` | `DevBus` | `ProdBus` |
| `emit` / `on` / `once` | in-process listeners | Redis pub/sub on a channel named after the event; payload JSON-encoded |
| `broadcast(payload)` | `IO.emit('comets', payload)` to this server's sockets | socket.io Redis emitter: `comets` to every socket of every server process |

- **Event names** are global Redis channels. Namespace them (`<feature>:<event>`) and keep them in a shared constants file, as with queue names.
- **Payloads** must be JSON-serialisable on `ProdBus`.
- **Broadcast shape.** Model writes broadcast one `{ verb: 'create' | 'update' | 'destroy', room: '<model>', data: record }` per record. Send the same shape so clients reuse one `comets` handler.
- **`ProdBus.on` unsubscribe** drops the whole channel, including other listeners for that event in the same process. Register one listener per event and fan out yourself.

## Bus vs queue

- **Bus:** fire-and-forget notifications that are fine to miss (nothing is stored), and pushes to connected browsers.
- **Queue:** work that must happen once, survive restarts, or report success or failure (`addJob`/`processJob`, see the `appbee-worker` skill).

## Known limitation (appbee ≤ 1.3.4, node-redis v4)

`ProdBus` builds its publisher and subscriber with `redisClient.duplicate()` and never calls `connect()`. The REST server does the same for the socket.io Redis adapter's subscriber. node-redis v4 rejects commands on an unconnected client with `ClientClosedError`. As a result:

- `emit`/`on`/`once` on `ProdBus` fail, as unhandled promise rejections.
- Cross-process `broadcast` may not reach sockets.

In-process `DevBus` and same-process socket broadcasts are unaffected.

Until appbee connects the duplicates:

- Use queues for cross-process signalling.
- Test any `ProdBus` path against a real Redis before relying on it.
- Fix upstream in `tools/event-bus.ts` and `rest/server.ts` by awaiting `connect()` on each duplicate.
