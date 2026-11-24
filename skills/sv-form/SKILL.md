---

## name: sv-form
description: Build Svelte 5 / SvelteKit forms with @steveesamson/microform (`uform`, `use:form`) as reusable components in `src/lib/modules/<module-name>/`. Use when the user asks for a form (create, edit, login, search, settings), form validation, or mentions microform, uform, `use:form`, or sv-form.

# sv-form

Create a form component backed by `@steveesamson/microform`. The form only collects and validates input. The page owns fetching and saving (see the `sv-page` skill).

## 1. Inspect first

- `package.json`: needs `svelte@^5` and `@steveesamson/microform`. If microform is missing, install it with the project's package manager.
- `svelte.config.js` `kit.alias`: use an existing alias such as `@modules` for imports. Otherwise use `$lib/modules/...`.
- `src/lib/modules/`: reuse the module folder if the feature already has one. Shared types usually live in `src/lib/modules/model-types.ts`.
- Field wrappers (e.g. `src/lib/core/form/input.svelte`): if the project has them, use them (see "Using project field wrappers"). If not, write raw fields. Do not invent a field library unless asked.

## 2. Where it goes

```
src/lib/modules/<module-name>/<thing>-form.svelte   # kebab-case; <module-name> = feature/domain (circle, users, memos)
```

One form per file. Name it for what it does: `circle-form.svelte`, `request-join-form.svelte`. A form shared by create and edit is one component that takes `defaultValues`.

## 3. Component contract

| Prop | Purpose |
|---|---|
| `defaultValues` | Initial data. Include `id` when editing. `{}` when creating. |
| `busy` | Parent is saving. Disables submit. |
| `onsubmit(data)` | Called with a copy of `values`, only when the form is valid. |
| `oncancel` | Optional. |

The form does not call APIs and does not navigate. It calls `onsubmit`, and the page calls `store.save(data)`.

## 4. Template

```svelte
<script lang="ts">
	import uForm, { type MicroFormReturn } from '@steveesamson/microform';
	import type { Params } from 'svesta';
	import type { Circle } from '../model-types.js';

	type Props = {
		defaultValues?: Partial<Circle>;
		busy?: boolean;
		onsubmit: (data: Params) => void | Promise<void>;
		oncancel?: () => void;
	};
	let { defaultValues = {}, busy = false, onsubmit, oncancel }: Props = $props();

	// $derived.by re-creates the form when defaultValues changes (edit pages).
	// For static defaults a plain `const { ... } = uForm({ data })` is fine.
	let { values, errors, form, submit, sanity, reset } = $derived.by<MicroFormReturn>(() =>
		uForm({ data: { ...defaultValues }, options: { validateEvent: 'input' } })
	);

	// Wrapper so `use:submit` always calls the latest `onsubmit` prop.
	const handle = (data: Params) => onsubmit(data);
</script>

<form use:submit={handle}>
   <label for="name">
     Name
     <input id="name" name="name" type="text" use:form={{ validations: ['required', 'maxlen:80'] }} />
     {#if errors.name}<small class="error">{errors.name}</small>{/if}
   </label>
   <label for="email">
     Email
     <input id="email" name="email" type="text" use:form={{ validations: ['required', 'email'] }} />
     {#if errors.email}<small class="error">{errors.email}</small>{/if}
   </label>
   <div class="buttons">
     {#if oncancel}<button type="button" onclick={oncancel}>Cancel</button>{/if}
     <button type="submit" disabled={!sanity.ok || busy}>
       {busy ? 'Saving…' : values.id ? 'Save changes' : 'Create'}
     </button>
    </div>
</form>
```

`uForm(...)` must run during component initialisation (top level of `<script>` or inside `$derived`). It registers `$effect`/`onMount` internally. Never call it from an event handler or a plain `.ts` file.

## 5. Field recipes

`use:form` reads the field name from `name` (or `data-name`, or the `name` option). It also sets the field's initial value from `defaultValues[name]`. You do not need `value={...}` or `bind:value`.

