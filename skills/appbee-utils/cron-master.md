# utils.useCronMaster

A process-wide registry of `node-cron` tasks, keyed by `key`. It is an **object**, not a function: `utils.useCronMaster.add(...)`. Run crons in the worker, so that scaling the REST server never multiplies them.

```ts
import { defineCron, utils } from 'appbee';    // defineCron is a typed identity; omit it if your appbee lacks it

const cancel = utils.useCronMaster.add(defineCron({
  key: `service:${svc.id}`,       // unique; re-adding a key replaces the old task
  name: svc.name,
  schedule: svc.cronExpression,   // invalid expressions log and return undefined
  timeZone: svc.timeZone,         // default: the host time zone
  enabled: true,
  immediate: false,               // true also runs the task once right away
  task: () => workerState().useQueue(svc.job).addJob({ ...svc.params, serviceId: `${svc.id}` }),
}));
```

| Member | Behaviour |
|---|---|
| `add(cron)` | Schedules the task and returns a cancel function, or `undefined` for an invalid schedule. |
| `start(key)` / `stop(key)` | Resume or pause; returns `false` for an unknown key. |
| `evict(key)` | Stop and remove. |
| `stopAll()` | Pause every task. |
| `has(key)`, `length` | Inspect the registry. |
| `dateToCronExpression(date)` | `'m H d M *'` for that minute, which recurs yearly. Stop or evict the task after its first run for one-shot jobs. |
| `init(req)` | Built-in boot routine. It requires a model registered as **`Service`** (folder `service`) and a `cronJobs` queue. It pages `Service` rows with `status` `running`/`starting` and enqueues each on `cronJobs` with `command: 'start'`. |

## The command-queue pattern (from the samples)

Clients control crons by updating a `Services` record:

1. The REST server's `Services.postUpdate` reads the transient `command` (`'start' | 'stop'`) from the request and enqueues the service on `cronJobs`.
2. The worker consumes `cronJobs`, then calls `add(...)`/`stop(key)` keyed by the service id.
3. On boot, the worker re-adds every running service.

This keeps HTTP fast and keeps all schedules in one process.

`init()` is hard-wired to the `Service` model name, and the sample's model is `Services` (folder `services`). When the names differ, write the boot routine yourself: `utils.useDataPager` over the services model, feeding `add(...)` or the `cronJobs` queue. The sample `plugins/cron` does exactly this.

## Gotchas

- Tasks run in-process. A second worker replica runs every cron twice, so schedule from exactly one replica, or make the task only enqueue (and dedupe in the consumer).
- `crons/` files and `defineCron` are not auto-loaded. Nothing is scheduled until code calls `add`.
- The task's return value is ignored. Catch errors inside the task, or make it enqueue, so failures surface through queue events.
