# svesta reference for pages

Package: `svesta` (Svelte 5). Verified against v1.0.4 source.

## Configure `Transport` once

`src/routes/+layout.ts`:

```ts
import { Transport, type Params, type TransportConfig } from 'svesta';
import { PUBLIC_BASE_URL } from '$env/static/public';

const config: TransportConfig = {
	BASE_URL: PUBLIC_BASE_URL,
	DEBUG: false,
	// Browser-side header hook. Server loads pass their own `beforeSend` per request.
	beforeSend: (headers: Params) => { /* e.g. headers['authorization'] = `Bearer ${token}` */ },
	// realTime: { transports: ['websocket'], auth: (cb) => cb({ token }) },   // optional socket.io
	// context: 'news',                                                          // extra named instance
};
Transport.configure(config);
```

- Without `BASE_URL`, svesta logs a warning and requests break unless you pass full URLs.
- `+page.server.ts` loads run only on the server and may not evaluate a universal `+layout.ts` first (for example a client-side navigation to a server-only load). If you use server loads, also call `Transport.configure(...)` from `hooks.server.ts`. Put the config in a shared module imported by both.
- Named contexts: `Transport.configure({ context: 'news', ... })`, then `Transport.instance({ fetch, context: 'news', beforeSend })`.

## Auth header helper

```ts
// e.g. src/lib/modules/utils/auth.ts
import type { Params } from 'svesta';
export const bearer = (user?: { token?: string }) => (headers: Params) => {
	if (user?.token && headers) headers['authorization'] = `Bearer ${user.token}`;
};
```

If the project already has one (`preSend`, `auth-from`), use that and don't add a second.

## `Transport`

`Transport.instance({ fetch, beforeSend, context? })` returns the shared transport for that context with `fetch`/`beforeSend` replaced. Methods: `get(url, params?)`, `post(url, body)`, `put`, `patch`, `delete`, `options`, `upload(url, FormData)`. Every method returns `Promise<{ data?, error?, message?, status, ...rest }>`:

- Success: the API's JSON body, plus `status`.
- HTTP error: `{ error: '<url> - <statusText>', status }`.
- Network failure: `{ error: 'Fetch error', status: 500 }`. Offline: `{ error: 'You seem to be offline :)', status: 404 }`.
- Request bodies are JSON with an added `__client_time`.

## `useStore<T>(resource, props?)`

`resource` is the REST resource name (`'memos'`, `'/memos'`, or `'~memos'` to reverse the order). Do not include the base URL.

```ts
type StoreProps<T> = {
	params?: Params;                // query for every request, and part of the cache key
	orderAndBy?: string;            // 'asc' | 'desc' | 'desc|id'
	initData?: StoreState<T>;       // pre-populate from a loader
	includes?: string;              // passed to the API as `includes`
	namespace?: string;             // realtime room, defaults to resource name
	transportContext?: string;
	resultTransformer?: (raw) => StoreResult<T>;  // applied to sync() responses only
	queryTransformer?: (query) => Params;         // reshape the outgoing query (e.g. drop `page`)
};
```

`store.result`: `{ data?, recordCount, pages, page, limit?, loading?, error? }`. It is reactive.

| Method | Effect |
|---|---|
| `sync(init?)` | Fetch page 1 (or load `init`). No-op if already in sync. |
| `next()`, `prev()`, `pageTo(n)` | Replace data with another page. |
| `more()` | Append the next page. Needs an initialised store. |
| `search(term)` (debounced), `filter(partial)` | Re-query from page 1. |
| `save(entity)` | `PUT /resource/:id` if `entity.id`, else `POST /resource`. Body is `{ data: entity }`. Returns `{ error, data, status, message }` and patches or adds in the store. |
| `destroy({ id })` | `DELETE /resource/:id`, removes from the store. |
| `get(path, params)`, `post(path, body)`, `upload(file)`, `despace(data)` | Scoped calls that do not touch the store. |
| `find(value, key='id')` | Look up in the store (syncs first if needed). |
| `add`, `patch`, `remove` | Local-only mutations. |
| `on('create'\|'update'\|'destroy'\|'refresh', handler)` | Realtime/store events. |

Behaviours that affect pages:

- Stores are cached in a module-level map keyed by resource, params, order and includes. A `useStore` call **with** `initData` replaces the cached store, and a call **without** it reuses the cached one.
- The API list response should return `{ data, recordCount, page, pages, limit }`. Otherwise transform it (see below).
- Requests from `sync/next/prev/more/search/filter` send `limit`, `offset`, `orderDirection`, optional `orderBy`, `includes`, `search`, and `page`, merged with `params`.

### Non-standard API shape

```ts
import type { Params, StoreResult } from 'svesta';

export const resultTransformer = (raw: Params): StoreResult<User> => {
	const { page, per_page: limit, total: recordCount, total_pages: pages, data } = raw;
	return { page, limit, recordCount, pages, data } as StoreResult<User>;
};
```

In the loader (`+page.ts` or `+page.server.ts`):

```ts
const { error, ...rest } = await api().get<User[]>('/users');
return { config: { initData: { ...resultTransformer(rest), error } as StoreState<User>, params: {} } };
```

In `+page.svelte`: `useStore<User>('users', { ...data.config, resultTransformer })`. The component passes the transformer because `config` has to stay serializable.

## `+page.server.ts` variant

```ts
import { redirect } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';
import { Transport, type Params, type StoreState } from 'svesta';
import type { Memo } from '$lib/modules/model-types';
import { bearer } from '$lib/modules/utils/auth';

export const load: PageServerLoad = async ({ fetch, locals, url }) => {
	const user = locals.currentUser;
	if (!user) redirect(303, `/login?redirect_to=${url.pathname}`);

	const api = () => Transport.instance({ fetch, beforeSend: bearer(user) });
	const params: Params = { streamId: user.streamId };
	const initData = (await api().get('/memos', params)) as unknown as StoreState<Memo>;
	return { config: { initData, params, orderAndBy: 'desc' } };
};
```

Server loads also get `cookies`, `locals`, `platform`, `request`. `parent()` still works. Form `actions` can only be exported from `+page.server.ts`.

## Other `Resource`/utility exports

- `<Resource store resolve />`: renders `resolve(store.result)` (a snippet).
- `<Offline>…</Offline>`: renders its children when the network is offline. `network.status.online` is the boolean.
- `useEvents<T>(key, default)`: shared reactive value across components (`.value`, `.error`, `.clear()`).
- `BlockLoader`, `Loader`: loading UI components.

## Svelte/SvelteKit rules this skill relies on

- Pages get data only through the `data` prop; fetching belongs in `load`.
- `+page.ts` can't export `actions` and must not use private env or DB. Use `+page.server.ts` for those.
- Loads re-run when `params`/`url.searchParams` they read change, when `parent()` re-ran, or on `invalidate`/`invalidateAll`.
- `redirect()`/`error()` throw; do not wrap them in `try/catch` without re-throwing.
