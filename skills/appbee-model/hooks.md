# appbee model hooks

Hooks are methods on the model object. `this` is the bound model, so declare them with method syntax.

## Write hooks: postCreate / postUpdate / postDestroy

```ts
async postCreate(req: RequestAware<PostContext<Article>>, data: AfterData<Article>) { ... }
```

- **When they run:** the model's `publishCreate/publishUpdate/publishDestroy(req, data)` calls them. Delegated controller verbs call publish automatically; owned handlers, plugins, and jobs must call it after their own writes, or the hook never fires.
- **`data`** is always an array of the written (or, for destroy, previously read) records.
- **`req`** is `{ source, io, context }`:
  - `source` binds other models to the same database: `appState().model.Users(req)`.
  - `context` is the validated request; `req.context.data` holds the client's input, including transient flags.
- **What they're for:** fan-out after a successful write, such as updating counters on related models, indexing into a search bucket, or enqueueing background work. Keep the HTTP response fast: fire-and-forget the side writes, and push slow or retryable work to a queue.

The sample pattern for enqueueing from a hook (`useQueue` exists only when `config/bus.ts` is set):

```ts
import { ARTICLE_CREATED } from '../../constants/queues.js';

async postCreate(req: RequestAware, data: AfterData<Article>) {
  const { model, useQueue } = appState();
  for (const { createdBy: streamId } of data) {
    model.Users(req).update({ query: { streamId }, $inc: { posts: 1 } });
  }
  const queue = useQueue?.(ARTICLE_CREATED);
  for (const { id, createdBy, visibility } of data.filter((a) => a.visibility !== 'public')) {
    queue?.addJob({ id: `${id}`, createdBy, visibility });
  }
},
```

A transient flag read from the request (`command` is in both `schema` and `transients`):

```ts
async postUpdate(req: RequestAware<PutContext<Service>>, data: AfterData<Service>) {
  const { command } = req.context.data;
  if (command !== 'start' && command !== 'stop') return;
  const queue = appState().useQueue?.('cronJobs');
  for (const svc of data) queue?.addJob({ ...svc, id: `${svc.id}`, command });
},
```

A worker that shares these model files has an empty `appState()`. If a hook can run there, resolve state from whichever process is live: `const { useQueue } = appState().useQueue ? appState() : workerState();`. Queue API: see the `appbee-worker` skill.

## Read hook: resolveResult

`resolveResult(data, includeMap)` runs on every `find` (including the re-read after create and update). It returns the enriched data, which may be one object or an array; keep that shape. It is where joins live, normally through `utils.useDataLoader`:

```ts
async resolveResult(input: ResolveData<Article>, includeMap: IncludeMap) {
  const { model } = appState();
  const extract = utils.useDataLoader<Article>();
  return extract({
    name: 'articles',
    input,
    pipeline: [{
      model: model.Users(this.aware!()),
      from: 'createdBy',
      to: 'streamId',
      includes: includeMap['users'] ?? 'title, fullName, avatar',
      map: 'id=>userId, title=>userTitle',
    }],
  });
},
```

`this.aware()` carries the source into the joined model. `includeMap['users'] ?? default` lets clients choose joined fields with `?includes=users:fullName`. DataLoader options and caveats: see `data-loader.md` in the `appbee-utils` skill.

## pipeline (MongoDB)

`pipeline()` returns aggregation stages inserted between `$match` and the paging facet: use it for `$lookup`/`$addFields` that must run in the database. Prefer `resolveResult` + DataLoader for cross-model joins.
