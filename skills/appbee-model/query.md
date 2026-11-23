# Calling an appbee model

Get a bound instance first:

- In a server: `appState().model.Users(req)`.
- In a worker: `workerState().model.Users({ source: utils.useSource('core') })`.
- Inside a model hook: `model.Users(req)`, or `model.Users(this.aware())` in `resolveResult`.

Every method resolves to `{ data?, error?, recordCount? }`; driver errors come back in `error`.

## find

```ts
const { data, error, recordCount } = await Articles.find<Article>({
  query: { visibility: 'public', 'likes>=': 10, tags: ['svelte', 'ts'] },
  includes: 'title, slug | users: fullName, avatar',
  offset: 0,
  limit: 20,
  orderBy: 'createdAt',
  orderDirection: 'DESC',
  search: 'svelte runes',
});
```

| Option | Meaning |
|---|---|
| `query` | Filters. Operators go in the key suffix (see below). An array value means IN. |
| `includes` | `1` for all fields, or a projection string (see below). |
| `offset` / `limit` | Paging. MongoDB defaults `limit` to 25; SQL has no default. |
| `orderBy` / `orderDirection` | Overrides the model default. |
| `search` | Space-separated words, each matched against `searchPath` fields. |
| `relaxExclude` | `true` keeps `excludes` fields, for example to verify a password. |
| `beeSkipCount` | `true` omits `recordCount`. |

**Result shape:**

- If `query` holds a `uniqueKeys` field with a scalar value, `data` is **one object** or `undefined`.
- Otherwise `data` is an array, plus `recordCount` (the total before paging).

### Operator keys

| Key | MongoDB | SQL |
|---|---|---|
| `'age>'`, `'age>='`, `'age<'`, `'age<='` | `$gt` / `$gte` / `$lt` / `$lte` | same comparison |
| `'status<>'`, `'status!='` | `$ne` | `!=` |
| `'id~': [...]` | `$nin` | `NOT IN` |
| `field: [...]` | `$in` | `IN` |

The same keys work over HTTP querystrings and in `utils.useDataPager` params (`{ status: 'scheduled', 'users<': 1 }`).

### `includes` syntax

Segments are separated by `|`. A segment without a `model:` prefix projects the model's own fields; a prefixed segment is passed to `resolveResult` as `includeMap['<model>']`, so clients can choose joined fields:

```
title, slug | users: fullName, avatar
```

On MongoDB, `_id` is always projected and returned as `id`.

## create

```ts
const { data, error } = await Users.create<UserData>({ data: { streamId, email, status: 'Active' } });
```

`data` may be an array for bulk insert. The result re-reads the inserted rows (`excludes` applied), so `data` holds the stored record or records with `id`.

## update

```ts
await Users.update({ query: { id }, data: { avatar } });                 // MongoDB: $set; SQL: SET
await Users.update({ query: { streamId }, $inc: { posts: 1 } });         // MongoDB operators
await Users.update({ query: { id }, $set: { subscription }, upsert: true });
await Circles.update({ query: { id }, $inc: { members: -1 }, includes: 'members' });
```

- On MongoDB, pass either `data` or operator keys: `$set`, `$inc`, `$unset`, `$setOnInsert`, `$currentDate`. Counters belong in `$inc`, never read-modify-write.
- A query with a scalar unique key updates one record; anything else updates many.
- The result re-reads with `query`, so `includes` trims what comes back. MongoDB returns `{ error }` when nothing was modified.

## destroy

```ts
const { data, error } = await Bucket.destroy({ query: { recordId } });
```

MongoDB returns the query as `data`. Delegated deletes read the records first, so `postDestroy` receives full rows.

## Firing hooks from your own writes

Direct method calls fire **no** hooks. After a write in owned code, call the publish method so `post*` hooks run (and connected sockets are notified when `req.io` exists):

```ts
const { error, data } = await Users.update({ query: { streamId }, $inc: { posts: 1 }, includes: 'posts' });
if (!error && data) Users.publishUpdate(req, data);
```