Rules by field type:

- **text/email/password/number/date/time/textarea**: `<input name="x" id="x" use:form={{ validations: [...] }} />`. Show `{errors.x}` next to it.
- **select**: add `value={values.x}` and use `validateEvent: 'change'`.
  ```svelte
  <select name="role" id="role" value={values.role} use:form={{ validateEvent: 'change', validations: ['required'] }}>
      <option value="">Select role</option>
      {#each roles as r (r.value)}<option value={r.value}>{r.label}</option>{/each}
  </select>
  ```
- **radio**: one `<input type="radio">` per item with the same `name`, `bind:group={values.x}`, and `use:form={{ validateEvent: 'change', validations }}`.
- **checkbox group**: same as radio with `type="checkbox"`. Microform writes the value as a comma-joined string (`"a,b"`), so split before submit:
  ```ts
  const toList = (v: unknown) => (Array.isArray(v) ? v : String(v ?? '').split(',').filter(Boolean));
  ```

  Give option values that are not substrings of each other.
- **file**: `validateEvent: 'change'`, `validations: ['required', 'file-size-mb:3']`. `values.x` only holds the input's string value (e.g. a fake path), not the `File`. Read the files from the `<input type="file">` node, or build `FormData` from the form element, when you upload.
- **contenteditable** (rich text): `<div contenteditable="true" use:form={{ name: 'story', html: true, validateEvent: 'input', validations: ['required'] }}></div>`. `html: false` stores plain text. Plain text is also stored as `values['story-text']`.
- **hidden**: `<input type="hidden" name="id" use:form />`. Hidden or not, `id` already comes through `defaultValues`, so you usually don't need a field for it.
- **match another field**: `validations: ['required', 'match:password']`, where `password` is the other field's `name`.

Validator list, custom validators and the full API are in [reference.md](reference.md).

## 6. Using project field wrappers

Where the project wraps fields (e.g. `Input`, `Select`, `Radio`, `RichText`, `FormButtons`), the usual call is:

```svelte
<Input {form} bind:values bind:errors name="name" label="Name" validations={['required']} />
```

Check the wrapper's prop list before using it. The wrappers pass `form`, `values` and `errors` down and render `errors[name]`. Match what neighbouring forms in the same module do.

## 7. Behaviour you must not "fix"

- `sanity.ok` is `false` until about 200 ms after mount (`fieldWaitTimeInMilliSecond`) and until a value changes. An untouched edit form therefore keeps its submit button disabled. Keep `disabled={!sanity.ok}`.
- `errors` only fill in on each field's validate event, but `sanity.ok` is computed from all registered validations. A disabled submit button can therefore appear before any error text does.
- Once a field's `use:form` has run, its validations stay registered even if the field leaves the DOM (microform 1.0.14). Do not conditionally render a `required` field and expect the form to become valid when it disappears. Use `disabled`/`hidden` styling instead, or drop `required`. If you must swap fields (as `circle-form` does with `values.id`), the hidden branch must not be required in the visible state.
- `submit` handlers get a shallow copy (`{ ...values }`). Strip UI-only keys before passing the data to a store.

## 8. Wiring to a page

Create page:

```svelte
<CircleForm busy={saving} onsubmit={onSubmit} oncancel={() => goto('/circles')} />
```

Edit page: pass the loaded entity, with `id`, as `defaultValues`. `store.save()` does `PUT` when `id` is present and `POST` otherwise. See `sv-page`.

## 9. Done checklist

- [ ] File is under `src/lib/modules/<module-name>/`.
- [ ] Every field has `name` (and `id` matching its `<label for>`).
- [ ] Every field with a rule has `use:form={{ validations: [...] }}`, and its error is rendered.
- [ ] Submit is `disabled={!sanity.ok || busy}` and the form uses `use:submit={handle}`.
- [ ] Props typed. No fetching or navigation in the form.
- [ ] Run the project's `check` script (`svelte-check`) and fix any errors in the new file.
