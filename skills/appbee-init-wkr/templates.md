# appbee init worker templates

Paths are relative to the worker package root. `<package-name>` and `<server-bee>` are replaced during step 3.

## src/worker.ts

```ts
import path from 'node:path';
import { fileURLToPath } from 'node:url';
import { work, workerState, utils } from 'appbee';
import { runTask } from './jobs/task.js';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

work(path.join(__dirname, 'bee'), async () => {
	const { useWorker } = workerState();
	const req = { source: utils.useSource('core') };

	useWorker('taskScheduled').processJob(async (job) => {
		await runTask(req, job.data);
		return { ok: true };
	});
});
```

`taskScheduled` is one of the sample's queue names. Replace it with a name the server enqueues. When the copy contains `src/bee/constants/queues.ts`, import the constant from there.

## src/jobs/task.ts

The sample logs `Task activated` from `src/jobs/task.ts` and does not export it. This file exports `runTask` so `src/worker.ts` can call it. `src/jobs` sits outside the copied bee, so `clean` does not delete it.

```ts
import type { Params } from 'appbee';

export const runTask = async (req: Params, data: Params) => {
	console.log('Task activated', req, data);
};
```

## scripts/strip-bee.mjs

The sample's `replace` script: delete `policies/`, every controller, and `plugins/s3.ts`, then overwrite `config/policy.ts`. This also writes `config/bus.ts` when the server copy has none.

```js
import fs from 'node:fs';
import path from 'node:path';

const rm = (target) => fs.rmSync(target, { recursive: true, force: true });

const walkControllers = (dir) => {
	if (!fs.existsSync(dir)) return;
	for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
		const full = path.join(dir, entry.name);
		if (entry.isDirectory()) walkControllers(full);
		else if (entry.name === 'controller.ts' || entry.name === 'controller.js') rm(full);
	}
};

rm('src/bee/policies');
walkControllers('src/bee/modules');
rm('src/bee/plugins/s3.ts');

fs.mkdirSync('src/bee/config', { recursive: true });
fs.writeFileSync(
	'src/bee/config/policy.ts',
	`import type { PolicyConfig } from 'appbee';

const policies: PolicyConfig = {
	'*': true
};

export default policies;
`
);

if (!fs.existsSync('src/bee/config/bus.ts')) {
	fs.writeFileSync(
		'src/bee/config/bus.ts',
		`import type { RedisStoreConfig } from 'appbee';

const bus: RedisStoreConfig = {
	host: process.env.REDIS_HOST ?? '127.0.0.1',
	port: Number(process.env.REDIS_PORT ?? 6379),
	...(process.env.REDIS_PASSWORD ? { password: process.env.REDIS_PASSWORD } : {})
};

export default bus;
`
	);
}
```

## package.json

Write this file when the project has no `package.json`. When the file exists, add only the keys it lacks.

```json
{
	"name": "<package-name>",
	"version": "1.0.0",
	"private": true,
	"type": "module",
	"engines": {
		"node": "^20.19.0 || >=22.12.0"
	},
	"scripts": {
		"clean": "rimraf src/bee dist",
		"copy": "ncp \"<server-bee>\" src/bee",
		"replace": "node scripts/strip-bee.mjs",
		"predev": "npm run clean && npm run copy && npm run replace",
		"prebuild": "npm run clean && npm run copy && npm run replace",
		"dev": "NODE_ENV=development tsx watch src/worker.ts",
		"build": "tsc",
		"start": "node dist/worker.js",
		"lint": "eslint .",
		"format": "prettier --write ."
	},
	"dependencies": {
		"appbee": "^1.3.4",
		"mongodb": "^4.17.2"
	},
	"devDependencies": {
		"@eslint/js": "^9.39.2",
		"@types/node": "^25.3.0",
		"eslint": "^9.39.2",
		"eslint-config-prettier": "^10.1.8",
		"ncp": "^2.0.0",
		"prettier": "^3.7.4",
		"rimraf": "^6.1.2",
		"tsx": "^4.21.0",
		"typescript": "^5.9.3",
		"typescript-eslint": "^8.50.0"
	}
}
```

- `<server-bee>` is the server's `src/bee`. The sample copies `../stream-bee/src/bee`.
- `clean`, `copy`, `replace`, `predev`, and `prebuild` are the sample's pipeline. `clean` deletes `dist` instead of `stream-worker`.
- `appbee` and `mongodb` are added because the sample does not declare them and the entry and the copied store need them.
- `dev` is `"NODE_ENV=development tsx watch src/worker.ts"`, the `appbee-init` dev script with the worker entry. `tsx` is `^4.21.0`.

## tsconfig.json

Enabled options from the sample. `outDir` is `dist` and `rootDir` is `src`. The sample emits to `stream-worker` and sets no `rootDir`.

```json
{
	"compilerOptions": {
		"target": "es2021",
		"module": "nodenext",
		"moduleResolution": "nodenext",
		"rootDir": "src",
		"outDir": "dist",
		"strict": true,
		"esModuleInterop": true,
		"forceConsistentCasingInFileNames": true,
		"skipLibCheck": true
	},
	"include": ["src/**/*.ts", "src/bee/types/**/*.d.ts"],
	"exclude": ["node_modules", "dist"]
}
```

## eslint.config.js

The worker sample has no ESLint config. This is the server scaffold's flat config, with `src/bee/**` ignored because that tree is the generated copy.

```js
import eslint from '@eslint/js';
import eslintConfigPrettier from 'eslint-config-prettier';
import tseslint from 'typescript-eslint';

export default tseslint.config(
	{ ignores: ['dist/**', 'node_modules/**', 'src/bee/**'] },
	eslint.configs.recommended,
	tseslint.configs.recommended,
	eslintConfigPrettier,
	{
		rules: {
			'no-undef': 'off'
		}
	}
);
```

## prettier.config.js

Same options as the server scaffold, which come from the stream-bee `.prettierrc`: tabs, single quotes, no trailing commas, print width 100.

```js
/** @type {import('prettier').Config} */
const config = {
	useTabs: true,
	singleQuote: true,
	trailingComma: 'none',
	printWidth: 100
};

export default config;
```

## .prettierignore

```
# Package managers
package-lock.json
pnpm-lock.yaml
yarn.lock
bun.lock
bun.lockb

# Generated
dist
src/bee
```
