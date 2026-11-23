---
name: appbee-controller
description: Writes appbee REST controllers at modules/<name>/controller.ts — Route mounts, delegated CRUD verbs, owned handlers, preCreate, and their policy entries. Use when adding or changing an appbee route, endpoint, verb, or controller.
---

# appbee controller

A controller is a side-effect module at `<base>/modules/<folder>/controller.ts`. `serve(base)` loads models first, then imports every `modules/*/controller.ts`; each `Route(...)` call registers its routes on import. On the REST server **every** entry under `modules/` is treated as a module folder that must contain a `controller.ts`. A missing controller, or a stray file such as `.DS_Store`, fails boot, so a model-only module still gets a minimal controller.

Files load as `.ts` when `NODE_ENV=development` (or `TEST=true`), else `.js` from the build.

## Steps

1. **Name the module.** The folder name decides the model key: each `_`-separated part is capitalised (`usertasks` → `Usertasks`, `user_tasks` → `UserTasks`).
2. **Mount it.** `const { get } = Route('Articles', '/articles');`
   - Argument 1, the module, is the resource name. It resolves the model `capitalize(module)`.
   - Argument 2, the mount, must start with `/`. It can differ from the folder name (`Route('Outmails', '/mails')`).
   - Argument 3 (optional) borrows another module's model: `Route('Auth', '/auth', 'Users')`.
   - With no model, appbee warns and every verb needs an owned handler.
