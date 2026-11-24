# microform reference

Package: `@steveesamson/microform` (Svelte 5). Verified against v1.0.14.

## Instantiation

```ts
import uForm, {
	type MicroFormReturn, type FieldProps, type Validator, type FormAction,
	type FormValues, type FormErrors, type FormSanity
} from '@steveesamson/microform';

const { values, errors, sanity, form, submit, onsubmit, reset } = uForm({
	data: { /* initial values */ },
	options: {
		validateEvent: 'blur',            // 'blur' | 'change' | 'input' | 'keyup' | 'keydown'. Default 'blur'.
		validators: {},                   // override/add validators (see below)
		fieldWaitTimeInMilliSecond: 200,  // init delay before sanity can be true
		debug: false
	}
});
```

| Return | Type | Use |
|---|---|---|
| `values` | reactive object | Current values. Writing `values.x = ...` updates validation. |
| `errors` | reactive object | `errors[name]` is a message or `''`. |
| `sanity` | `{ ok: boolean }` | Gate the submit button. |
| `form` | action | `use:form={{ validations, validateEvent, name, html }}` on each field. |
| `submit` | action | `<form use:submit={handler}>`. Calls `handler({ ...values })` if `sanity.ok`. |
| `onsubmit` | `(handler) => (e) => void` | For a non-submit button: `onclick={onsubmit(handler)}`. |
| `reset` | `() => void` | Restores `data`, clears errors, resets DOM nodes. `sanity.ok` becomes false. |

`form` options:

| Option | Default | Meaning |
|---|---|---|
| `validations` | `[]` | Array of `Validator` strings. |
| `validateEvent` | form-level option | Event that triggers validation for this field. |
| `name` | node `name` / `data-name` | Needed for contenteditable. |
| `html` | `false` | contenteditable: store `innerHTML` (true) or text (false). |

## Built-in validators

Each validator is `'key'` or `'key:arg'`. Every one has a default message. All except `required` pass on an empty value, so pair them with `required` when the field is mandatory.

| Validator | Checks |
|---|---|
| `required` | Non-empty (also empty arrays). Always runs first. |
| `email`, `url`, `ip` | Format. |
| `number`, `integer` | Numeric format. |
| `alpha`, `alphanum` | Letters (and spaces) only; alphanumeric starting with a letter. |
| `len:N`, `minlen:N`, `maxlen:N` | String length. |
| `min:N`, `max:N` | Numeric bound (`parseInt`). |
| `match:fieldName` | Equals `values[fieldName]`. |
| `file-size-mb:N` | Every selected file is at most N MB. |

Validations run in order and stop at the first error. `required` is moved to the front.

Exported key constants for overriding: `IS_REQUIRED, IS_EMAIL, IS_URL, IS_IP, IS_INTEGER, IS_NUMBER, IS_ALPHA, IS_ALPHANUM, IS_MIN_LEN, IS_MAX_LEN, IS_LEN, IS_MIN, IS_MAX, IS_MATCH_FOR, IS_MAX_FILE_SIZE`. Note that `IS_MATCH_FOR` and `IS_MAX_FILE_SIZE` are the exported aliases for the `match` and `file-size-mb` keys.

## Custom validators

A validator returns an error string, or `''` when valid. Overriding a built-in key (as below) is the supported path. The `Validator` and `validators` types are closed to the built-in keys, so a brand-new rule name needs casts on both the `validators` entry and the `validations` string. An unknown rule name makes validation for that field stop silently.

```ts
import uForm, { IS_LEN, type FieldProps } from '@steveesamson/microform';

uForm({
	options: {
		validators: {
			[IS_LEN]: ({ label, value, parts }: FieldProps) => {
				const n = parseInt(parts?.[1] ?? '', 10);
				if (!n) return `${label}: length validation requires a length.`;
				return value && value.length !== n ? `${label} must be exactly ${n} characters long.` : '';
			}
		}
	}
});
```

`FieldProps`: `{ name, value, label, node?, values, parts? }`. `label` is the humanised field name (`supper_time` → `Supper Time`). `parts` is the validation string split on `:`.

## Source-level facts worth knowing

- `values` for a field is initialised from the node's own value, else `data[name]`, else `''`.
- Checkbox groups: on `change` the action writes a comma-joined string to `values[name]`.
- contenteditable: also writes `values[`${name}-text`]` (trimmed plain text).
- `sanity.ok = !anyError && !anyUnfit && canWatch`. `unfit` is the same validations run against every registered field on every value change.
- `validationMap` entries are never removed when a field unmounts.
- The `form` action must run inside component initialisation context (it uses `onMount`).
