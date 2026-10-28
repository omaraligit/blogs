# How to Build a Node.js Application with Docker (Dockerfile, .dockerignore, and Docker Compose)

Author: [greencrew](https://greencrew.space/blog)

Tags: Node.js, Docker, Docker Compose, Nginx, DevOps

Description: To build a Node.js application with Docker, write a Dockerfile that starts FROM node:24-alpine, copies package*.json, runs npm ci --omit=dev, copies your code, switches to USER node, and ends with CMD ["node", "server.js"], then run docker build -t myapp . and docker run -p 3000:3000 myapp. This guide covers .dockerignore, multi-stage builds, health checks, graceful shutdown, Docker Compose with PostgreSQL, and putting Nginx in front.

Date: 2026-10-28

**Short answer:** Create a `Dockerfile` that uses an official Node.js LTS image (`FROM node:24-alpine`). Copy `package.json` and `package-lock.json` first and run `npm ci --omit=dev`, so Docker caches the dependency layer. Then copy the rest of your code, run as the unprivileged `node` user, and start with `CMD ["node", "server.js"]`. Add a `.dockerignore` that excludes `node_modules` and `.env`, then run `docker build -t myapp .` and `docker run -d -p 127.0.0.1:3000:3000 myapp`. Use Docker Compose to run the app next to a database, and Nginx in front for HTTPS.

## Why Run a Node.js App in Docker?

A Docker image packages your code, the exact Node.js version, and every dependency into one artifact that runs the same on your laptop, in CI, and on the server.

| Problem without Docker | How the container fixes it |
| --- | --- |
| "Works on my machine" Node version drift | The Node version is pinned in the `FROM` line |
| Native modules built for the wrong OS | `npm ci` runs inside the image's OS |
| Manual server setup for each app | `docker compose up -d` brings up app + database |
| Rollbacks mean redeploying old code | Re-run the previous image tag |

If you'd rather run Node directly on the host, see [how to install Node.js on Ubuntu](/blog/how-to-install-nodejs-on-ubuntu) and [how to deploy a Node.js app with PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx).

## Prerequisites

- Docker Engine with the Compose plugin. On Ubuntu, follow the [official install guide](https://docs.docker.com/engine/install/ubuntu/), then `sudo usermod -aG docker $USER` and log in again.
- A Node.js app with a `package-lock.json` (run `npm install` once to create it)

## Step 1: Create a Sample Node.js App

If you already have an app, skip to Step 2. Otherwise:

```bash
mkdir myapp && cd myapp
npm init -y
npm install express
```

Create `server.js` with a health route and graceful shutdown:

```javascript
const express = require('express');

const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => res.send(`Hello from Node.js ${process.version} in Docker`));
app.get('/health', (req, res) => res.json({ status: 'ok' }));

const server = app.listen(port, '0.0.0.0', () => console.log(`Listening on ${port}`));

// Docker sends SIGTERM on `docker stop`. Finish in-flight requests, then exit.
process.on('SIGTERM', () => {
  console.log('SIGTERM received, shutting down');
  server.close(() => process.exit(0));
});
```

Inside a container the app must listen on `0.0.0.0`, not `127.0.0.1`. Otherwise the published port can't reach it.

## Step 2: Write the Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine

ENV NODE_ENV=production
WORKDIR /app

# Install dependencies first so this layer is cached between code changes
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Copy the application code
COPY --chown=node:node . .

USER node
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"

CMD ["node", "server.js"]
```

| Instruction | Why it's written this way |
| --- | --- |
| `FROM node:24-alpine` | Official image, pinned LTS major, small base. Use `node:24-slim` (Debian) if a native module needs glibc. |
| `ENV NODE_ENV=production` | Express and many libraries enable production optimizations |
| `COPY package*.json` before code | Code changes don't invalidate the cached `npm ci` layer |
| `npm ci --omit=dev` | Exact versions from the lockfile, no devDependencies |
| `USER node` | The official image ships a non-root `node` user. Don't run as root. |
| `HEALTHCHECK` with `node -e` | Alpine images have no `curl`. Node's built-in `fetch` does the job. |
| `CMD ["node", "server.js"]` | Exec form, so Node receives `SIGTERM` directly. `npm start` doesn't always forward signals. |

The [Docker and Node.js best practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md) from the official image maintainers explain each of these in more depth.

## Step 3: Add a .dockerignore File

Without this, `COPY . .` sends your local `node_modules`, Git history, and secrets into the image:

```gitignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.*
Dockerfile
docker-compose*.yml
coverage
dist
```

Remove `dist` from this list if you build outside Docker and copy the output in.

## Step 4: Build and Run the Image

```bash
docker build -t myapp:1.0 .
docker run -d --name myapp -p 127.0.0.1:3000:3000 --restart unless-stopped myapp:1.0
curl http://127.0.0.1:3000/health
```

| Command | What it does |
| --- | --- |
| `docker ps` | Running containers and health status |
| `docker logs -f myapp` | Follow stdout/stderr (log to stdout, not files) |
| `docker exec -it myapp sh` | Shell inside the container |
| `docker stop myapp` | Sends `SIGTERM`, then `SIGKILL` after 10 seconds |
| `docker image ls myapp` | Image tags and sizes |

Binding to `127.0.0.1:3000` matters. Ports published by Docker bypass UFW rules, so `-p 3000:3000` exposes the app to the internet even if your firewall says otherwise. Keep it on localhost and let Nginx handle public traffic.

## Step 5: Use a Multi-Stage Build for TypeScript or Bundled Apps

If your app has a build step (TypeScript, NestJS, Next.js standalone), build with dev dependencies in one stage and ship only the output:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:24-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY --from=build --chown=node:node /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

The final image contains no TypeScript compiler, no test tools, and no source maps unless you copy them.

## Step 6: Run the App and a Database with Docker Compose

Create `compose.yaml`:

```yaml
services:
  app:
    build: .
    image: myapp:latest
    init: true
    restart: unless-stopped
    env_file: .env
    environment:
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
    ports:
      - "127.0.0.1:3000:3000"
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 10s
      retries: 5

volumes:
  pgdata:
```

Put `POSTGRES_PASSWORD=...` in `.env` (and keep `.env` out of Git), then:

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f app
```

| Key | Purpose |
| --- | --- |
| `init: true` | Runs a tiny init process as PID 1 that forwards signals and reaps zombie processes |
| `depends_on: condition: service_healthy` | Waits for Postgres to accept connections, not just start |
| `db` hostname | Containers on the same Compose network reach each other by service name |
| `pgdata` volume | Database files survive `docker compose down` (but not `down -v`) |

See the [Compose file reference](https://docs.docker.com/reference/compose-file/) for every option. For a PHP version of this pattern, see [Laravel, Nginx, and MySQL with Docker Compose](/blog/how-to-set-up-laravel-nginx-mysql-with-docker-compose).

### Live Reload in Development

Mount your code and use Node's built-in watcher in a `compose.override.yaml`, which Compose loads automatically:

```yaml
services:
  app:
    command: node --watch server.js
    environment:
      NODE_ENV: development
    volumes:
      - .:/app
      - /app/node_modules
```

The anonymous `/app/node_modules` volume stops your host folder from hiding the container's installed modules.

## Step 7: Put Nginx in Front for HTTPS

On the host, proxy your domain to the container's localhost port:

```nginx
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Then add a certificate with Certbot. The full header explanation is in [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy), and HTTPS is covered in [how to install Let's Encrypt SSL on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot). If Express reads client IPs, add `app.set('trust proxy', 1)`.

## Deploying Updates

```bash
git pull
docker compose up -d --build app      # rebuilds and replaces only the app container
docker image prune -f                 # clean up old layers
```

Tag images with a version or Git SHA (`myapp:3f2a1c9`) so a rollback is `docker compose up -d` with the previous tag.

## Troubleshooting Node.js in Docker

| Symptom | Cause | Fix |
| --- | --- | --- |
| `curl: (52) Empty reply` / connection reset | App listens on `127.0.0.1` inside the container | Listen on `0.0.0.0` |
| `Cannot find module 'express'` | `node_modules` excluded and `npm ci` didn't run, or a bind mount hides it | Check the Dockerfile order. Add the `/app/node_modules` anonymous volume in dev. |
| `npm ci` fails: lockfile missing | No `package-lock.json` | Run `npm install` locally and commit the lockfile |
| `docker stop` takes 10 seconds | Node isn't handling `SIGTERM` (PID 1) | Handle `SIGTERM`, use exec-form `CMD`, add `init: true` |
| `Error relocating ... symbol not found` | Native module needs glibc on Alpine | Switch to `node:24-slim` |
| `EACCES` writing files | Running as `node` user without ownership | `COPY --chown=node:node`, or write to a volume owned by `node` |
| App can't reach database at `localhost` | `localhost` is the container itself | Use the Compose service name (`db`) |
| Every build reinstalls dependencies | Code copied before `npm ci` | Copy `package*.json` first |
| Container `unhealthy` | `/health` failing or slow to start | `docker inspect --format '{{json .State.Health}}' myapp`. Raise `start-period`. |

## Monitor the Container from Outside

Docker's `HEALTHCHECK` and `restart: unless-stopped` cover a crashed process, but they run on the same host as your app. If the server, Docker daemon, Nginx, or network goes down, nothing inside the box can tell you.

[GreenCrew](https://greencrew.space) checks your public URL over HTTP from outside and alerts you on Discord, Slack, or Telegram when it stops responding correctly, with a public status page for your users. Point it at the same `/health` route the container uses. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should I use node:alpine or node:slim?

Alpine images are smaller and work for most pure-JavaScript apps. Use `slim` (Debian-based) when a native dependency needs glibc, or when you hit odd DNS or build issues on Alpine's musl libc.

### Should I use npm start as the CMD?

Prefer `CMD ["node", "server.js"]`. Running through `npm` adds an extra process that may not forward `SIGTERM`, which makes shutdowns slow and can cut off in-flight requests.

### Do I need PM2 inside a Docker container?

Usually not. Docker (or your orchestrator) already restarts the container, and you scale by running more containers. If you want to use every CPU core in one container, see [how to scale Node.js with clustering](/blog/how-to-scale-nodejs-applications-with-clustering).

### How do I pass secrets to the container?

Use environment variables from an `.env` file referenced by `env_file`, or Docker secrets. Never `COPY` a `.env` into the image or put secrets in `ENV` lines, because they're baked into the image layers.

## Conclusion

A solid Node.js Dockerfile is short. Use an official LTS base image, copy the lockfile before your code, run `npm ci --omit=dev`, switch to the `node` user, and start Node directly so it receives signals. Add a `.dockerignore`, publish ports only on `127.0.0.1`, run your database next to the app with Docker Compose, and put Nginx in front for HTTPS. Then add an outside check with [GreenCrew](https://greencrew.space) so you hear about downtime on Discord, Slack, or Telegram before your users do.
