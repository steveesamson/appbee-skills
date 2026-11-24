---

## name: sv-page

description: Create SvelteKit pages (`src/routes/<route>/+page.svelte`) together with their data loaders (`+page.ts` or `+page.server.ts`) using svesta (`Transport`, `useStore`, `Resource`). Use when the user asks for a page, route, list/detail/create/edit screen, or a page loader, or mentions svesta, sv-page, or `+page.ts`.

# sv-page

A page is two files: `+page.svelte` (UI) and a loader (data). The loader fetches with svesta `Transport` and hands `+page.svelte` a ready-to-use `useStore` config. Forms come from `sv-form` components in `src/lib/modules/<module-name>/`.

API cheat sheet and extra recipes are in [reference.md](reference.md).

## 1. Inspect first

- `src/routes/+layout.ts`: it should call `Transport.configure({ BASE_URL, beforeSend, ... })` once. If it doesn't, add it (see reference.md).
- An existing auth helper (e.g. `preSend(currentUser)`) and how `currentUser` reaches pages (`await parent()`, `locals`). Reuse them.
- `src/routes/` grouping (e.g. `(authed)`) and neighbouring pages. Match their layout, SEO and styling conventions.
- `kit.alias` in `svelte.config.js` (`@modules`, `@core`...) and the `$lib/modules/<module>` that holds the feature's components and types.
- Ask the user only if the route path or the API resource is unclear.

## 2. Pick the page kind and the loader file

| Page kind | Loader returns | UI |
|---|---|---|
| **List** | `{ config }` (a `StoreProps` with `initData`) | `useStore(name, data.config)` + `Resource` |
| **Detail** | `{ value }` (one entity) or `error(404)` | plain props |
| **Create** | nothing (guard only) | `useStore(name)` + form from `sv-form` |
| **Edit** | `{ value }` (entity with `id`) | `useStore(name)` + form with `defaultValues={value}` |

Loader file:

- `+page.ts` **(default)**: public or token-bearing REST calls. Runs on the server for SSR and again in the browser on client navigation.
- `+page.server.ts`: use only if the load needs `locals`, cookies, private env (`$env/*/private`), must keep a token out of the browser, or the page needs form `actions`. Return serializable data only.

## 3. Always follow these loader rules

1. Destructure `fetch` from the load event and pass it: `Transport.instance({ fetch, beforeSend })`. Never use the global `fetch` or omit it.
2. `Transport.instance(...)` mutates one process-wide transport. Call it immediately before each request, not once per load. After an `await`, another request may have swapped `fetch`/`beforeSend`.
   ```ts
     const api = () => Transport.instance({ fetch, beforeSend: bearer(currentUser) });
     const a = await api().get('/a');
     const b = await api().get('/b');
   ```
3. Always pass `beforeSend`, including on public pages (use a no-op). Otherwise a token from an earlier request can stay on the shared transport.
4. `transport.get` returns `{ ...apiBody, status }` or `{ error, status }`. For list endpoints whose body already is `{ data, recordCount, page, pages, limit }`, that result is the `initData`.
5. GET params must be flat string/number values (they go through `URLSearchParams`).
6. Per-user lists must carry the user key in `params` (e.g. `streamId`). Stores are cached by resource, params, order and includes, in one module-level map that the server shares across requests.
7. Keep `config` serializable (no functions). Put `resultTransformer`/`queryTransformer` in `+page.svelte`, or register them globally with `useGlobalResultTransformer`/`useGlobalQueryTransformer`.
8. If the API response is not StoreState-shaped, transform it in the loader as well. `resultTransformer` is only applied by `sync()`, not to `initData`.
9. Use `error(status, message)` and `redirect(303, url)` from `@sveltejs/kit`. They throw themselves, so there is no need to `return` them.
10. Type with the generated `./$types` (`PageLoad`, `PageServerLoad`, `PageProps`).

## 4. Recipes

### List: `+page.ts`

```ts
import type { PageLoad } from './$types';
import { Transport, type Params, type StoreProps, type StoreState } from 'svesta';
import type { Memo } from '$lib/modules/model-types';
import { bearer } from '$lib/modules/utils/auth'; // project's auth helper; see reference.md

export const load: PageLoad = async ({ fetch, parent }) => {
	const { currentUser } = await parent();
	const api = () => Transport.instance({ fetch, beforeSend: bearer(currentUser) });

	const params: Params = { streamId: currentUser.streamId };
	const initData = (await api().get('/memos', params)) as unknown as StoreState<Memo>;

	const config: StoreProps<Memo> = { initData, params, orderAndBy: 'desc' };
	return { config };
};
```

`orderAndBy` is `'asc' | 'desc'` or `'desc|<field>'`. `includes` (e.g. `'title,slug|users:!'`) is passed through to the API. Use both only when the API understands them.

### List: `+page.svelte`

```svelte
<script lang="ts">
	import type { PageProps } from './$types';
	import { useStore, Resource, type StoreResult } from 'svesta';
	import type { Memo } from '$lib/modules/model-types';
	import MemoList from '$lib/modules/memos/memo-list.svelte';

	let { data }: PageProps = $props();

	// Re-derived whenever the loader re-runs, so fresh initData replaces the store.
	const memos = $derived(useStore<Memo>('memos', data.config));
</script>

<svelte:head>
	<title>Memos</title>
	<meta name="description" content="Manage memos." />
</svelte:head>

{#snippet resolve({ data: items, loading, error, page, pages }: StoreResult<Memo>)}
	{#if error}
            <p class="error">{error}</p>
        {/if}
	{#if items}
            <MemoList memos={items} />
        {/if}
	{#if loading}
             <p>Loading…</p>
        {/if}
	<nav>
	     <button onclick={memos.prev} disabled={page === 1}>Previous</button>
	     <button onclick={memos.next} disabled={page === pages}>Next</button>
	     <button onclick={memos.more} disabled={page === pages}>Load more</button>
	</nav>
{/snippet}

<Resource store={memos} {resolve} />
```

