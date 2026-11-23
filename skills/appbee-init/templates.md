# appbee init templates

Paths are relative to the project root. `<package-name>` is replaced during step 3.

## src/bee/config/application.ts

```ts
import type { AppConfig } from 'appbee';

const application: AppConfig = {
	appName: '<package-name>',
	host: '127.0.0.1',
	port: Number(process.env.PORT ?? 3000),
	spa: false,
	useMultiTenant: false,
	ioTransport: ['websocket', 'polling'],
	mountRestOn: '/api',
	uploadDir: 'uploads',
	templateDir: 'template'
};

export default application;
```

## src/bee/config/store.ts

```ts
import type { StoreConfigMap } from 'appbee';

const store: StoreConfigMap = {
	core: {
		type: 'mongodb',
		host: process.env.DB_HOST ?? '127.0.0.1',
		port: Number(process.env.DB_PORT ?? 27017),
		database: process.env.DB_NAME ?? '<package-name>',
		user: process.env.DB_USER,
		password: process.env.DB_PASSWORD
	}
};

export default store;
```

For SQL, set `type` to `pg`, `mysql2`, `mssql`, `oracledb`, or `sqlite3`, or pass `connectionString`.

## src/bee/config/policy.ts

```ts
import type { PolicyConfig } from 'appbee';

const policies: PolicyConfig = {
	'*': false,
	get: {
		'/health': true
	}
};

export default policies;
```

## src/bee/config/security.ts

```ts
import type { Params } from 'appbee';

const security: Params = {
	// Signs and verifies every JWT issued by utils.useToken().
	// CHANGE THIS before deploying: replace the default with a long random value,
	// e.g. `openssl rand -hex 32`, and supply it through APPBEE_SECRET rather than committing it.
	secret: process.env.APPBEE_SECRET ?? 'appbee-default-secret-change-me'
};

export default security;
```

## src/bee/config/view.ts

```ts
import type { ViewConfig } from 'appbee';

const view: ViewConfig = {
	staticDir: 'public',
	viewDir: 'views',
	indexFile: 'index'
};

export default view;
```

## src/bee/middlewares/logger.ts

```ts
import type { Request, Response, NextFunction } from 'appbee';

const logger = (req: Request, res: Response, next: NextFunction) => {
	console.log(`${new Date().toISOString()} ${req.method} ${req.url}`);
	next();
};

export default logger;
```

## src/bee/modules/health/controller.ts

```ts
import { Route, StatusCodes, type Request, type Response } from 'appbee';

const { get } = Route('Health', '/health');

get('/', (req: Request, res: Response) => {
	res.status(StatusCodes.OK).json({ data: { status: 'ok' } });
});
```

## src/bee/types/extensions.d.ts

```ts
declare global {
	export interface Models {}
	export interface Jobs {}
	export interface Plugins {}
}

export {};
```

## src/app.ts

```ts
import path from 'node:path';
import { fileURLToPath } from 'node:url';
import { serve } from 'appbee';

const __dirname = path.dirname(fileURLToPath(import.meta.url));

serve(path.join(__dirname, 'bee'));
```

## package.json

