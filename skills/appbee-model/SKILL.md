---
name: appbee-model
description: Writes appbee models at modules/<name>/model.ts — Valibot schema, store binding, keys, excludes/transients, and lifecycle hooks — and calls model find/create/update/destroy. Use when adding or changing an appbee model, schema, collection, query, or post-create/update/destroy hook.
---

# appbee model

A model is the default export of `<base>/modules/<folder>/model.ts`. Both `serve()` and `work()` load every model at boot. The registry key is the folder name with each `_`-separated part capitalised (`articles` → `Articles`, `user_tasks` → `UserTasks`). It is a factory: `appState().model.Articles(req)` (server) or `workerState().model.Articles(req)` (worker) returns a fresh model bound to `req.source`.

In development appbee regenerates `<base>/types/extensions.d.ts` with the `Models` keys. Treat that file as generated output; it is overwritten on each dev boot.

## Steps

1. **Write the schema** with `v` (Valibot) and `x` (appbee extensions) from `appbee`. It must include `id`:
   - MongoDB: `id: x.objectId()`, which converts hex strings to `ObjectId` and back to `id` on reads.
   - SQL: `id: v.number()`, or a string type for string keys.
2. **Mark optionality deliberately.**
   - Delegated create requires every field that isn't `v.optional(...)` (only `id` is relaxed). Update accepts any subset.
   - Valibot `object` **drops unknown keys**, so every field a client may write must be in the schema.
   - `v.optional(schema, default)` fills the default on create.
3. **Export the model.** If the installed appbee exports `defineModel` (check `node_modules/appbee/dist/index.d.ts`), use it. Otherwise use a typed const. Both produce the same typing:

   ```ts
   import { v, x, type Infer, type Model } from 'appbee';

   const schema = v.object({
     id: x.objectId(),
     title: v.pipe(v.string(), v.nonEmpty()),
     slug: v.pipe(v.string(), v.nonEmpty()),
     tags: v.array(v.string()),
     visibility: v.picklist(['public', 'private']),
     likes: v.optional(v.pipe(v.number(), v.integer()), 0),
     createdBy: v.optional(v.string()),
     createdAt: v.optional(v.string()),
   });
   export type Article = Infer<typeof schema>;

   const Articles: Model<typeof schema> = {   // or: export default defineModel<typeof schema>({ ... })
     schema,
     uniqueKeys: ['id', 'slug'],
     insertKey: 'id',
     transients: ['__client_time'],
     searchPath: ['title'],
     orderBy: 'id',
     orderDirection: 'DESC',
   };
   export default Articles;
   ```

4. **Export the types** (`Infer<typeof schema>`, and a context schema if controllers validate with `use.schema`). Controllers, plugins, and jobs import them from the model file.
5. **Add hooks** only when a write must cause side effects. See [hooks.md](hooks.md).
6. **Done when** the module folder also has a `controller.ts` (the REST server requires one per folder), every client-writable field is in the schema, and every field that must never be persisted is in `transients`.

## Options

| Option | Meaning |
|---|---|
| `schema` | Valibot object; drives delegated validation, read coercion, and DataLoader key parsing. |
| `store` | Key in `config/store.ts`. Omit it to use the request's source: `core`, or the tenant key when `useMultiTenant` is on. `config/store.ts` must define `core` for store-less models. |
| `collection` | Table or collection name; defaults to the lowercased folder name. |
| `uniqueKeys` | Fields that identify one record (`id` is always added). A query holding one of them with a scalar value acts on **one** record and returns an object, not a list. |
| `insertKey` | Primary key used to re-read inserts (`'id'`). |
| `excludes` | Fields stripped from every read (for example `password`) unless the call passes `relaxExclude: true`. |
| `transients` | Fields stripped before writes and queries. Use for request-only flags. |
| `searchPath` | Fields matched (case-insensitively, any word) by the `search` option. |
| `includes` | Fields always added to the model's own projection. |
| `orderBy` / `orderDirection` | Default sort. |
| `dbSchema` | SQL schema name. |
| `pipeline()` | MongoDB only: aggregation stages inserted after `$match`. |
| `resolveResult`, `postCreate`, `postUpdate`, `postDestroy` | Hooks; see [hooks.md](hooks.md). |

### Transient control flags

To let a write carry an instruction that isn't data, declare the flag in the schema (so validation keeps it) **and** in `transients` (so it is never stored). Then read it in a hook from `req.context.data`. The samples use this for `Services.command` (`'start' | 'stop'`), `Tasks.action`, and `Articles.forshare`.

## Calling a model

Read [query.md](query.md) when writing `find`/`create`/`update`/`destroy` calls by hand: in owned handlers, hooks, plugins, or jobs. It covers operator keys, `includes` syntax, MongoDB update operators, and result shapes. The core rule: every call resolves to `{ data?, error?, recordCount? }` and never throws, so branch on `error`.

## Gotchas

- Only `get` coerces strings through the schema. In hooks, jobs, and owned handlers, convert ids yourself before querying MongoDB: `v.parse(x.objectId(), id)`. A hex string never matches an `ObjectId` `_id`.
- A MongoDB `find` without `limit` returns at most 25 rows.
- A MongoDB `update` that changes nothing returns `{ error: 'No, record was updated.' }`. Treat that as a no-op where it's expected.
- Overriding `publishCreate`/`publishUpdate`/`publishDestroy` (the sample `Tasks` model does) replaces both the `post*` hook call and the realtime broadcast. Put side effects in `post*` hooks instead.
- Hooks are methods called with `this` bound to the model, so write them as `async postCreate(req, data) {}` rather than as arrow functions.
