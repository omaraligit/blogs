# How to Deploy a NestJS Application with Nginx on Ubuntu (Build, PM2, Health Checks, and HTTPS)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, NestJS, Node.js, TypeScript, DevOps

Description: To deploy a NestJS app, run npm run build to compile TypeScript into dist/, start dist/main.js with PM2 on 127.0.0.1:3000, and put Nginx in front with proxy_pass http://127.0.0.1:3000. This guide covers main.ts production settings (trust proxy, shutdown hooks, global prefix), environment config, a Terminus health check, WebSocket gateways, HTTPS, and common NestJS deployment errors.

Date: 2026-10-19

**Short answer:** Don't run `nest start` or `ts-node` in production. Build the app with `npm ci && npm run build`, which compiles TypeScript into `dist/`, then run `node dist/main.js` under a process manager like PM2 (`pm2 start dist/main.js --name api`). In `main.ts`, listen on `127.0.0.1`, enable `trust proxy`, and call `app.enableShutdownHooks()`. Put Nginx in front with `proxy_pass http://127.0.0.1:3000;` and the usual `X-Forwarded-*` headers, add WebSocket headers if you use gateways, and run Certbot for HTTPS. Expose a `/health` endpoint with `@nestjs/terminus` so you can monitor the app from outside.

## What Changes When You Deploy NestJS to Production?

NestJS is a TypeScript framework for Node.js that runs on Express by default (or Fastify). In development, `npm run start:dev` compiles and watches your source on the fly. Production is different:

| | Development | Production |
| --- | --- | --- |
| Command | `nest start --watch` | `node dist/main.js` |
| TypeScript | Compiled on the fly | Compiled once with `nest build` |
| Dependencies | All, including dev | Production only after the build |
| Process | Your terminal | PM2 or systemd, restarts on crash and reboot |
| Port | `localhost:3000` in the browser | `127.0.0.1:3000`, only reachable through Nginx |
| TLS, compression, static files | None | Nginx |

The PM2 and Nginx layers are the same as for any Node.js app, covered in depth in [how to deploy a Node.js app with PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx). This guide focuses on the NestJS-specific parts: the build, `main.ts`, configuration, health checks, and gateways.

## Prerequisites

- Ubuntu 24.04 or 22.04 with a non-root sudo user
- Node.js LTS installed (NodeSource or nvm, see the [Node.js PM2 guide](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx))
- Nginx installed ([how to install Nginx on Ubuntu](/blog/how-to-install-nginx-on-ubuntu))
- A NestJS project in Git and a domain such as `api.example.com` pointed at the server

## Step 1: Prepare main.ts for Production

Most NestJS deployment problems trace back to `main.ts`. Here's a production-ready version for the default Express platform:

```ts
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { NestExpressApplication } from '@nestjs/platform-express';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create<NestExpressApplication>(AppModule);

  // Nginx is the only proxy in front of the app
  app.set('trust proxy', 'loopback');

  // Run onModuleDestroy / beforeApplicationShutdown on SIGTERM and SIGINT
  app.enableShutdownHooks();

  app.setGlobalPrefix('api', { exclude: ['health'] });

  app.enableCors({
    origin: ['https://example.com'],
    credentials: true,
  });

  const port = Number(process.env.PORT ?? 3000);
  await app.listen(port, '127.0.0.1');
}
bootstrap();
```

