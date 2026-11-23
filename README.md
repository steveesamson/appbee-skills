# appbee-skills

Agent skills for writing [appbee](https://www.npmjs.com/package/appbee) applications. Each skill is a `SKILL.md` an agent loads when the task matches its description, plus reference files the skill points to for the branches that need them.

The skills cover the surfaces an appbee app is built from: REST controllers, models and queries, plugins, the built-in `use` handlers, runtime helpers, and the BullMQ worker.

## Install

Install with the [Skills CLI](https://skills.sh/) (`npx skills`). It requires Node.js 22.20 or newer. Run the command from the project that should receive the skills. The CLI prompts for which agents to install into. Add `-g` to install at user level, for every project.

List what the repository contains without installing:

```bash
npx skills add steveesamson/appbee-skills --list
```

### The whole set

```bash
npx skills add steveesamson/appbee-skills
```

The prompt lets you pick any subset. `-y` skips that prompt and installs every skill in the repository:

```bash
npx skills add steveesamson/appbee-skills -y
```

`--all` is the same selection plus every detected agent: `--skill '*' --agent '*' -y`.

### One skill

Pass the skill's `name` from its `SKILL.md` frontmatter. The `@skill` form and `--skill` select the same skill:

```bash
npx skills add steveesamson/appbee-skills@appbee-controller
npx skills add steveesamson/appbee-skills --skill appbee-controller
```

`--skill` accepts more than one name:

```bash
npx skills add steveesamson/appbee-skills --skill appbee-model appbee-worker
```

| Skill | Install |
|---|---|
| `appbee-init` | `npx skills add steveesamson/appbee-skills@appbee-init` |
| `appbee-controller` | `npx skills add steveesamson/appbee-skills@appbee-controller` |
| `appbee-model` | `npx skills add steveesamson/appbee-skills@appbee-model` |
| `appbee-plugin` | `npx skills add steveesamson/appbee-skills@appbee-plugin` |
| `appbee-use` | `npx skills add steveesamson/appbee-skills@appbee-use` |
| `appbee-utils` | `npx skills add steveesamson/appbee-skills@appbee-utils` |
| `appbee-worker` | `npx skills add steveesamson/appbee-skills@appbee-worker` |

Each install is the skill directory, so reference files such as `hooks.md` and `query.md` come with the skill. Bring installed skills up to date with `npx skills update`.

## Layout

```
skills/
├── appbee-init/
│   ├── SKILL.md                    scaffold src/bee and the serve() entry
│   └── templates.md                default setup files
├── appbee-controller/SKILL.md      routes, verbs, policies, realtime
├── appbee-model/
│   ├── SKILL.md                    schema, store binding, model options
│   ├── hooks.md                    postCreate / postUpdate / postDestroy, resolveResult
│   └── query.md                    find, create, update, destroy
├── appbee-plugin/SKILL.md          plugins/<name>.ts and the Plugins interface
├── appbee-use/SKILL.md             use.schema, use.captcha, use.unlink, use.excelExport
├── appbee-utils/
│   ├── SKILL.md                    process state and utils.* helpers
│   ├── bus.md                      useBus
│   ├── cron-master.md              utils.useCronMaster
│   ├── data-loader.md              utils.useDataLoader
│   └── data-pager.md               utils.useDataPager
└── appbee-worker/SKILL.md          work(), queues, jobs
```

`SKILL.md` is the entry point. Its YAML frontmatter supplies `name` and `description`; the description is what decides when the skill is loaded. Sibling markdown files are reference, not separate skills. An agent reaches them only when the entry point points there.

## Skills

| Skill | Load it when | Entry |
|---|---|---|
| `appbee-init` | Starting a new appbee project in the current directory | [`skills/appbee-init/SKILL.md`](skills/appbee-init/SKILL.md) |
| `appbee-controller` | Adding or changing a route, endpoint, verb, or controller | [`skills/appbee-controller/SKILL.md`](skills/appbee-controller/SKILL.md) |
| `appbee-model` | Adding or changing a model, schema, collection, query, or write hook | [`skills/appbee-model/SKILL.md`](skills/appbee-model/SKILL.md) |
| `appbee-plugin` | Adding or changing a plugin, shared service, or third-party client wrapper | [`skills/appbee-plugin/SKILL.md`](skills/appbee-plugin/SKILL.md) |
| `appbee-use` | Validating a request, or adding captcha, file-delete, or Excel-export | [`skills/appbee-use/SKILL.md`](skills/appbee-use/SKILL.md) |
| `appbee-utils` | A handler, hook, plugin, policy, or job needs events, Redis, joins, paging, cron, config, data sources, JWTs, or password hashing | [`skills/appbee-utils/SKILL.md`](skills/appbee-utils/SKILL.md) |
| `appbee-worker` | Adding or changing a worker, queue, producer, consumer, or background job | [`skills/appbee-worker/SKILL.md`](skills/appbee-worker/SKILL.md) |

### appbee-init

Scaffolds a new app with the base folder always at `src/bee`, plus a `src/app.ts` entry that calls `serve(path.join(__dirname, 'bee'))`. The skill refuses, and changes nothing, if appbee is already initialised. That means `src/bee/` exists, some folder outside `node_modules` holds both `config/` and `modules/`, or a source file imports `serve` or `work` from `appbee`.

It creates `config/`, `middlewares/`, `modules/`, `plugins/`, `policies/`, and `types/`, then writes the setup files from [`templates.md`](skills/appbee-init/templates.md):

| File | Default |
|---|---|
| `config/application.ts` | App name from `package.json`, port `PORT` or 3000, REST mounted on `/api` |
| `config/store.ts` | A `core` MongoDB store configured from `DB_*` environment variables |
| `config/policy.ts` | Deny by default; only `GET /health` is open |
| `config/security.ts` | A default `secret`, overridden by `APPBEE_SECRET`, with comments to replace it with a strong value |
| `config/view.ts` | `public` static dir, `views` view dir |
| `middlewares/logger.ts` | Logs each request |
| `modules/health/controller.ts` | `GET /api/health` → `{ data: { status: 'ok' } }` |
| `types/extensions.d.ts` | Empty `Models`, `Jobs`, `Plugins` interfaces; regenerated in development |

It also adds missing project wiring without overwriting existing values:
- `package.json` from the template when the project has none, otherwise only the template keys it lacks. The defaults are `"type": "module"`, Node `^20.19.0 || >=22.12.0`, and the scripts `dev` (`tsx watch`), `prebuild` (`rimraf dist`), `build` (`tsc`), `start`, `test`, `test:run`, `lint` (`eslint .`), and `format` (`prettier --write .`). Dependencies are `appbee` and `mongodb@^4.17.2`. Dev dependencies add `eslint@^9.39.2`, `@eslint/js`, `eslint-config-prettier`, `prettier@^3.7.4`, and `typescript-eslint@^8.50.0` alongside `tsx`, `typescript@^5.9.3`, `@types/node`, `@types/express@^5`, `rimraf`, `vite@^7`, and `vitest@^4`.
- `tsconfig.json`, `vite.config.js`, `prettier.config.js`, and `eslint.config.js` from the templates, when any is absent. Both tsconfig and Vite alias `$lib` to `src`. `vite.config.js` is the Vitest config from the stream-bee sample. `prettier.config.js` carries that sample's `.prettierrc` options. `eslint.config.js` is a flat config for the Node server. `outDir` is `dist`.
- `.gitignore` and `.env.example`.

It prints the install command instead of running it. It doesn't create `config/bus.ts`, because a bus file makes the server require Redis at boot; add one with `appbee-worker` when you need queues.

### appbee-controller

A controller is `modules/<folder>/controller.ts`. `serve(base)` loads models first, then imports every `modules/*/controller.ts`. On the REST server every folder under `modules/` must contain a `controller.ts`; a missing controller or a stray file fails boot. Files load as `.ts` when `NODE_ENV=development` or `TEST=true`, otherwise as compiled `.js`.

`Route(module, mount, model?)` registers the resource. The folder name is the model key: each `_`-separated part is capitalised (`user_tasks` → `UserTasks`). The mount must start with `/` and can differ from the folder. A third argument borrows another module's model (`Route('Auth', '/auth', 'Users')`).

Verbs are chained. A **delegated** verb validates against the model schema and runs the CRUD call. An **owned** verb is a handler that sends the response. Delegated `post` distinguishes `preCreate` from an owned handler by arity: one parameter stamps fields onto `data` after validation; two parameters `(req, res)` owns the response.

Every new path is registered in `config/policy.ts`. Lookup order is `method[path]`, then `method['*']`, then `'*'`. `true` is `allowAll`, `false` is `denyAll`, and a string or array names files under `policies/`. The same routes are served over socket.io with the same policies.

### appbee-model

A model is the default export of `modules/<folder>/model.ts`. Both `serve()` and `work()` load every model at boot. The registry key matches the controller rule. `appState().model.Articles(req)` on the server, or `workerState().model.Articles(req)` in the worker, returns a fresh model bound to `req.source`.

The schema is Valibot (`v`) plus appbee extensions (`x`) and must include `id` (`x.objectId()` on MongoDB, `v.number()` or a string type on SQL). Delegated create requires every field that is not `v.optional`. Valibot objects drop unknown keys, so every client-writable field belongs in the schema, and every field that must never be stored belongs in `transients`.

Model options cover `store`, `collection`, `uniqueKeys`, `insertKey`, `excludes`, `transients`, `searchPath`, `includes`, sort defaults, `dbSchema`, a MongoDB `pipeline()`, and the lifecycle hooks. In development appbee regenerates `types/extensions.d.ts` with the `Models` keys; that file is generated output.

[`hooks.md`](skills/appbee-model/hooks.md) covers `postCreate`, `postUpdate`, and `postDestroy` (fired by `publish*`, with `data` always an array) and the read hook `resolveResult`. [`query.md`](skills/appbee-model/query.md) covers `find`, `create`, `update`, and `destroy`. Every call resolves to `{ data?, error?, recordCount? }` and never throws. Direct method calls fire no hooks; owned code calls `publishCreate`, `publishUpdate`, or `publishDestroy` after its own writes.

### appbee-plugin

A plugin is a named export of any file in `plugins/`. Both processes import every file there after models load and merge the exports into one map. Call sites are `utils.usePlugin('name')` (either process), `appState().plugin.name` (server), and `workerState().plugin.name` (worker). Each export is typed by augmenting the global `Plugins` interface in the same file.

Plugins initialise lazily. The files are imported before process state is filled, so config, models, and env are read inside the factory. Config is read with `utils.useConfig`, which works in both processes. Model access takes a `req` so the call can bind a source.

### appbee-use

`use` from `appbee` is a set of handler factories wired into controller verbs.

| Handler | Kind | Role |
|---|---|---|
| `use.schema(schema)` | middleware | Validates the whole `req.context`, replaces it with the parsed output, answers `400` on failure |
| `use.captcha()` | owned | Returns an svg-captcha `{ text, data }` |
| `use.unlink()` | owned | Deletes uploaded files listed in a top-level `files` field |
| `use.excelExport()` | owned | Turns client-supplied content into an `.xls` download |

`use.schema` drops keys the schema does not name, so the schema lists every key the next step reads. It also coerces MongoDB route ids on delegated `put`, `patch`, and `destroy`, which do not coerce params themselves.

### appbee-utils

Helpers come from two places. `utils` from `appbee` holds stateless helpers that work in both processes (`useToken` is server-only). Process state lives in `appState()` after `serve()`, or in `workerState()` inside the `work()` app function. In the worker, `appState()` is empty.

| Helper | Purpose | Reference |
|---|---|---|
| `useBus()` | Pub/sub events and socket broadcast | [`bus.md`](skills/appbee-utils/bus.md) |
| `useQueue(name)` / `useWorker(name)` | BullMQ producer and consumer | [`appbee-worker`](skills/appbee-worker/SKILL.md) |
| `useRedis()` | The bus Redis client (node-redis v4) | — |
| `useJob(name)` | A named export of `jobs/*.ts` (worker only) | [`appbee-worker`](skills/appbee-worker/SKILL.md) |
| `utils.useDataLoader` | Batch-join related models into result rows | [`data-loader.md`](skills/appbee-utils/data-loader.md) |
| `utils.useDataPager` | Walk a model one page at a time | [`data-pager.md`](skills/appbee-utils/data-pager.md) |
| `utils.useCronMaster` | Process-wide `node-cron` registry | [`cron-master.md`](skills/appbee-utils/cron-master.md) |
| `utils.usePlugin` / `useConfig` / `useSource` | Plugins, `config/<key>.ts`, store connections | — |
| `utils.useToken` / `useEncrypt` | JWT sign/verify, scrypt password hash/verify | — |

On the server, `useQueue`, `useWorker`, and `useRedis` exist only when `config/bus.ts` is set.

### appbee-worker

Background work is a BullMQ queue shared by a producer and a consumer. The producer (usually a model hook on the REST server) calls `useQueue(name).addJob(payload)`. The consumer is a separate `work()` process that calls `useWorker(name).processJob(fn)`. Both sides use the same Redis from `config/bus.ts` and the same queue name.

`work(base, app)` loads config, data sources, models, plugins, and `jobs/`. It does not load controllers, policies, or middlewares. `config/bus.ts` uses `host`, `port`, and `password`; `work()` refuses to start without it. Payloads are JSON-serialised, so ids are re-parsed before querying MongoDB. A processor throws on failure so the job is marked failed, and returns only JSON-serialisable values. Reusable functions live in `jobs/*.ts` and are reached with `useJob`, typed by the global `Jobs` interface.

## Conventions the skills share

These rules show up in more than one skill. The skill files remain the source of truth.

- **Model keys** are the folder name with each `_`-separated part capitalised.
- **Results never throw.** Model methods and fallible plugin methods resolve to `{ data?, error?, recordCount? }`. Callers branch on `error`.
- **Writes and hooks are separate.** `create`, `update`, and `destroy` do not run `post*` hooks. `publishCreate`, `publishUpdate`, and `publishDestroy` do, and they broadcast to sockets when `req.io` exists. Delegated controller verbs publish for you.
- **MongoDB ids.** Only delegated `get` coerces route params through the schema. Everywhere else, parse with `v.parse(x.objectId(), id)` before a query. A hex string does not match an `ObjectId` `_id`.
- **MongoDB pages.** A `find` without `limit` returns at most 25 rows. DataLoader joins hit the same cap.
- **Two processes.** Server code uses `appState()`. Worker code uses `workerState()`. Shared hooks and plugins resolve whichever is live, and read config with `utils.useConfig`.
- **Transient flags.** A field listed in both the schema and `transients` survives validation and is never stored. Hooks read it from `req.context.data`.


#### Init (`appbee-init`, `templates.md`)

`SKILL.md` fixes the base at `src/bee` and the entry at `src/app.ts`. Step 1 is a refusal check covering an existing `src/bee/`, any folder holding both `config/` and `modules/`, and any `serve`/`work` import from `appbee`. A bare `appbee` dependency doesn't count as initialised. The skill spells the folder `middlewares/`, plural, because appbee ignores `middleware/`.

It explains each default:
- No `bus.ts`, so the server doesn't need Redis.
- `core` is the store that store-less models and requests read from.
- Policies deny by default, so routes are opened on purpose.
- The default `secret` lets the app boot right away, but anyone who knows it can forge tokens, so the agent reminds the user to set `APPBEE_SECRET`.
- `extensions.d.ts` is a seed that appbee regenerates in development.
- The `health` module exists because every `modules/` entry needs a controller; its "No model was found" warning is expected.

The done check requires every path to exist, no literal password in any file, and, with dependencies installed and the store reachable, a dev boot where `GET /api/health` returns `{ "data": { "status": "ok" } }`.

`templates.md` holds the app files verbatim, with `<package-name>` as the only placeholder, plus `package.json`, `tsconfig.json`, `vite.config.js`, `prettier.config.js`, `eslint.config.js`, and `.prettierignore`. The app files are formatted with that Prettier config, and `eslint .` passes on them. `no-empty-object-type` is off for `src/bee/types/**/*.d.ts` because the seed `extensions.d.ts` declares empty `Models`, `Jobs`, and `Plugins` interfaces for declaration merging. The `package.json` versions follow the sample. `mongodb` stays on the 4.x major appbee is built against, `@types/express` matches the 5.x types appbee's declarations compile with, and `typescript` stays on 5.x because TypeScript 6 deprecates the `baseUrl` used for `$lib`. The tsconfig follows the stream-bee sample (`module` `node16`, `moduleResolution` `nodenext`, `$lib/*` → `src/*`) and emits to `dist`. `vite.config.js` is that sample's Vitest config. `@types/express` is a dev dependency because the sample's `types` array names `express` without installing the package, which makes `tsc` report TS2688.

These versions together pass `tsc`, `npm run build`, and a Vitest test that imports through `$lib`. npm 10.9.8 crashed while resolving peer dependencies for these versions, with `Cannot read properties of null (reading 'edgesOut')`; pnpm installed them cleanly.

#### Controllers (`appbee-controller`)

Defines a controller as a side-effect module at `modules/<folder>/controller.ts` whose `Route(...)` calls register on import. Spells out boot rules: every `modules/` entry needs a `controller.ts`, and a stray file such as `.DS_Store` fails boot. A model-only module still gets a minimal controller.

Specifies the `Route` arguments (resource name, mount, optional borrowed model) and the verb chain. Documents the delegated pipeline for each verb:

- `get` coerces query and params, then `find`. A `/:id?` route returns one object when `id` or any `uniqueKeys` field is present, otherwise a page.
- `post` validates `{ data, params?, __client_time? }` and `create`s. With `preCreate`, the returned fields merge over `data` and override client values. They are not validated again.
- `put` and `patch` validate a partial `data` plus params or query, then `update`.
- `destroy` validates params or query, finds, then destroys.
- `head` and `options` take handlers only.

Response shapes are `201 { data }` on create, `200 { data }` otherwise (plus `recordCount` on lists), `200 { error }` on store errors, and `400 { error }` on validation failure. Middleware that calls `next()` on `get`, `put`, `patch`, and `destroy` runs before the CRUD call; a handler that responds instead owns the verb.

Records the arity rule for `post`: exactly one parameter is `preCreate`, exactly two is an owned handler, and a zero- or three-argument function hangs the route. Owned `post` skips create-schema validation, so input validation is `use.schema` in front of the handler. Owned handlers get models with `appState().model.<Key>(req)` and call `publish*` after their own writes.

Documents `req.context` per method, `req.currentUser` from a Bearer JWT, and `req.files` for multipart uploads. Querystring operators live in the key (`?status<>=draft`, repeated keys for `IN`).

Policies run after context, session, and `middlewares/*.ts`. The skill gives the `config/policy.ts` shape, lookup order, `true` / `false` / named-policy mapping, and the boot failure when a named policy file is missing. Realtime is the same routes over socket.io: `socket.emit('get' | 'post', { path, data? }, cb)`, with writes broadcasting `{ verb, room, data }` on `comets` only when the request arrived over the socket.

Gotchas recorded in this file: two `Route()` calls on the same mount share one table and the first module name wins; `Route` resolves the model at import time and a wrong name only warns, so delegated verbs fail later at request time.

#### Models (`appbee-model`, `hooks.md`, `query.md`)

`SKILL.md` defines the model as the default export of `modules/<folder>/model.ts`, loaded by both `serve()` and `work()`, and used as a factory bound to `req.source`. It requires an `id` field, explains optional versus required for delegated create versus update, and shows the schema-plus-options object (or `defineModel` when the installed appbee exports it). The options table covers store binding, collection name, unique keys, insert key, excludes, transients, search path, projection includes, default sort, SQL `dbSchema`, and MongoDB `pipeline()`. Transient control flags are the pattern for request-only instructions such as `command`, `action`, and `forshare`: declare them in the schema and in `transients`.

`hooks.md` specifies method syntax so `this` stays bound to the model. Write hooks run from `publish*` with `data` as an array and `req` as `{ source, io, context }`. The file shows fan-out (counter `$inc`, enqueue via `useQueue`) and reading a transient `command` in `postUpdate`. Workers that share model files see an empty `appState()`, so a hook that can run in either process picks `appState()` or `workerState()` according to which has `useQueue`. `resolveResult` is the read hook on every `find`, including the re-read after create and update, and is where joins go through `utils.useDataLoader`. `pipeline()` is the MongoDB alternative for `$lookup` / `$addFields` that must run in the database.

`query.md` is the call reference for owned handlers, hooks, plugins, and jobs. It documents `find` options (`query`, `includes`, paging, sort, `search`, `relaxExclude`, `beeSkipCount`) and the result rule: a scalar unique key yields one object, anything else yields an array plus `recordCount`. Operator keys (`>`, `>=`, `<`, `<=`, `<>`, `!=`, `~` for not-in, array for in) work on MongoDB, SQL, HTTP querystrings, and `useDataPager` params. The `includes` string uses `|` to separate the model's own projection from `model: fields` segments that become `includeMap` for `resolveResult`. `create` accepts one record or an array and re-reads the insert. `update` takes `data` (MongoDB `$set`, SQL `SET`) or operator keys (`$inc`, `$unset`, `$setOnInsert`, `$currentDate`) plus `upsert`. `destroy` on MongoDB returns the query as `data`; delegated deletes read the rows first so `postDestroy` receives full records.

Gotchas recorded with the model skill: a MongoDB `update` that changes nothing returns `{ error: 'No, record was updated.' }`; overriding `publish*` replaces both the hook call and the realtime broadcast, so side effects belong in `post*` hooks.

#### Plugins (`appbee-plugin`)

Defines a plugin as a named export under `plugins/`, merged into one map and reached by name. The sample shape is a class singleton behind a factory (`notifier`, and the same shape for `acl`, `s3`, `bucket`), with `declare global { interface Plugins { ... } }` beside the export. Export names must be unique across `plugins/`; a later file silently overwrites an earlier one. Files do nothing at import time beyond declarations.

Rules recorded here: initialise lazily; read config with `utils.useConfig` because `appState().env` exists only on the server (the sample `webpush` plugin breaks when copied into the worker for this reason); take `req: RequestAware` to touch models; return `{ data?, error? }` from fallible methods; catch errors on fire-and-forget writes called from hooks. Every file in `plugins/` is imported by each process that has the folder, so process-specific dependencies stay out of the shared tree. The worker sample deletes `plugins/s3.ts` when it copies the server base. One file may export several functions (`addCron`, `initCron`); the skill prefers one factory per concern.

#### Built-in handlers (`appbee-use`)

Documents the four `use` factories and where they mount. The owned three (`captcha`, `unlink`, `excelExport`) go on a module without a model, or on an owned `post`, so model-schema validation does not strip their input, and each path is added to `config/policy.ts`.

`use.schema` validates the whole context, applies coercions and defaults, drops unknown keys, and answers `400` with a joined field-error string. The skill shows an upload schema, the `byId` schema that coerces `:id` on `put` and `destroy`, and building request schemas from the model with `v.omit(schema, ['id'])` and `v.partial(schema)`. Multipart text fields arrive as strings. `use.schema` cannot be combined with `preCreate` on a delegated `post`.

`use.captcha` returns `{ text, data }` from svg-captcha (4 characters, noise 3, ambiguous glyphs removed). `text` is the answer, so the built-in handler is a client-side check. The skill sketches a server-verified alternative: store the answer in Redis under a random token and verify it in a policy. `use.unlink` deletes paths relative to `UPLOAD_DIR` from a comma-separated `files` field on `post`, `put`, or `patch`. Paths are not sanitised, so `../` escapes the upload directory; the skill requires an auth policy and, for untrusted clients, a policy that rejects `..`. `use.excelExport` writes client-supplied `content` as an `.xls` attachment named from the app name and `storeName`. It does not query; exporting server data means an owned handler that runs `find` and reuses the same headers.

#### Runtime helpers (`appbee-utils` and its references)

`SKILL.md` is the index: which object each helper comes from, which process it runs in, and which sibling file has the contract. It includes short references for `useRedis` (promise API, key prefixes, `{ EX }` for TTL), `usePlugin`, `useConfig` (the eight config keys: `application`, `bus`, `ldap`, `policy`, `security`, `smtp`, `store`, `view`), `useSource` (`{ db, storeType }` wrapped as a request outside HTTP), `useToken` (sign and verify resolve to `undefined` instead of throwing; tokens expire only when the payload carries `exp`; a `tenant` claim selects the source when `useMultiTenant` is on), and `useEncrypt` (scrypt `"<hex>.<salt>"`, stored in an `excludes` field and read back with `relaxExclude: true`).

`bus.md` contrasts `DevBus` (no bus config, in-process listeners, `IO.emit` on this server) with `ProdBus` (Redis pub/sub, socket.io Redis emitter). Event names are global channels and should be namespaced. `ProdBus` payloads must be JSON. `broadcast` uses the same `{ verb, room, data }` shape as model writes. Unsubscribe on `ProdBus` drops the whole channel. The file records a known limitation on appbee ≤ 1.3.4 with node-redis v4: `ProdBus` duplicates the Redis client and never calls `connect()`, so `emit` / `on` / `once` fail with `ClientClosedError` and cross-process `broadcast` may not reach sockets. Until that is fixed in `tools/event-bus.ts` and `rest/server.ts`, cross-process signalling goes through queues.

`cron-master.md` documents `utils.useCronMaster` as an object, not a function: `add`, `start`, `stop`, `evict`, `stopAll`, `has`, `length`, `dateToCronExpression`, and `init`. Crons run in the worker so scaling the REST server does not multiply them. Re-adding a key replaces the task. The command-queue pattern updates a `Services` record with a transient `command` of `'start'` or `'stop'`, the server hook enqueues `cronJobs`, and the worker calls `add` or `stop`. `init()` is hard-wired to a model registered as `Service` (folder `service`); when the model is `Services`, the boot routine is written with `useDataPager` instead. Tasks run in-process, `crons/` files are not auto-loaded, and a task's return value is ignored.

`data-loader.md` documents the batch join used from `resolveResult`: one `find` per pipeline job for `{ [to]: values }`, then a merge. `from` values are parsed with the related schema, and a value that fails to parse throws. A job with no `includes`, or with `'!'`, is skipped. The input row wins name clashes, the related `id` is never merged unless mapped, jobs run in parallel so a later job cannot join on a field an earlier job added, and a MongoDB related `find` still caps at 25 rows.

`data-pager.md` documents walking a filter one page at a time for workers and startup. `onPage(rows, next)` continues only when `next()` is called; `next` is absent on the last page. `start()` resolves after the first page, and later pages run as `next()` is called. A `find` error calls `onPage([])` once. Mutating the filtered field while paging skips rows, because the matching set shrinks under the cursor.

#### Workers (`appbee-worker`)

Defines the producer/consumer split, the shared queue-name module (`constants/queues.ts` in the samples), and a `config/bus.ts` of `{ host, port, password }`. The `url` key is not honoured. The REST server exposes `useQueue`, `useWorker`, and `useRedis` only when that file exists.

The worker entry points `work()` at a base folder with the same `config/` and `modules/` as the server. The samples copy the server base into the worker and strip controllers and policies. The processor example parses the job id, calls `useJob('sendMail')`, marks the row sent, and broadcasts an `update` on `comets`. Development runs under a TypeScript runner so `.ts` files load; production runs the compiled `.js`.

`addJob(payload, tag?)` returns `Promise<Job>`. `tag` becomes `job.name` and defaults to the queue name. Completed jobs are removed immediately, the last 5000 failures are kept, and there are no retries or delay; delayed work is `useCronMaster`. `useQueue(name)` is memoised per name. `addJob` on a worker throws `You cannot add jobs to a worker`.

`processJob` starts one BullMQ worker. Returning marks the job completed; throwing marks it failed. `processJob` on a queue throws `You cannot process jobs on a non-worker`. Calling it twice for the same name starts a second worker on that queue. `on('completed' | 'failed')` listens from any process; the handler receives `{ jobId, returnvalue }` or `{ jobId, failedReason }`. The job record is already gone after completion, so listeners need everything in `returnvalue`, which may arrive as a JSON string.

`jobs/*.ts` named exports merge into `workerState().useJob`, typed by augmenting `Jobs`. The file closes by restating worker versus server state and pointing `useBus().broadcast` at the Redis-client limitation in `bus.md`.
