---
name: appbee-use
description: Wires appbee's built-in route handlers from the `use` export — use.schema (request validation), use.captcha, use.unlink (delete uploads), and use.excelExport (download as .xls) — into controller verbs. Use when validating an appbee request body or params, or adding a captcha, file-delete, or Excel-export endpoint.
---

# appbee `use` handlers

`import { use } from 'appbee'`. Each member is a factory: call it to get a handler, then pass the handler to a controller verb (see the `appbee-controller` skill). `use.schema` is middleware that goes before other handlers; the other three are **owned** `(req, res)` handlers that send the response.

| Handler | Kind | Typical mount |
|---|---|---|
| `use.schema(schema)` | middleware `(req, res, next)` | before an owned handler or a delegated CRUD step |
| `use.captcha()` | owned | `get('/captcha', use.captcha())` |
| `use.unlink()` | owned | `post('/unlink', use.unlink())` |
| `use.excelExport()` | owned | `post('/export', use.excelExport())` |

Mount the owned three on a module without a model (for example `Route('Core', '/core')`), or as an owned `post`, so no model schema validation strips their input. Then add each path to `config/policy.ts`.

## use.schema(schema)

`use.schema(schema)` validates the **whole** `req.context` against a Valibot object.

- **On success** it **replaces** `req.context` with the parsed output and calls `next()`. Coercions apply (`x.objectId()` yields an `ObjectId`), defaults fill in, and unknown keys are dropped.
- **On failure** it answers `400 { error: 'data.title: Invalid length; params.id: ...' }`.

Describe the context shape for the method, not just the payload:

```ts
import { Route, use, v, x, type Infer } from 'appbee';

const uploadSchema = v.object({
  data: v.object({
    fileName: v.pipe(v.string(), v.nonEmpty()),
    visibility: v.pipe(v.string(), v.nonEmpty()),
    mediaType: v.picklist(['image', 'video', 'doc']),
    partSize: v.optional(v.number()),
  }),
  params: v.optional(v.any()),
  __client_time: v.optional(v.string()),
});
type UploadContext = Infer<typeof uploadSchema>;

post('/upload', use.schema(uploadSchema), async (req, res) => {
  const { data } = req.context as UploadContext;
  // ...
});
```

- **Name every key the next step reads.** Context keys you leave out (`query`, `__client_time`, `includes`) are gone afterwards.
- **Coerce MongoDB route ids** on delegated `put`/`patch`/`destroy`, which don't coerce params themselves:

  ```ts
  const byId = v.object({ params: v.object({ id: x.objectId() }), data: v.optional(v.any()) });
  get('/:id?').put('/:id', use.schema(byId)).destroy('/:id', use.schema(byId));
  ```

- **Reuse model schemas** to build request schemas. A create body is `v.omit(schema, ['id'])` and a patch body is `v.partial(schema)`.
- **Multipart fields are strings.** For uploads, use `v.string()` (or `v.pipe(v.string(), v.transform(Number))`) for fields that look numeric.
- On a delegated `post`, `use.schema` cannot be combined with `preCreate`. Use it on an owned `post`, or on `get`/`put`/`patch`/`destroy`.

## use.captcha()

The handler answers `200` with an [svg-captcha](https://www.npmjs.com/package/svg-captcha) object `{ text, data }`:

- `data` is a URI-encoded SVG. Render it with `decodeURIComponent(data)`.
- Settings are fixed: 4 characters, noise 3, and no `0 o 1 i l`.

The response includes `text`, the answer, so the built-in handler only supports client-side checks. For a server-verified captcha, write an owned handler instead:

1. Call `svgCaptcha.create(...)`.
2. Store `text` in Redis under a random token with an expiry (see `useRedis` in the `appbee-utils` skill).
3. Return only `{ token, data }`.
4. Verify `token` + answer in a policy.

## use.unlink()

The handler deletes uploaded files under `UPLOAD_DIR` (`<base>/<view.staticDir>/<application.uploadDir>`).

- **Body:** a top-level `files` field holding a comma-separated list of paths relative to `UPLOAD_DIR`: `{ "files": "avatars/a.png, docs/b.pdf" }`.
- **Responses:** `200 { text: 'Files successfully deleted' }`, or `400 { error: 'There are no files' }` when `files` is missing. Missing files are logged and skipped.
- **Methods:** it reads `req.context.files`, so it only works on `post`/`put`/`patch`. A DELETE puts querystring values under `query`, where the handler can't see them.
- **Security:** paths are not sanitised, so `../` escapes `UPLOAD_DIR`. Always gate it with an auth policy, and when clients aren't trusted, put a policy in front that rejects names containing `..`.

## use.excelExport()

The handler turns client-supplied content into an `.xls` download.

- **Body:** `{ "data": { "storeName": "orders", "content": "<table>…</table>" } }`. `content` is any spreadsheet-readable text, typically an HTML table or tab-separated rows built by the client.
- **Response:** `application/vnd.ms-excel` with `Content-Disposition: attachment; filename=<appName>_<storeName>.xls`. `content` is written as-is in binary encoding.

The handler doesn't query anything. To export server data, write an owned handler that runs `find` and renders the rows, then reuses the same headers.