Write this file when the project has no `package.json`. When the file exists, add only the keys it lacks: `type`, `engines`, each script, and each dependency.

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
		"dev": "NODE_ENV=development tsx watch src/app.ts",
		"prebuild": "rimraf dist",
		"build": "tsc",
		"start": "node dist/app.js",
		"test": "vitest",
		"test:run": "vitest run",
		"lint": "eslint .",
		"format": "prettier --write ."
	},
	"dependencies": {
		"appbee": "^1.3.4",
		"mongodb": "^4.17.2"
	},
	"devDependencies": {
		"@eslint/js": "^9.39.2",
		"@types/express": "^5.0.0",
		"@types/node": "^25.3.0",
		"eslint": "^9.39.2",
		"eslint-config-prettier": "^10.1.8",
		"prettier": "^3.7.4",
		"rimraf": "^6.1.2",
		"tsx": "^4.21.0",
		"typescript": "^5.9.3",
		"typescript-eslint": "^8.50.0",
		"vite": "^7.3.0",
		"vitest": "^4.0.16"
	}
}
```

- The `engines` range is the Node.js versions that Vite 7, Vitest 4, and rimraf 6 all support.
- `mongodb` stays on `^4.17.2`, the major appbee is built and tested against.
- `@types/express` is `^5.0.0` because appbee's own declarations compile against it, and they import their request types from `express`.
- `typescript` stays on `^5.9.3` because TypeScript 6 deprecates `baseUrl`, which `tsconfig.json` uses for the `$lib` path.
- For SQL, replace `mongodb` with `knex` plus one driver: `pg`, `mysql2`, or `sqlite3`.

## tsconfig.json

`outDir` is `dist`. The sample uses `streambee` for that one field.

```json
{
	"compilerOptions": {
		"types": ["node", "express", "vitest/importMeta"],
		"rootDir": "src",
		"outDir": "dist",
		"module": "node16",
		"target": "esnext",
		"moduleResolution": "nodenext",
		"skipLibCheck": true,
		"resolveJsonModule": true,
		"esModuleInterop": true,
		"preserveSymlinks": true,
		"forceConsistentCasingInFileNames": true,
		"strict": true,
		"baseUrl": ".",
		"paths": {
			"$lib/*": ["src/*"]
		}
	},
	"include": ["src/**/*.ts", "src/bee/types/**/*.d.ts"],
	"exclude": ["node_modules", "**/node_modules", "**/*.test.ts", "dist"]
}
```

`src/**/*.ts` does not include declaration files, so `src/bee/types/**/*.d.ts` is listed on its own. `**/*.test.ts` stays out of the emit; Vitest still runs those files.

## vite.config.js

Vitest config, same as the sample. `dev` does not start the Vite dev server.

```js
// vite.config.ts
import { defineConfig } from 'vitest/config';
import path from 'path';
// import { fileURLToPath, URL } from 'node:url';
export default defineConfig({
	resolve: {
		alias: {
			$lib: path.resolve(__dirname, './src')
		}
	},
	test: {
		coverage: {
			reporter: ['text', 'json', 'html'],
			include: ['src/lib/**/*.ts']
		},
		include: ['src/**/*.{test,spec}.{js,ts}']
	}
});
```

## prettier.config.js

The sample keeps these options in `.prettierrc`: tabs, single quotes, no trailing commas, print width 100. That file also names `prettier-plugin-typescript` and a `*.ts` parser override. Prettier 3 parses TypeScript itself, and the sample does not install the plugin, so this config leaves both out.

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

## eslint.config.js

Flat config for the REST server, using the ESLint 9, `typescript-eslint`, and `eslint-config-prettier` versions the sample installs. `eslint-config-prettier` goes last so it turns off the formatting rules Prettier owns. `no-undef` is off because typescript-eslint reports false positives from it on TypeScript projects.

```js
import eslint from '@eslint/js';
import eslintConfigPrettier from 'eslint-config-prettier';
import tseslint from 'typescript-eslint';

export default tseslint.config(
	{ ignores: ['dist/**', 'node_modules/**'] },
	eslint.configs.recommended,
	tseslint.configs.recommended,
	eslintConfigPrettier,
	{
		rules: {
			'no-undef': 'off'
		}
	},
	{
		files: ['src/bee/types/**/*.d.ts'],
		rules: {
			'@typescript-eslint/no-empty-object-type': 'off'
		}
	}
);
```

`src/bee/types/extensions.d.ts` starts with empty `Models`, `Jobs`, and `Plugins` interfaces, which is how TypeScript declaration merging is seeded. `no-empty-object-type` flags those, so the rule is off for `src/bee/types/**/*.d.ts`. appbee regenerates that file in development and fills `Models` in.

## .prettierignore

The sample ignores lockfiles. `dist` is added so `format` leaves the build output alone.

```
# Package managers
package-lock.json
pnpm-lock.yaml
yarn.lock
bun.lock
bun.lockb

# Build
dist

# Miscellaneous
/static/
```
