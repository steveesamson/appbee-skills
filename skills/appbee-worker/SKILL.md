---
name: appbee-worker
description: Builds appbee background processing — the work() worker process, BullMQ queues via useQueue().addJob and useWorker().processJob, job events with on('completed'|'failed'), and jobs/<name>.ts with useJob. Use when adding or changing an appbee worker, queue, producer, consumer, background job, or job listener.
---

# appbee worker

Background work is a queue shared by two sides:

- The **producer** (usually the REST server, often from a model hook) calls `useQueue(name).addJob(payload)`.
- The **consumer** (a separate `work()` process) calls `useWorker(name).processJob(fn)`.

Both sides reach the same Redis through `config/bus.ts` and must use the **same queue name**. Keep the names in one shared module (the samples use `constants/queues.ts`).

## Steps

1. **Configure the bus** in `config/bus.ts`. Use `host`, `port`, and `password`; the `url` key is not honoured.

   ```ts
   import type { RedisStoreConfig } from 'appbee';
   const bus: RedisStoreConfig = { host: '127.0.0.1', port: 6379, password: 'foobar' };
   export default bus;
   ```

   The REST server only exposes `useQueue`/`useWorker`/`useRedis` when this file exists. `work()` refuses to start without it.

2. **Name the queue** in the shared constants: `export const MAIL_CREATED = 'mailCreated';`.

3. **Produce.** Enqueue after the triggering write, normally in a model `post*` hook (see the `appbee-model` skill, [hooks.md](../appbee-model/hooks.md)):

   ```ts
   const queue = appState().useQueue?.(MAIL_CREATED);
   await queue?.addJob({ ...mail, id: `${mail.id}` });
   ```

4. **Write the worker entry** pointing `work()` at a base folder with the same `config/` and `modules/` as the server. The samples copy `stream-bee/src/bee` into the worker and strip controllers and policies. `work()` loads config, data sources, models, plugins, and `jobs/`; it never loads controllers, policies, or middlewares.

   ```ts
   import path from 'node:path';
   import { fileURLToPath } from 'node:url';
   import { work, workerState, utils, v, x } from 'appbee';
   import { MAIL_CREATED } from './bee/constants/queues.js';

   const __dirname = path.dirname(fileURLToPath(import.meta.url));

   work(path.join(__dirname, 'bee'), async () => {
     const { useWorker, useBus, useJob, model } = workerState();
     const req = { source: utils.useSource('core') };

     useWorker(MAIL_CREATED).processJob(async (job) => {
       const Outmails = model.Outmails(req);
       const id = v.parse(x.objectId(), job.data.id);
       await useJob('sendMail')(job.data);
       const { data } = await Outmails.update({ query: { id }, data: { sent: true } });
       useBus().broadcast({ verb: 'update', room: 'outmails', data });
       return { sent: true };
     });
   });
   ```

   Run it with `NODE_ENV=development` under a TypeScript runner (`tsx`, or `ts-node/esm`) so `.ts` files load; production runs the compiled `.js`.

5. **Done when** every `addJob` queue name has exactly one `processJob` consumer. Each processor must also:
   - re-parse ids from `job.data` before querying MongoDB;
   - throw on failure rather than returning an error object, so the job is marked failed;
   - return only JSON-serialisable values.

## addJob (producer)

`useQueue(name).addJob(payload, tag?)` returns `Promise<Job>`. It is available on `appState()` (when the bus is configured) and on `workerState()`, so a worker can chain work into another queue.

- **`payload` is JSON-serialised.** `ObjectId`s and `Date`s arrive as strings, and functions and class instances are lost. Send ids and plain fields, not live objects.
- **`tag`** becomes `job.name` (it defaults to the queue name). Use it to multiplex related kinds on one queue and branch on `job.name` in the processor.
- **Fixed options:** completed jobs are removed immediately, the last 5000 failures are kept, and there are no retries or delay. For delayed work, schedule with `utils.useCronMaster` in the worker (see the `appbee-utils` skill).
- `useQueue(name)` is memoised per name, so call it where you need it rather than caching it in module scope.
- Calling `addJob` on a `useWorker` instance throws `You cannot add jobs to a worker`.

## processJob (consumer)

`useWorker(name).processJob(async (job) => result)` starts one BullMQ worker for the queue.

- **`job.data`** is the payload; `job.name` is the tag; `job.id` correlates with the producer's `Job.id`.
- **Returning** marks the job completed, and the value becomes `returnvalue` for `on('completed')`. **Throwing** marks it failed with `failedReason`; BullMQ worker `error` events are logged with the queue name.
- **Models need a source.** Pass `{ source: utils.useSource('core') }` (or a tenant key) as `req`. Direct model calls fire no hooks; call `Model.publishUpdate(req, data)` when the `post*` side effects should run.
- **Call `processJob` once per queue name.** Each call starts another BullMQ `Worker` on the same queue.
- Calling `processJob` on a `useQueue` instance throws `You cannot process jobs on a non-worker`.

## on (job events)

`queueOrWorker.on(event, handler)` listens to BullMQ queue events for that queue name, from **any** process: the producer can learn about completion without polling.

```ts
const queue = appState().useQueue!(REPORT_REQUESTED);
queue.on('completed', ({ jobId, returnvalue }) => { ... });
queue.on('failed', ({ jobId, failedReason }) => { ... });
const job = await queue.addJob({ reportId });
// correlate later events by job.id
```

- The events are `'completed' | 'failed'`. The handler receives `{ jobId, returnvalue }` or `{ jobId, failedReason }`.
- `returnvalue` arrives as BullMQ stored it, which may be a JSON string, so parse it defensively.
- The job record itself is already gone after completion, so put everything the listener needs in `returnvalue`.
- Register listeners once at startup (in the worker app function, or a server plugin initialised once), not per request.

## jobs/ and useJob

`jobs/*.ts` hold reusable job functions. Every **named export** is merged into `workerState().useJob` (a file without exports contributes nothing). Type them by augmenting `Jobs`:

```ts
// jobs/sendMail.ts
import type { Params } from 'appbee';

export const sendMail = async (mail: Params) => { /* ... */ };

declare global {
  interface Jobs {
    sendMail: typeof sendMail;
  }
}
```

`useJob('sendMail')` returns the typed function in the worker only. A job with the processor's signature plugs in directly: `useWorker(MAIL_CREATED).processJob((job) => useJob('sendMail')(job.data))`.

## Worker state vs server state

In the worker, `appState()` is empty: use `workerState()` for `model`, `plugin`, `useQueue`, `useWorker`, `useBus`, `useRedis`, `useJob`, and `env`. Model hooks and plugins shared with the server that read `appState()` break there. Resolve state per process, and read config with `utils.useConfig(...)`, which works in both (see the `appbee-plugin` skill).

`useBus().broadcast({ verb, room, data })` is the designed path from a worker to browsers: the same `comets` event and payload shape as model broadcasts, relayed through Redis. It currently depends on Redis clients that appbee duplicates but never connects (see `bus.md` in the `appbee-utils` skill). Verify that it arrives before relying on it; otherwise hand results back through a queue the server consumes.
