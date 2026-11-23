---
name: appbee-plugin
description: Writes appbee plugins at plugins/<name>.ts — named exports merged into the global Plugins interface and reached with utils.usePlugin, appState().plugin, or workerState().plugin. Use when adding or changing an appbee plugin, shared service, third-party client wrapper, or the Plugins interface.
---

# appbee plugin

A plugin is a **named export** of any file in `<base>/plugins/`. Both `serve()` and `work()` import every file there after models load, and merge all named exports into one map. The map is reached by name:

- `utils.usePlugin('name')` works in either process; prefer it.
- `appState().plugin.name` works on the server.
- `workerState().plugin.name` works in the worker.

Use a plugin for code shared by controllers, policies, model hooks, and jobs: third-party clients (S3, web-push, mail), access control, or helpers that write to other models. Request gates belong in `policies/`, and per-request middleware in `middlewares/`.

## Steps

1. **Create `plugins/<file>.ts`** and export a factory that returns a lazily-created singleton. This is the samples' shape (`acl`, `s3`, `bucket`):

   ```ts
   import { utils, type Params } from 'appbee';
   import * as webpush from 'web-push';

   class Notifier {
     private static instance: Notifier;
     private constructor() {
       const { notifications: { mailTo, publicKey, privateKey } } = utils.useConfig('security');
       webpush.setVapidDetails(mailTo, publicKey, privateKey);
     }
     static getInstance() {
       return (Notifier.instance ??= new Notifier());
     }
     async notify(subscription: webpush.PushSubscription, message: Params) {
       await webpush.sendNotification(subscription, JSON.stringify(message));
     }
   }

   export const notifier = (): Notifier => Notifier.getInstance();

   declare global {
     interface Plugins {
       notifier: () => Notifier;
     }
   }
   ```

2. **Augment `Plugins`** in the same file with `declare global { interface Plugins { ... } }`, one entry per export, typed `typeof` or with an explicit signature. This is what types `usePlugin('notifier')`.
3. **Call it** where it's needed: `const n = utils.usePlugin('notifier')(); await n.notify(sub, payload);`.
4. **Done when** every named export has a matching `Plugins` entry, export names are unique across all of `plugins/` (a later file silently overwrites an earlier one), and the file does nothing at import time beyond declarations.

## Rules

- **Initialise lazily.** Plugin files are imported before `appState()`/`workerState()` are filled. Read config, models, or env inside the factory or constructor, never at module top level.
- **Read config with `utils.useConfig(key)`.** It works in both processes. `appState().env.security` only exists on the server, which is why the sample `webpush` plugin breaks when copied into the worker.
- **Take `req` to touch models.** Model methods need a source, so accept `req: RequestAware` and call `model.Bucket(req)`. Callers pass `req` from a handler or hook, or `{ source: utils.useSource('core') }` from a worker. For a plugin used in both processes, resolve the registry with `(appState().model ?? workerState().model)`.
- **Return `{ data?, error? }`** from fallible async methods, matching model results, so callers branch the same way everywhere.
- **Fire-and-forget deliberately.** Side writes called from hooks (such as the sample `bucket.create(req, ...)`) needn't be awaited, but must catch their own errors.
- **Keep process-specific dependencies out of shared folders.** Every file in `plugins/` is imported by each process that has the folder. The worker sample deletes `plugins/s3.ts` (and its `aws-sdk` import) when it copies the server's base folder.

## Multiple exports

One file may export several functions. Each becomes its own key:

```ts
export const addCron = (cron: CronConfig) => utils.useCronMaster.add(cron);
export const initCron = () => { /* page models and enqueue */ };

declare global {
  interface Plugins {
    addCron: typeof addCron;
    initCron: typeof initCron;
  }
}
```

Prefer one factory per concern (`acl()`, `s3()`) over many loose functions: the factory groups related methods behind one key and controls initialisation.