3. **Chain the verbs.** Each verb is either **delegated** (appbee validates against the model schema and runs the CRUD call) or **owned** (your handler sends the response). Verbs return the chain: `get('/:id?').post('/', preCreate).put('/:id').destroy('/:id')`.
4. **Register policies** for every new path in `config/policy.ts` (see [Policies](#policies)).
5. **Done when** every route path is either listed in `config/policy.ts` or deliberately inherits a method default. Owned handlers must send exactly one response on every branch, and each owned `post` handler must take exactly `(req, res)`.

## Delegated verbs

When the module has a model, each verb builds this pipeline. Extra handlers sit **between** validation and the CRUD call:

| Verb | Pipeline | Delegated call |
|---|---|---|
| `get(path, ...h)` | coerce query/params by schema → `...h` → find | `find({ query: {...query, ...params}, includes, limit, offset, orderBy, orderDirection, search })` |
| `post(path)` | validate `{ data, params?, __client_time? }` (all schema fields required except `id`) → create | `create({ data })` |
| `post(path, preCreate)` | same, with `preCreate(req)`'s return merged over `data` | `create({ data: {...data, ...injected} })` |
| `put` / `patch(path, ...h)` | validate `{ data: partial, params?, query? }` (params or query required) → `...h` → update | `update({ query: {...query, ...params}, data })` |
| `destroy(path, ...h)` | validate `{ params?, query? }` → `...h` → find, then destroy | `destroy({ query })` |
| `head` / `options` | handlers only | none |

Delegated verbs answer `201 { data }` on create, `200 { data }` (plus `recordCount` on lists) otherwise, and `200 { error }` on store errors. Validation failures answer `400 { error: "data.title: Invalid length; ..." }`.

A `get` on `/:id?` returns one object when the id (or any `uniqueKeys` field) is present, otherwise a page (MongoDB default `limit` is 25).

### Middleware before a delegated verb

On `get`, `put`, `patch`, and `destroy`, a `(req, res, next)` handler that calls `next()` runs before the CRUD call. Use it to scope or reshape the request, then let appbee finish:

```ts
get('/:id?', (req: Request, res: Response, next: NextFunction) => {
  (req.context as GetContext).query.createdBy = req.currentUser?.streamId;
  next();
});
```

A handler that responds instead of calling `next()` makes the verb owned; the CRUD call never runs.

### Coerce MongoDB ids on put/patch/destroy

`get` coerces `:id` through the schema (`x.objectId()` turns the string into an `ObjectId`). `put`, `patch`, and `destroy` do **not** coerce route params, so a string `id` never matches a MongoDB `_id`. Coerce with `use.schema`:

```ts
const byId = v.object({ params: v.object({ id: x.objectId() }), data: v.optional(v.any()) });

get('/:id?')
  .post('/', preCreate)
  .put('/:id', use.schema(byId))
  .destroy('/:id', use.schema(byId));
```

`use.schema` replaces `req.context` with the parsed output, which drops keys the schema doesn't name. Name every key the next step reads, such as `data` above.

## preCreate

`preCreate(req)` returns fields that are stamped onto `data` and override client values. It runs after validation, so the fields it returns are not validated again. It is the idiomatic way to set ownership, timestamps, and counters:

```ts
const preCreate = (req: Request) => {
  const { streamId } = req.currentUser as { streamId: string };
  const { __client_time } = req.context;
  return { createdAt: __client_time, createdBy: streamId, likes: 0 };
};
```

`post` tells a preCreate from an owned handler by **arity**: exactly one parameter means preCreate, exactly two means owned handler. Declare `(req: Request)` even when it's unused, because `() => ({...})` is neither and the route hangs. Pass the preCreate as the only extra argument; mixing it with handlers throws at boot.

## Owned handlers

Own the verb when the response isn't a single CRUD call: auth, uploads, aggregates, or calls to plugins.

```ts
import { Route, appState, utils, use, v, StatusCodes, type Request, type Response, type PostContext } from 'appbee';
import { schema, type UserData } from '../users/model.js';

const { post } = Route('Auth', '/auth', 'Users');

const signup = v.object({
  data: v.omit(schema, ['id']),
  params: v.optional(v.any()),
  __client_time: v.optional(v.string()),
});

post('/signup', use.schema(signup), async (req: Request, res: Response) => {
  const { data: input, __client_time } = req.context as PostContext<UserData>;
  const Users = appState().model.Users(req);
  const { hash } = await utils.useEncrypt();
  const { error, data } = await Users.create({ data: { ...input, password: await hash(input.password), joinedAt: __client_time } });
  if (error) return res.status(StatusCodes.INTERNAL_SERVER_ERROR).json({ error: 'Create error' });
  Users.publishCreate(req, data);
  res.status(StatusCodes.CREATED).json({ data });
});
```

- An owned `post` skips create-schema validation. Add `use.schema(...)` before the handler when the input needs validation.
- An owned `put`/`patch` on a model module still runs update-schema validation first, so the body must be `{ data: {...} }` with params or query.
- Get models with `appState().model.<Key>(req)`. Passing `req` binds the tenant's data source.
- Call `Model.publishCreate/publishUpdate/publishDestroy(req, data)` after a write. This fires the model's `post*` hooks and the realtime broadcast; delegated verbs do it for you.
- Respond `{ data }` or `{ error }` with `StatusCodes` from `appbee`, and `return` after every early response.
- For model query options and result shapes, see the `appbee-model` skill ([query.md](../appbee-model/query.md)).

## Request context

`restRouter` builds `req.context` before policies run:

| Method | `req.context` |
|---|---|
| GET | `{ query, params, includes, limit, offset, orderBy, orderDirection, search, __client_time }`; every other querystring key goes into `query` |
| POST | `{ ...body, params }`; clients send `{ data: {...}, __client_time? }` |
| PUT/PATCH | `{ ...body, params, query }` |
| DELETE | `{ params, query, __client_time }` |

Cast with `GetContext<T>`, `PostContext<T>`, `PutContext<T>`, or `DeleteContext<T>`. Other request fields:

- `req.currentUser` holds the decoded payload of an `Authorization: Bearer <jwt>` token signed by `utils.useToken()`.
- `req.files` holds multipart uploads (`MultiPartFile` with `path`, `filename`, `ext`, `mimetype`, `renameTo(dir, name)`). Multipart text fields arrive as strings under `req.context.data`.

Querystring operators go in the key: `?status<>=draft`, `?id=a&id=b` (array → IN). See [query.md](../appbee-model/query.md).

## Policies

Per route, the pipeline runs in this order: context and session → every `middlewares/*.ts` → policies → the verb pipeline. Policies come from `config/policy.ts`. Each key is the **full route path** (mount + path, without `mountRestOn`), written exactly as registered:

```ts
const policies: PolicyConfig = {
  '*': false,                      // global fallback; omitted means deny
  post: { '*': ['access'], '/auth/signin': true, '/auth/signup': ['signup'] },
  get:  { '*': ['access'], '/articles/:id?': ['public_or_authed'] },
  put:  { '*': ['access', 'updated_by'] },
  delete: { '*': ['access'], '/articles/:id': false },
};
```

- Lookup order is `method[path]`, then `method['*']`, then `'*'`.
- `true` maps to `allowAll` and `false` to `denyAll`. A string `'a, b'` or an array names files under `policies/`.
- A named policy file that doesn't exist fails boot.
- A policy file default-exports `(req, res, next)`. It gates the request or mutates `req.context` (for example, stamping `data.updatedBy`) and then calls `next()`.

## Realtime

Every route is also served over socket.io with the same policies. A client calls `socket.emit('get', { path: '/articles?limit=5' }, cb)` or `socket.emit('post', { path: '/articles', data: body }, cb)`. Writes broadcast `{ verb, room: '<model>', data }` on the `comets` event only when the request came over the socket.

## Gotchas

- `post('/x', (req, res, next) => ...)` alone has arity 3, so it registers no handler and the request hangs. Owned post handlers take exactly `(req, res)`.
- Two `Route()` calls with the same mount share one route table, and the module named first wins the resource name.
- `Route` resolves the model at import time. A wrong module or third argument only logs a warning, and delegated verbs then fail at request time.
