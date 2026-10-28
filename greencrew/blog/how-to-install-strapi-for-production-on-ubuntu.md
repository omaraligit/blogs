# How to Install Strapi for Production on Ubuntu (PostgreSQL, PM2, Nginx, and HTTPS)

Author: [greencrew](https://greencrew.space/blog)

Tags: Strapi, Node.js, PostgreSQL, Nginx, PM2, DevOps

Description: To run Strapi in production on Ubuntu, create the project with npx create-strapi@latest, point it at PostgreSQL through DATABASE_CLIENT=postgres in .env, build the admin panel with NODE_ENV=production npm run build, start it with PM2, and proxy a domain to port 1337 with Nginx. This guide covers each step, plus HTTPS with Certbot, upload limits, the first admin account, updates, and troubleshooting.

Date: 2026-10-28

**Short answer:** Install a supported Node.js LTS release and PostgreSQL. Create the project with `npx create-strapi@latest my-cms` and run `npm install pg`. In `.env`, set `DATABASE_CLIENT=postgres` with your database credentials, `HOST=127.0.0.1`, and `PUBLIC_URL=https://cms.example.com`. Build the admin panel with `NODE_ENV=production npm run build`, then start Strapi with `pm2 start npm --name strapi -- run start`. Proxy your domain to `http://127.0.0.1:1337` with Nginx, add HTTPS with `certbot --nginx`, and register the first admin user right away at `/admin`.

## What Is Strapi?

[Strapi](https://strapi.io/) is an open-source headless CMS built on Node.js. You define content types in an admin panel, and Strapi generates REST (and optionally GraphQL) APIs that your frontend (React, Next.js, Vue, mobile apps) consumes. Self-hosting it means running a Node.js server, a database, and a reverse proxy, which is what this guide sets up.

| Component | Role | This guide uses |
| --- | --- | --- |
| Strapi | Admin panel + content API | `create-strapi` project on port 1337 |
| Node.js | Runtime | NodeSource LTS package |
| Database | Stores content, users, settings | PostgreSQL (MySQL/MariaDB also supported) |
| Process manager | Keeps Strapi running, restarts on crash and boot | PM2 |
| Reverse proxy | TLS, domain, upload size, compression | Nginx |

SQLite is Strapi's quick-start default. It's fine for local development, but use PostgreSQL or MySQL in production for concurrent writes, backups, and replication.

## Prerequisites

- Ubuntu 24.04 or 22.04 with a non-root `sudo` user
- At least 2 GB of RAM, or swap. The admin panel build can be killed by the OOM killer on smaller servers.
- A domain such as `cms.example.com` pointing at the server
- Nginx installed ([how to install Nginx on Ubuntu](/blog/how-to-install-nginx-on-ubuntu)) and ports 80/443 open

If your server has 1 GB of RAM, add swap before building:

```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Step 1: Install Node.js

Strapi supports only specific Node.js LTS releases. Check the [Strapi installation requirements](https://docs.strapi.io/cms/installation/cli) for the versions your Strapi release accepts, then install that major from NodeSource:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x -o nodesource_setup.sh
sudo -E bash nodesource_setup.sh
sudo apt install -y nodejs build-essential
node -v
```

Replace `22` with a version from Strapi's supported list. The trade-offs between NodeSource, nvm, and Ubuntu's package are in [how to install Node.js on Ubuntu](/blog/how-to-install-nodejs-on-ubuntu).

## Step 2: Install PostgreSQL and Create the Database

```bash
sudo apt install -y postgresql
sudo -u postgres psql <<'SQL'
CREATE USER strapi WITH PASSWORD 'change-this-strong-password';
CREATE DATABASE strapi OWNER strapi;
SQL
```

PostgreSQL listens only on localhost by default, which is what you want when Strapi runs on the same server.

## Step 3: Create the Strapi Project

Run the project as a regular user, not root:

```bash
mkdir -p ~/apps && cd ~/apps
npx create-strapi@latest my-cms
```

The CLI asks a few questions. Skip the Strapi Cloud login if you're self-hosting. You can accept the default SQLite database here and switch to PostgreSQL in the next step, which keeps the setup the same whether you create the project on the server or on your laptop.

Install the PostgreSQL driver:

```bash
cd my-cms
npm install pg
```

In a real workflow you'd create the project locally, commit it to Git (the generated `.gitignore` already excludes `.env`, `node_modules`, and build output), and `git clone` it onto the server. Content *types* are code and travel through Git. Content *entries* live in the database.

## Step 4: Configure .env for Production

The generated `config/database` file reads its settings from environment variables, so production configuration lives in `.env`:

```bash
# Server
HOST=127.0.0.1
PORT=1337
PUBLIC_URL=https://cms.example.com

# Database
DATABASE_CLIENT=postgres
DATABASE_HOST=127.0.0.1
DATABASE_PORT=5432
DATABASE_NAME=strapi
DATABASE_USERNAME=strapi
DATABASE_PASSWORD=change-this-strong-password
DATABASE_SSL=false

# Secrets: keep the generated values, or create new ones with: openssl rand -base64 32
APP_KEYS=...
API_TOKEN_SALT=...
ADMIN_JWT_SECRET=...
TRANSFER_TOKEN_SALT=...
JWT_SECRET=...
```

`HOST=127.0.0.1` keeps Strapi off the public interface, so only Nginx can reach it. Keep the secret values the generator wrote. Changing them later invalidates admin sessions and API tokens. If you cloned the project, copy `.env.example`, fill it in, and generate fresh secrets.

Tell Strapi its public URL and that it sits behind a proxy. Edit `config/server.ts` (or `.js`):

```typescript
export default ({ env }) => ({
  host: env('HOST', '0.0.0.0'),
  port: env.int('PORT', 1337),
  url: env('PUBLIC_URL', 'http://localhost:1337'),
  proxy: true,
  app: {
    keys: env.array('APP_KEYS'),
  },
});
```

`url` is used for admin panel links, emails, and upload URLs. The `proxy` option makes Strapi trust the `X-Forwarded-*` headers Nginx sends, so it knows requests arrived over HTTPS. The exact shape of the `proxy` option has changed between Strapi versions, so confirm it against the [server configuration docs](https://docs.strapi.io/cms/configurations/server) for your release.

## Step 5: Build the Admin Panel and Test

```bash
NODE_ENV=production npm run build
NODE_ENV=production npm run start
```

The build compiles the React admin panel with your `url` baked in, so **rebuild every time you change `PUBLIC_URL` or install a plugin**. In another terminal:

```bash
curl -I http://127.0.0.1:1337/_health
```

A `204 No Content` response means Strapi is up and connected to PostgreSQL. Stop it with `Ctrl+C`.

| Command | When to use it |
| --- | --- |
| `npm run develop` | Local development only. Enables the Content-Type Builder and auto-reload. |
| `npm run build` | After install, upgrades, plugin changes, or `url` changes |
| `npm run start` | Production. Content types are locked, no file watching. |

Production mode disables the Content-Type Builder by design. Change content types locally in `develop` mode, commit, and deploy.

## Step 6: Run Strapi with PM2

```bash
sudo npm install -g pm2
cd ~/apps/my-cms
NODE_ENV=production pm2 start npm --name strapi -- run start
pm2 save
pm2 startup systemd      # run the sudo command it prints
```

Or keep the settings in `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [{
    name: 'strapi',
    cwd: '/home/deploy/apps/my-cms',
    script: 'npm',
    args: 'run start',
    env: { NODE_ENV: 'production' },
    max_memory_restart: '1G',
  }],
};
```

```bash
pm2 start ecosystem.config.js && pm2 save
pm2 logs strapi
```

Run a single Strapi instance. Strapi isn't designed for PM2 cluster mode on one server. PM2 basics such as logs, reloads, and boot startup are covered in [how to deploy a Node.js app with PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx).

## Step 7: Configure Nginx as a Reverse Proxy

Create `/etc/nginx/sites-available/cms.example.com`:

```nginx
server {
    listen 80;
    server_name cms.example.com;

    client_max_body_size 100M;   # Media Library uploads

    location / {
        proxy_pass http://127.0.0.1:1337;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/cms.example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

| Directive | Why Strapi needs it |
| --- | --- |
| `client_max_body_size 100M` | Nginx's default 1 MB limit rejects image and video uploads with `413` |
| `X-Forwarded-Proto $scheme` | Strapi knows the request was HTTPS (secure cookies, correct URLs) |
| `proxy_read_timeout 120s` | Large uploads and image processing can take longer than the default 60 s |

Strapi has its own upload size limit (in the upload plugin's config). Raise both if large files fail. The header details are explained in [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy).

## Step 8: Enable HTTPS and Create the First Admin

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d cms.example.com
```

Full Certbot details, including renewal checks, are in [how to install a Let's Encrypt SSL certificate on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

**Then immediately open `https://cms.example.com/admin`.** On a fresh database, the first person to visit the admin panel creates the super admin account. Don't leave a new deployment sitting in that state on a public domain.

After that, harden it:

- Create separate admin roles for editors instead of sharing the super admin
- Review **Settings → Users & Permissions → Roles → Public**. Only expose the content types your frontend needs.
- Use API tokens with read-only scope for frontends that only read content
- Add [rate limiting in Nginx](/blog/how-to-set-up-rate-limiting-in-nginx) on `/admin/login` and `/api/auth/` routes

## Updating Strapi in Production

Back up first:

```bash
pg_dump -U strapi -h 127.0.0.1 strapi > ~/backups/strapi-$(date +%F).sql
tar czf ~/backups/uploads-$(date +%F).tgz -C ~/apps/my-cms public/uploads
```

Then upgrade locally with Strapi's upgrade tool (`npx @strapi/upgrade minor` or `major`), test, commit, and deploy:

```bash
cd ~/apps/my-cms
git pull
npm ci
NODE_ENV=production npm run build
pm2 restart strapi
```

Uploaded files live in `public/uploads` on the server by default and are *not* in Git. Back them up, or switch to an S3-compatible upload provider so the server holds no state.

## Troubleshooting Strapi in Production

| Symptom | Cause | Fix |
| --- | --- | --- |
| Build killed / `JavaScript heap out of memory` | Not enough RAM for the admin build | Add swap, or build in CI and deploy the output |
| `502 Bad Gateway` | Strapi not running or crashed on boot | `pm2 logs strapi`. See [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx). |
| `Cannot find module 'pg'` | PostgreSQL driver missing | `npm install pg`, rebuild, restart |
| `password authentication failed for user "strapi"` | Wrong `.env` credentials | Check `DATABASE_*` values. Test with `psql -U strapi -h 127.0.0.1 strapi`. |
| Admin panel loads but API calls go to `localhost:1337` | `url` not set before building | Set `PUBLIC_URL`, rebuild, restart |
| Login loops or secure-cookie errors | Strapi doesn't know it's behind HTTPS | Set the `proxy` option, send `X-Forwarded-Proto` |
| `413 Request Entity Too Large` on uploads | Nginx body size limit | Raise `client_max_body_size` |
| Content-Type Builder missing | Running in production mode | Edit content types locally with `npm run develop` |
| `Unsupported engine` on install | Node.js version outside Strapi's range | Install a supported LTS version |

## Monitor Your Strapi CMS

When Strapi goes down, every frontend that reads from it breaks too: product pages, blog posts, and mobile apps that fetch content at runtime. PM2 restarts a crashed process, but it won't tell you if PostgreSQL is down, the disk is full, or Nginx stopped proxying.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks from outside your server and alerts you on Discord, Slack, or Telegram when your site stops responding correctly, with a public status page to share with your content team. Point a check at `https://cms.example.com/_health`, and another at a public API endpoint your frontend relies on. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Can I use SQLite for Strapi in production?

You can for a small single-editor site, but PostgreSQL or MySQL is the safer choice. They handle concurrent writes, standard backup tools like `pg_dump` work against them, and they let you move the database to its own server later.

### Why can't I edit content types in production?

Strapi locks the Content-Type Builder when `NODE_ENV=production`, because content types are source code (files under `src/api`). Change them in development, commit, and deploy, so every environment stays in sync.

### Does Strapi need Nginx?

Strapi can listen publicly on its own, but Nginx gives you HTTPS with automatic certificate renewal, upload size control, compression, rate limiting, and the ability to host other sites on the same server. Keep Strapi on `127.0.0.1` and put Nginx in front.

### Can I run Strapi in Docker instead of PM2?

Yes. Build an image with a multi-stage Dockerfile and run it with PostgreSQL in Docker Compose, using the same `.env` variables. The pattern is shown in [how to build a Node.js application with Docker](/blog/how-to-build-a-nodejs-application-with-docker).

## Conclusion

Running Strapi in production on Ubuntu takes a supported Node.js LTS, PostgreSQL with a dedicated user, a `.env` with `DATABASE_CLIENT=postgres` and your public URL, a production build, PM2 for process management, and Nginx with HTTPS on port 443. Register the admin account the moment the site is live, back up the database and `public/uploads`, and rebuild after every URL or plugin change. Then let [GreenCrew](https://greencrew.space) watch the `/_health` endpoint so you hear about outages on Discord, Slack, or Telegram before your content team does.
