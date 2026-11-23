---
name: appbee-init
description: Scaffolds a new appbee app in src/bee — config, middlewares, modules, plugins, policies, and types — with default setup files and a src/app.ts serve() entry, after checking the project isn't already initialised. Use when starting, bootstrapping, or initialising an appbee project or backend.
---

# appbee init

An appbee app is a **base folder** passed to `serve(base)` (and `work(base)` for a worker). appbee boots only if the base contains both `config/` and `modules/`. This skill always puts the base at **`src/bee`** in the current working directory, with `src/app.ts` as the entry that serves it.

## Steps

1. **Refuse if already initialised.** Stop, change nothing, and report what you found if any of these hold in the current working directory:
   - `src/bee/` exists, even if it is empty or partial;
   - any folder holds both `config/` and `modules/`, outside `node_modules/`;
   - any source file imports `serve` or `work` from `appbee`.

   An `appbee` dependency in `package.json` alone doesn't count: the project may have installed it without scaffolding.
2. **Create the folders** under `src/bee/`: `config/`, `middlewares/`, `modules/`, `plugins/`, `policies/`, `types/`.
   - The folder is `middlewares/`, plural. appbee ignores a `middleware/` folder.
   - Add an empty `.gitkeep` to `plugins/` and `policies/`. The other folders get starter files in step 3.
3. **Write the setup files** from [templates.md](templates.md), verbatim except for the placeholders:
   - `src/bee/config/application.ts`, `store.ts`, `policy.ts`, `security.ts`, `view.ts`
   - `src/bee/middlewares/logger.ts`
   - `src/bee/modules/health/controller.ts`
   - `src/bee/types/extensions.d.ts`
   - `src/app.ts`

   Replace `<package-name>` with `package.json` `name`, or the folder name when there is no `package.json`.
4. **Wire the project.** Add what is missing; never overwrite existing values.
   - `package.json`:
     - Set `"type": "module"`.
     - Add the scripts `"dev": "NODE_ENV=development tsx watch src/app.ts"`, `"build": "tsc"`, and `"start": "node dist/app.js"`.
     - Dependencies: `appbee`, plus the store's driver (`mongodb`, or `knex` plus `pg`/`mysql2`/`sqlite3`).
     - Dev dependencies: `tsx`, `typescript`, `@types/node`.
   - `tsconfig.json`, if absent: `module` and `moduleResolution` `nodenext`, `target` `esnext`, `strict`, `skipLibCheck`, `esModuleInterop`, `rootDir` `src`, `outDir` `dist`, `include` `["src/**/*.ts"]`.
   - `.gitignore`: add `node_modules`, `dist`, and `.env`.
   - `.env.example`: list `APPBEE_SECRET`, `PORT`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`.
   - Print the install command for the project's package manager; run it only when the user asks.
5. **Done when:**
   - every path in steps 2 and 3 exists, and no file holds a literal password; the only literal secret is the commented default in `security.ts`;
   - with dependencies installed and the store reachable, `NODE_ENV=development npx tsx src/app.ts` boots and `GET /api/health` returns `{ "data": { "status": "ok" } }`. If any of those prerequisites is missing, name the missing one instead of claiming the boot works.

## Defaults and why

- **No `config/bus.ts`.** With a bus file, the server needs Redis at boot. Add one with the `appbee-worker` skill when queues or a worker are needed.
- **`store.ts` defines `core`.** Store-less models and every request read from `core`, or from the tenant key when `useMultiTenant` is on.
- **Policies deny by default** (`'*': false`), with only `/health` open. Each new route is opened on purpose in `config/policy.ts` (see the `appbee-controller` skill).
- **`security.ts` ships a default `secret`**, so the app boots immediately. `APPBEE_SECRET` overrides it. JWTs from `utils.useToken()` are signed with it, and anyone who knows the default can forge tokens. Keep its change-me comment, and in the final report tell the user to set a strong `APPBEE_SECRET` (for example `openssl rand -hex 32`) before deploying.
- **`types/extensions.d.ts` is a seed.** In development appbee regenerates it with the `Models` keys on every boot.
- **The `health` module exists because every `modules/` entry needs a `controller.ts`.** It has no model, so boot logs "No model was found for module:Health"; that warning is expected. Keep stray files such as `.DS_Store` out of `modules/`.

## Next

Add modules with the `appbee-model` and `appbee-controller` skills, shared services with `appbee-plugin`, and background work with `appbee-worker` (the worker reuses `src/bee` as its base).