- `memos.more()` does nothing unless the store was initialised with data, which is why the loader passes `initData`.
- Other store calls: `search(term)`, `filter({...})`, `pageTo(n)`, `save`, `destroy`. See reference.md.

### Detail: `+page.ts`

```ts
import { error } from '@sveltejs/kit';
import type { PageLoad } from './$types';
import { Transport } from 'svesta';
import type { Article } from '$lib/modules/model-types';

export const load: PageLoad = async ({ params, fetch, parent }) => {
	const { currentUser } = await parent();
	const transport =  Transport.instance({ fetch, beforeSend: bearer(currentUser) });

	const { data: value, error: err, status } = await transport.get<Article>('/articles', { slug: params.slug });
	if (err) error(status, err);
	if (!value) error(404, 'Article not found.');
	return { value };
};
```

```svelte
<script lang="ts">
	import type { PageProps } from './$types';
	let { data }: PageProps = $props();
	const article = $derived(data.value);
</script>

<svelte:head>
    <title>{article.title}</title>
</svelte:head>
<article>
   {@html article.html}
</article>
```

### Create: `+page.svelte` (no loader, or a guard-only one)

```svelte
<script lang="ts">
	import { goto } from '$app/navigation';
	import { useStore, type Params } from 'svesta';
	import type { Circle } from '$lib/modules/model-types';
	import CircleForm from '$lib/modules/circle/circle-form.svelte';

	const circles = useStore<Circle>('circles');
	let saving = $state(false);
	let failure = $state('');

	const onSubmit = async (data: Params) => {
		if (saving) return;
		saving = true;
		failure = '';
		const { error } = await circles.save(data as Circle); // POST (no id)
		saving = false;
		if (error) return void (failure = error);
		await goto('/circles');
	};
</script>

<svelte:head>
    <title>Create circle</title>
</svelte:head>

<h1>Create circle</h1>
{#if failure}
    <p class="error">{failure}</p>
{/if}
<CircleForm busy={saving} onsubmit={onSubmit} oncancel={() => goto('/circles')} />
```

### Edit: loader returns the entity, `+page.svelte` passes it as `defaultValues`

Loader: same as Detail, with the lookup narrowed to what the user may edit (e.g. `{ id: params.id, createdBy: currentUser.streamId }`). Page:

```svelte
<script lang="ts">
	import type { PageProps } from './$types';
	import { goto } from '$app/navigation';
	import { useStore, type Params } from 'svesta';
	import type { Circle } from '$lib/modules/model-types';
	import CircleForm from '$lib/modules/circle/circle-form.svelte';

	let { data }: PageProps = $props();
	const circles = useStore<Circle>('circles');
	let saving = $state(false);

	const onSubmit = async (values: Params) => {
		if (saving) return;
		saving = true;
		const { error } = await circles.save({ ...values, id: data.value.id } as Circle); // PUT (has id)
		saving = false;
		if (!error) await goto('/circles');
	};
</script>

<CircleForm busy={saving} defaultValues={data.value} onsubmit={onSubmit} oncancel={() => goto('/circles')} />
```

### Guard (auth)

Put the guard in the route group's `+layout.ts` (or `hooks.server.ts` for real enforcement):

```ts
export const load: LayoutLoad = async ({ parent, url }) => {
	const { currentUser } = await parent();
	if (!currentUser) redirect(303, `/login?redirect_to=${url.pathname}`);
};
```

Layout loads don't re-run on every client navigation. Each guarded `+page.ts` should still `await parent()` and handle a missing user itself. The API must enforce authorization regardless.

## 5. `+page.svelte` conventions

- Props: `let { data }: PageProps = $props();` (Svelte 5 runes; no `export let`, no `$:`, no `on:click`).
- Read loader data from `data`. Do not fetch inside the component. The only exceptions are user-triggered store calls (`save`, `destroy`, `search`, `more`).
- Anything that depends on `data` must be `$derived`, so it updates when the loader re-runs.
- Use `page` from `$app/state` (not `$app/stores`) for URL or error info.
- Set `<svelte:head>` title and description on every page.
- Handle `error` from store calls (`save`/`destroy` return `{ error, message, data, status }`) and show it.

## 6. Done checklist

- [ ] Files live at `src/routes/<route>/+page.svelte` and `+page.ts` (or `+page.server.ts`).
- [ ] Loader destructures `fetch`, calls `api()` per request, always passes `beforeSend`, and uses `error()`/`redirect()` from `@sveltejs/kit`.
- [ ] List pages return `{ config }` with `initData` and user-scoped `params`; `config` is serializable.
- [ ] Forms come from `src/lib/modules/<module>/` (use `sv-form` to create them); the page does the saving.
- [ ] `Transport.configure` exists in `+layout.ts` (and in `hooks.server.ts` if you use `+page.server.ts` loads).
- [ ] Run the project's `check` script (`svelte-check`) and fix errors in the new files.
