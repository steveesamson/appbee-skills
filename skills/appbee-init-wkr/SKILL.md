---
name: appbee-init-wkr
description: Scaffolds an appbee worker package in the current directory — src/worker.ts calling work(), a predev copy of the server src/bee with controllers and policies stripped, plus package.json, tsconfig.json, eslint.config.js, and prettier.config.js. Use when starting, bootstrapping, or initialising an appbee worker, background process, or stream-wkr style package.
---

# appbee init worker

A worker is its own package. `predev` and `prebuild` copy the server's `src/bee` into this package, then strip the files `work()` does not run. The entry is `src/worker.ts`, and it calls `work()` on that copy. This follows the stream-wkr sample: a sibling package whose `src/bee` is produced from the server, not written by hand.

## Steps

1. **Refuse if this directory is the wrong place.** Stop, change nothing, and report what you found when any of these hold:
   - `src/worker.ts` already exists, or any source file imports `work` from `appbee` (the worker is already scaffolded);
   - `src/app.ts` imports `serve` from `appbee`, or this directory is an appbee server (`src/bee/config` and `src/bee/modules` with a `serve()` entry). Scaffold the worker in a new sibling directory. Point `<server-bee>` at this server's `src/bee`.

   An `appbee` dependency in `package.json` alone doesn't count.
2. **Create** `src/jobs/` and `scripts/`. `src/bee/` is not source. `clean` deletes it, and `copy` recreates it.
3. **Write the files** from [templates.md](templates.md), verbatim except the placeholders:
   - `src/worker.ts`, `src/jobs/task.ts`, `scripts/strip-bee.mjs`
   - `package.json` when the project has none. When it exists, add each template key it lacks and leave existing values untouched.
   - `tsconfig.json`, `eslint.config.js`, `prettier.config.js`, and `.prettierignore`, when any is absent.

   Replace `<package-name>` with `package.json` `name`, or the folder name when there is no `package.json`. Replace `<server-bee>` with the relative path to the server's `src/bee`. The sample uses `../stream-bee/src/bee`.
4. **Wire the project.** Add what is missing; never overwrite existing values.
   - `.gitignore`: `node_modules`, `dist`, `src/bee`, `.env`. `src/bee` is the copy.
   - `.env.example`: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`. The copied `store.ts` reads the `DB_*` variables.
   - Print the install command for the project's package manager; run it only when the user asks.
5. **Done when:**
   - every path in step 3 exists; `package.json`, `tsconfig.json`, `eslint.config.js`, and `prettier.config.js` match the templates, with existing `package.json` values kept;
   - `npx eslint .` and `npx prettier --check .` both pass;
   - with `<server-bee>` present, dependencies installed, Redis reachable, and the store reachable, `npm run dev` copies the bee, strips it, and `work()` stays up. `work()` throws `I cannot find working config for bus` when `config/bus.ts` is missing, and `Unable to connect to bus` when Redis is down. Name whichever prerequisite is missing instead of claiming the boot works.

## Defaults and why

- **`src/bee` is a copy.** `clean` deletes it, `copy` runs `ncp` from `<server-bee>`, and `replace` runs `scripts/strip-bee.mjs`. `predev` and `prebuild` run all three, which is the sample's pipeline. The sample's `clean` also deletes an output folder named `stream-worker`; this scaffold emits to `dist`.
- **`strip-bee` removes what the worker must not keep.** It deletes `policies/`, every `modules/**/controller.ts`, and `plugins/s3.ts` (the sample drops that plugin because it imports `aws-sdk`). It rewrites `config/policy.ts` to `{ '*': true }`. `work()` still imports every file in `config/`, so the policy file has to stay valid after `policies/` is gone. `work()` loads models, plugins, and `src/bee/jobs`. It does not load controllers or middlewares.
- **Worker-owned jobs stay in `src/jobs/`.** That directory survives `clean`. `useJob` only reads `<base>/jobs`, which is `src/bee/jobs` and gets replaced on every copy, so the entry imports `src/jobs/task.ts` itself. Queue names and processors after this scaffold belong to the `appbee-worker` skill.
- **`config/bus.ts` is written only when the copy has none.** `work()` refuses to start when the bus config is empty. An app created with `appbee-init` has no `bus.ts`. The written file reads `REDIS_HOST`, `REDIS_PORT`, and `REDIS_PASSWORD`. A `bus.ts` that arrived with the copy is left as it is. The sample hardcodes `127.0.0.1:6379` and the password `foobar`.
- **`appbee` and `mongodb` are dependencies.** The sample's `package.json` lists `date-fns`, `node-cron`, `redis`, and `web-push`, and does not list `appbee` or the store driver. The entry imports `appbee`, and the copied store uses `mongodb`. After the first copy, install whatever the kept plugins import; the sample's four packages are there for its own plugins. A plugin whose import throws is logged and skipped.
- **`dev` is the `appbee-init` script, aimed at the worker entry:** `"dev": "NODE_ENV=development tsx watch src/worker.ts"`. `tsx` is a dev dependency at `^4.21.0`, the same range as the server scaffold. `NODE_ENV=development` makes appbee load `.ts` files.
- **`tsconfig.json` keeps the sample's enabled compiler options** (`target` `es2021`, `module` and `moduleResolution` `nodenext`, `strict`, `esModuleInterop`, `skipLibCheck`) and drops the commented catalog. `outDir` is `dist` and `rootDir` is `src`, so `start` runs `node dist/worker.js`. The sample's `build` is `tsc --build` without `composite`; this scaffold's `build` is `tsc`.
- **ESLint and Prettier match the server scaffold.** The worker sample ships neither file. `eslint.config.js` ignores `src/bee/**` because that tree is generated. `prettier.config.js` is tabs, single quotes, no trailing commas, and print width 100. `.prettierignore` ignores lockfiles, `dist`, and `src/bee`.

## Next

Point `useWorker` at the queue names the server enqueues, and share those names with `src/bee/constants/queues.ts` when the copy has that file. Adding a queue, a `processJob` body, or a `jobs/` export is the `appbee-worker` skill.
