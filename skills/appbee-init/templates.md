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
  templateDir: 'template',
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
    password: process.env.DB_PASSWORD,
  },
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
    '/health': true,
  },
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
  secret: process.env.APPBEE_SECRET ?? 'appbee-default-secret-change-me',
};

export default security;
```

## src/bee/config/view.ts

```ts
import type { ViewConfig } from 'appbee';

const view: ViewConfig = {
  staticDir: 'public',
  viewDir: 'views',
  indexFile: 'index',
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
import type { AppModel, DBAware } from 'appbee';

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
