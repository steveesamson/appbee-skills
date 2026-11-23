# utils.useDataLoader

A batch join: for each pipeline job, collect the `from` values of every input row, run **one** `find` on the related model for `{ [to]: values }`, and merge each match into its row. Its home is a model's `resolveResult` (see `hooks.md` in the `appbee-model` skill), but it works on any rows.

```ts
const extract = utils.useDataLoader<UserCircle>();
const loaded = await extract({
  name: 'usercircles',                       // label for errors and debug logs
  input,                                     // one row or an array; the result keeps the same shape
  debug: false,
  pipeline: [
    {
      model: model.Circles(this.aware!()),   // bound related model
      from: 'circleId',                      // field on the input rows
      to: 'id',                              // field on the related model
      includes: includeMap['circles'] ?? 'name, members, type',
    },
    {
      model: model.Users(this.aware!()),
      from: 'streamId',
      to: 'streamId',
      includes: includeMap['users'] ?? 'fullName, avatar',
      map: 'id=>userId, fullName=>userName',
    },
  ],
});
```

| Job field | Meaning |
|---|---|
| `model` | The related model, bound with the same source (`this.aware()` in hooks, `req` elsewhere). |
| `from` / `to` | The join keys. Each `from` value is parsed with the related schema's `to` entry, so hex strings become `ObjectId`s for `x.objectId()` keys; a value that fails to parse **throws**. |
| `includes` | Comma-separated fields to merge, or `1` for all. A job with no `includes`, or with `'!'`, is **skipped**, so always give a default. |
| `map` | `'from=>to, ...'` renames merged fields. |

## Merge rules

- Only the selected `includes` fields are merged.
- The input row's own fields win on name clashes.
- The related record's `id` is never merged. Map it (`id=>userId`) if the client needs it.
- Rows without a match are left unchanged.
- Jobs run in parallel and each one queries the input rows as given, so job B cannot join on a field that job A adds.

## Gotchas

- On MongoDB the related `find` has no `limit`, so it returns at most 25 rows. A page with more than 25 distinct `from` values silently misses joins. Keep pages that need joins at 25 rows or fewer, or join in a MongoDB `pipeline()` `$lookup`.
- Passing `includeMap['<model>'] ?? default` lets clients request joined fields (`?includes=users:fullName,avatar`). Clients can also send `users:!` to skip a join they don't need.