| Line | Why it matters in production |
| --- | --- |
| `NestExpressApplication` | Types the app as Express so `app.set()` is available |
| `trust proxy` | Without it, `req.ip` is always `127.0.0.1` and `req.protocol` is `http`, which breaks rate limiting, logging, and secure cookies |
| `enableShutdownHooks()` | Lets Nest close database connections cleanly when PM2 or systemd stops the process. See the [lifecycle events docs](https://docs.nestjs.com/fundamentals/lifecycle-events#application-shutdown) |
| `setGlobalPrefix('api')` | All routes live under `/api`, which keeps the Nginx config simple when a frontend shares the domain |
| `enableCors` | Only needed if a browser app on another origin calls the API. Never use `origin: '*'` with `credentials: true` |
| `listen(port, '127.0.0.1')` | The app is only reachable through Nginx, not directly on port 3000 |

**Using Fastify?** Pass `new FastifyAdapter({ trustProxy: true })` to `NestFactory.create`, and call `app.listen(port, '127.0.0.1')` the same way. Fastify listens on localhost by default, but stating the host makes it explicit.

## Step 2: Load Configuration from Environment Variables

Don't commit secrets. Use the official `@nestjs/config` module to read a `.env` file that lives only on the server:

```bash
npm install @nestjs/config
```

```ts
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
    // ...other modules
  ],
})
export class AppModule {}
```

On the server:

```ini
# /var/www/api/.env
NODE_ENV=production
PORT=3000
DATABASE_URL=postgres://api:a-long-random-password@127.0.0.1:5432/api
JWT_SECRET=replace-with-a-long-random-string
```

```bash
chmod 600 /var/www/api/.env
```

Inject `ConfigService` wherever you need a value (`config.get<string>('DATABASE_URL')`). The [NestJS configuration docs](https://docs.nestjs.com/techniques/configuration) also show how to validate the environment at startup, which makes a missing variable fail loudly on boot instead of on the first request.

## Step 3: Add a Health Check with Terminus

A health endpoint gives Nginx, PM2 users, and external monitors one URL that answers "is this app actually working?"

```bash
npm install @nestjs/terminus
```

```ts
// src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
} from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}
```

```ts
// src/health/health.module.ts
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

Import `HealthModule` in `AppModule`. When the database ping fails, `/health` returns **503** with details instead of 200. Terminus also has indicators for Prisma, Mongoose, HTTP dependencies, memory, and disk; see the [NestJS health checks docs](https://docs.nestjs.com/recipes/terminus). If you don't use TypeORM, swap in the matching indicator or return `this.health.check([])` for a basic liveness check.

## Step 4: Build the NestJS App on the Server

```bash
sudo mkdir -p /var/www/api
sudo chown $USER:$USER /var/www/api
git clone https://github.com/you/api.git /var/www/api
cd /var/www/api

npm ci                    # needs dev dependencies for the TypeScript build
npm run build             # nest build -> dist/
npm prune --omit=dev      # drop dev dependencies afterwards
ls dist/main.js
```

Run it once in the foreground to catch startup errors:

```bash
NODE_ENV=production node dist/main.js
# in another terminal:
curl http://127.0.0.1:3000/health
```

If you use an ORM with migrations (TypeORM, Prisma, MikroORM), run them now, before starting the new build. With Prisma, for example, that's `npx prisma migrate deploy`. Keep the Prisma CLI available at this point (run migrations before `npm prune`, or keep `prisma` in `dependencies`).

### Monorepos and Path Aliases

If `dist/main.js` doesn't exist, check `nest-cli.json` and `tsconfig.build.json`. In a Nest monorepo the output is `dist/apps/<app-name>/main.js`. If startup fails with `Cannot find module '@app/...'`, the TypeScript path aliases weren't resolved at build time; the Nest CLI handles this for monorepo libraries, but custom `paths` in a standard project need the Nest CLI build (not plain `tsc`) or a tool like `tsconfig-paths`.

## Step 5: Run NestJS with PM2

```bash
sudo npm install -g pm2
```

Create `ecosystem.config.js` in the project root:

```js
module.exports = {
  apps: [
    {
      name: 'api',
      script: 'dist/main.js',
      cwd: '/var/www/api',
      instances: 2,
      exec_mode: 'cluster',
      max_memory_restart: '500M',
      kill_timeout: 10000,
      env: {
        NODE_ENV: 'production',
      },
    },
  ],
};
```

```bash
pm2 start ecosystem.config.js
pm2 startup systemd      # run the sudo command it prints
pm2 save
```

`kill_timeout: 10000` gives Nest 10 seconds to run its shutdown hooks before PM2 force-kills the process. Two cluster instances mean `pm2 reload api` restarts them one at a time without downtime. Startup scripts, logs, and log rotation are covered in the [Node.js PM2 guide](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx).

**Cluster mode and NestJS:** each instance has its own memory. In-memory caches, `@nestjs/schedule` cron jobs, and WebSocket rooms aren't shared. Cron jobs will run once **per instance**, so either run scheduled tasks in a separate single-instance process or use a lock in Redis or the database. For gateways across instances, use the Socket.IO Redis adapter.

## Step 6: Configure Nginx for NestJS

Create `/etc/nginx/sites-available/api`:

```nginx
upstream nest_api {
    server 127.0.0.1:3000;
    keepalive 32;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name api.example.com;

    client_max_body_size 10M;

    location / {
        proxy_pass http://nest_api;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
    }

    # NestJS WebSocket gateway (Socket.IO default path)
    location /socket.io/ {
        proxy_pass http://nest_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 3600s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
curl -I http://api.example.com/health
```

| Directive | Purpose |
| --- | --- |
| `upstream` + `keepalive 32` | Reuses connections to Nest instead of opening one per request |
| `proxy_set_header Connection ""` | Required for upstream keepalive on normal HTTP requests |
| `X-Forwarded-For` / `X-Forwarded-Proto` | Real client IP and scheme, read by Express because of `trust proxy` |
| `location /socket.io/` | Upgrades WebSocket connections for `@WebSocketGateway()`; see [nginx WebSocket proxying](https://nginx.org/en/docs/http/websocket.html) |
| `proxy_read_timeout 3600s` | Stops idle sockets being closed after the 60s default |
| `client_max_body_size` | Raise it if you accept uploads with `FileInterceptor` (default is 1 MB) |

**Socket.IO with PM2 cluster mode:** Socket.IO's HTTP long-polling fallback needs sticky sessions, which PM2's cluster balancer doesn't provide. Either configure clients to use the WebSocket transport only, or run separate instances on different ports with `ip_hash` in the Nginx upstream (see [Nginx load balancing](/blog/how-to-set-up-nginx-load-balancing-with-failover)).

To protect a public API from abuse, add [Nginx rate limiting](/blog/how-to-set-up-rate-limiting-in-nginx) in front of auth routes, or use `@nestjs/throttler` in the app. Throttler only sees real IPs if `trust proxy` is set.

## Step 7: Enable HTTPS

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.example.com
```

Certbot installs the certificate and redirects HTTP to HTTPS. Nest needs no TLS code, because Nginx terminates TLS and forwards `X-Forwarded-Proto: https`. Full guide: [how to install Let's Encrypt SSL on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

## Deploying NestJS Updates

```bash
#!/usr/bin/env bash
set -e
cd /var/www/api
git pull
npm ci
npm run build
# run database migrations here
npm prune --omit=dev
pm2 reload ecosystem.config.js --update-env
```

Building on the server is simplest. For bigger apps, build in CI, ship `dist/`, `package.json`, and `package-lock.json`, and run only `npm ci --omit=dev` on the server, so a failed build never touches production.

## Troubleshooting NestJS Deployments

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Error: Cannot find module '/var/www/api/dist/main'` | Build not run, or output path differs | `npm run build`, check `nest-cli.json` for the output path |
| `Cannot find module '@nestjs/...'` at startup | Package is in `devDependencies` but used at runtime | Move it to `dependencies`, reinstall |
| 502 Bad Gateway, `connect() failed (111: Connection refused)` | App crashed or listens on another port/host | `pm2 logs api --err`, compare `PORT` with the `upstream` |
| App works locally, crashes on server with `undefined` config | `.env` missing on server or not loaded | Check the file path, use `ConfigModule` validation |
| `req.ip` is `127.0.0.1`, throttler blocks everyone at once | `trust proxy` not set | `app.set('trust proxy', 'loopback')` |
| CORS error in the browser | Origin not in `enableCors`, or CORS headers added twice (Nginx and Nest) | Configure CORS in one place only |
| WebSocket fails with 400 or falls back to polling | `Upgrade` headers not forwarded | Add the `/socket.io/` location |
| Cron jobs run twice | PM2 cluster mode with 2 instances | Run schedulers in one instance or use a lock |
| `EADDRINUSE: address already in use :::3000` | An old process still holds the port | `pm2 list`, `sudo ss -ltnp \| grep 3000`, stop the duplicate |
| Database connections not closed on deploy | Shutdown hooks off | `app.enableShutdownHooks()` and `kill_timeout` in PM2 |

For more on gateway errors, read [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Your NestJS API

PM2 restarts a crashed process, but it reports "online" for an app that answers every request with a 503 because its database is gone. The Terminus endpoint from Step 3 turns that state into a status code, and an external check turns the status code into an alert.

[GreenCrew](https://greencrew.space) monitors `https://api.example.com/health` from outside your server and sends alerts to Discord, Slack, or Telegram when it fails. You also get a public status page for the people who use your API. Because the check runs through DNS, TLS, Nginx, and Nest, it catches failures in any layer. Setup: [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should I run NestJS with PM2, systemd, or Docker?

All three work behind the same Nginx config. PM2 is quickest on a single server and adds cluster mode and zero-downtime reloads. A systemd unit running `node dist/main.js` has no extra dependencies. Docker suits teams that already build images in CI; publish the container port on `127.0.0.1` only, so it isn't exposed past the firewall.

### Do I need Nginx in front of NestJS?

For a public API, yes in practice. Nginx handles TLS certificates, HTTP/2, compression, request size limits, slow clients, and static files, and lets you run several apps on one server. NestJS can serve HTTPS itself with `httpsOptions`, but then the app has to handle certificate renewal and run with permission to bind port 443.

### Why does my NestJS app work with npm run start:dev but not in production?

The usual causes are a runtime package listed under `devDependencies` (removed by `npm prune --omit=dev`), a missing `.env` file on the server, or TypeScript path aliases that only resolve in development. Run `node dist/main.js` in the foreground on the server to see the actual error.

### How do I serve a frontend and the NestJS API on the same domain?

Use `app.setGlobalPrefix('api')` and add a `location /api/ { proxy_pass http://nest_api; ... }` block, then serve the frontend build from `root` with `try_files`, as in [how to deploy a React app with Nginx](/blog/how-to-deploy-react-app-with-nginx). Same-origin requests also mean you don't need CORS.

## Conclusion

Deploying NestJS on Ubuntu means compiling once with `npm run build`, running `dist/main.js` under PM2 with shutdown hooks enabled, and putting Nginx in front for TLS, WebSockets, and request limits. Get `main.ts` right (`trust proxy`, `127.0.0.1`, `enableShutdownHooks()`), keep secrets in a server-only `.env`, and expose a Terminus `/health` endpoint. Then point [GreenCrew](https://greencrew.space) at that endpoint so you get an alert when the API stops responding, not a support ticket.
