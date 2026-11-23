---
name: appbee-utils
description: Reference for appbee runtime helpers — useBus, useQueue, useWorker, useRedis, useJob (process state) and utils.useDataLoader, useDataPager, useCronMaster, usePlugin, useConfig, useSource, useToken, useEncrypt. Use when an appbee handler, hook, plugin, policy, or job needs events, Redis, joins, paging, cron, config, data sources, JWTs, or password hashing.
---

# appbee utils

The helpers come from two places. Pick the source by **process**:

- `import { utils } from 'appbee'` holds stateless helpers that work in both the REST server and the worker (except `useToken`).
- Process state lives in `appState()` (REST server, after `serve()`) or `workerState()` (worker, inside the `work()` app function). In the worker, `appState()` is empty.

| Helper | From | Process | Purpose |
|---|---|---|---|
| `useBus()` | `appState()` / `workerState()` | both | pub/sub events + socket broadcast. See [bus.md](bus.md) |
| `useQueue(name)` | `appState()`¹ / `workerState()` | both | producer: `addJob`, `on` |
| `useWorker(name)` | `appState()`¹ / `workerState()` | both (normally worker) | consumer: `processJob`, `on` |
| `useRedis()` | `appState()`¹ / `workerState()` | both | the connected bus Redis client |
| `useJob(name)` | `workerState()` | worker | functions exported from `jobs/*.ts` |
| `utils.useDataLoader<T>()` | `utils` | both | batch-join related models into results. See [data-loader.md](data-loader.md) |
| `utils.useDataPager(opts)` | `utils` | both | walk a model in pages. See [data-pager.md](data-pager.md) |
| `utils.useCronMaster` | `utils` | worker (both possible) | schedule cron tasks. See [cron-master.md](cron-master.md) |
| `utils.usePlugin(name)` | `utils` | both | a plugin export |
| `utils.useConfig(key)` | `utils` | both | a loaded `config/<key>.ts` |
| `utils.useSource(key)` | `utils` | both | a store connection `{ db, storeType }` |
| `utils.useToken()` | `utils` | server | JWT sign/verify |
| `utils.useEncrypt()` | `utils` | both | scrypt password hash/verify |

¹ On the server, `useQueue`, `useWorker`, and `useRedis` exist only when `config/bus.ts` is set. Call them with `?.` or guard them, and type-narrow with `!` only where the bus is guaranteed.

## useQueue / useWorker / useJob

Queues are BullMQ, keyed by name and memoised per name. Queue instances `addJob(payload, tag?)` and worker instances `processJob(async job => result)`; both support `on('completed' | 'failed', handler)`. `useJob('name')` returns a named export of `jobs/*.ts`, typed by augmenting the global `Jobs` interface. The full contract, payload rules, and examples are in the `appbee-worker` skill.

## useRedis

```ts
const redis = appState().useRedis!();          // workerState().useRedis() in the worker
await redis.set(`resource:${token}`, JSON.stringify(payload), { EX: 2 * 3600 });
const raw = await redis.get(`resource:${token}`);
```

`useRedis()` returns the node-redis v4 client the bus connected at boot, so use the promise API. Prefix keys per feature (`resource:`, `captcha:`). Set the TTL in the same call with `{ EX }` rather than a separate `expire`. Use it for short-lived tokens, rate limits, and caches; models own persistent data.

## usePlugin

`utils.usePlugin('acl')` returns the export named `acl` from `plugins/`, typed by the global `Plugins` interface. Samples export factories, hence the trailing call: `utils.usePlugin('acl')().can(req)`. Writing plugins: see the `appbee-plugin` skill.

## useConfig

`utils.useConfig('application' | 'bus' | 'ldap' | 'policy' | 'security' | 'smtp' | 'store' | 'view')` returns that file's default export. It is the process-agnostic way to read settings:

```ts
const { mountRestOn = '' } = utils.useConfig('application');
const { spaces } = utils.useConfig('security');
```

The server also mirrors a subset into `appState().env`: `SECRET`, `security`, `UPLOAD_DIR`, `APP_NAME`, and more. The worker mirrors a smaller subset into `workerState().env`.

## useSource

`utils.useSource('core')` returns `{ db, storeType }` for a key in `config/store.ts`, or a tenant key in multi-tenant apps. Wrap it as a request to bind models outside an HTTP request:

```ts
const req = { source: utils.useSource('core') };
const Tasks = workerState().model.Tasks(req);
```

`db` is the raw driver handle (a MongoDB `Db` or a knex instance) for queries the model API can't express.

## useToken (server)

```ts
const { sign, verify } = await utils.useToken();
const token = await sign({ id, streamId, role, exp: Math.floor(Date.now() / 1000) + 7 * 86400 });
const payload = await verify(token);   // undefined when invalid or expired
```

- Both functions resolve to `undefined` instead of throwing. They sign with `security.secret` from `appState().env`, so they work only in the REST process; in a worker `sign` returns `undefined`.
- Tokens never expire unless the payload carries `exp`.
- appbee decodes `Authorization: Bearer <token>` (or the socket handshake `auth.token`) into `req.currentUser` on every request. A `tenant` claim selects the data source when `useMultiTenant` is on. Sign only what handlers and policies need (`id`, `role`, `tenant`), never secrets.

## useEncrypt

```ts
const { hash, verify } = await utils.useEncrypt();
const stored = await hash(plain);                 // "<hex>.<salt>", or undefined on failure
const ok = await verify(supplied, stored);        // false on mismatch or bad input
```

Hash before `create`/`update`, and keep the field in the model's `excludes`. Then read it back for verification with `find({ query, relaxExclude: true })`.
