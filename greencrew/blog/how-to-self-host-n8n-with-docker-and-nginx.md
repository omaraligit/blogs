# How to Self-Host n8n with Docker and Nginx (Docker Compose, WebSockets, SSL, and WEBHOOK_URL)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, n8n, Docker, Automation, DevOps

Description: To self-host n8n, run the docker.n8n.io/n8nio/n8n image with Docker Compose bound to 127.0.0.1:5678, set N8N_HOST, N8N_PROTOCOL=https, and WEBHOOK_URL, then put Nginx in front with proxy_pass http://127.0.0.1:5678 plus WebSocket Upgrade headers and a Let's Encrypt certificate. This guide covers the compose file, PostgreSQL, the encryption key, backups, updates, and fixing secure cookie, WebSocket, and webhook URL errors.

Date: 2026-10-19

**Short answer:** Install Docker, create `/opt/n8n/docker-compose.yml` that runs `docker.n8n.io/n8nio/n8n` with a persistent volume, and publish the port only on localhost (`127.0.0.1:5678:5678`). Set `N8N_HOST=n8n.example.com`, `N8N_PROTOCOL=https`, `WEBHOOK_URL=https://n8n.example.com/`, and `N8N_PROXY_HOPS=1`. Put Nginx in front with `proxy_pass http://127.0.0.1:5678;` and the `Upgrade`/`Connection` headers, because the n8n editor uses WebSockets. Run Certbot for HTTPS, then open the domain and create the owner account. Back up the data volume, because it holds your workflows, credentials, and the encryption key.

## What Is n8n and Why Self-Host It?

n8n is a workflow automation tool, similar to Zapier or Make, with a visual editor and hundreds of integrations. It's distributed under n8n's [Sustainable Use License](https://docs.n8n.io/sustainable-use-license/), and you can run it on your own server. Self-hosting gives you:

- **Data control**: credentials and execution data stay on your server
- **No per-task pricing**: you pay for the server, not per workflow run
- **Custom code**: Code nodes and community nodes without platform limits

The trade-off is that you own uptime, updates, backups, and security. This guide sets up the pieces that make that manageable.

### How the Setup Fits Together

| Component | Job |
| --- | --- |
| Docker + Compose | Runs n8n (and optionally PostgreSQL) with pinned versions and easy updates |
| n8n container | Editor UI, API, webhooks, and workflow execution on port 5678 |
| Docker volume | Persists `/home/node/.n8n`: SQLite database, encryption key, settings |
| Nginx | Public ports 80/443, TLS, WebSocket proxying, upload limits |
| Certbot | Free Let's Encrypt certificate |

n8n's own [Docker installation docs](https://docs.n8n.io/hosting/installation/docker/) are the reference for the container options used here.

## Prerequisites

- Ubuntu 24.04 or 22.04 with a sudo user. n8n's docs list minimum resources; 2 GB of RAM is a comfortable start for light use
- A subdomain such as `n8n.example.com` with an A record pointing at the server
- Nginx installed ([how to install Nginx on Ubuntu](/blog/how-to-install-nginx-on-ubuntu))

## Step 1: Install Docker and Docker Compose

Install Docker Engine from Docker's own apt repository, following the [official Ubuntu instructions](https://docs.docker.com/engine/install/ubuntu/). That gives you the `docker compose` plugin too. Then check:

```bash
docker --version
docker compose version
sudo docker run --rm hello-world
```

Optionally add your user to the `docker` group (`sudo usermod -aG docker $USER`, then log out and back in). Members of that group effectively have root access, so only add trusted users.

## Step 2: Create the n8n Docker Compose File

```bash
sudo mkdir -p /opt/n8n
sudo chown $USER:$USER /opt/n8n
cd /opt/n8n
```

Create `/opt/n8n/.env`:

```ini
N8N_HOST=n8n.example.com
GENERIC_TIMEZONE=Europe/Paris
N8N_ENCRYPTION_KEY=replace-with-a-long-random-string
POSTGRES_PASSWORD=another-long-random-password
```

Generate random values with `openssl rand -hex 32`, then `chmod 600 .env`.

Create `/opt/n8n/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n -d n8n"]
      interval: 10s
      timeout: 5s
      retries: 5

  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: unless-stopped
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      N8N_HOST: ${N8N_HOST}
      N8N_PORT: 5678
      N8N_PROTOCOL: https
      WEBHOOK_URL: https://${N8N_HOST}/
      N8N_PROXY_HOPS: 1
      N8N_ENCRYPTION_KEY: ${N8N_ENCRYPTION_KEY}
      GENERIC_TIMEZONE: ${GENERIC_TIMEZONE}
      TZ: ${GENERIC_TIMEZONE}
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  n8n_data:
  postgres_data:
```

In production, pin the n8n image to a specific version tag (check n8n's release notes for the current one) instead of the default `latest`, so updates only happen when you choose.

### Key n8n Environment Variables

| Variable | Value | Why it matters |
| --- | --- | --- |
| `N8N_HOST` | `n8n.example.com` | The public hostname n8n uses for its own URLs |
| `N8N_PROTOCOL` | `https` | n8n sits behind Nginx's TLS, so public URLs must be HTTPS |
| `WEBHOOK_URL` | `https://n8n.example.com/` | **Required behind a proxy.** Without it, webhook nodes show `http://localhost:5678/...` URLs that nothing outside can reach |
| `N8N_PROXY_HOPS` | `1` | Tells n8n to trust one proxy's `X-Forwarded-For`, so logs and rate limits see real client IPs |
| `N8N_ENCRYPTION_KEY` | Random string | Encrypts stored credentials. **Lose it and every saved credential is unreadable** |
| `GENERIC_TIMEZONE` | Your time zone | Used by Schedule/Cron trigger nodes |
| `DB_TYPE` | `postgresdb` | PostgreSQL instead of the default SQLite |

The full list is in the [n8n environment variables reference](https://docs.n8n.io/hosting/configuration/environment-variables/).

### SQLite or PostgreSQL?

| | SQLite (default) | PostgreSQL |
| --- | --- | --- |
| Setup | Nothing to configure, a file in the volume | One more container |
| Good for | Personal use, a few workflows | Teams, many executions, queue mode |
| Backups | Copy the volume (stop n8n first) | `pg_dump` while running |
| Scaling | Single instance only | Required for n8n's queue mode with workers |

To use SQLite, delete the `postgres` service, the `DB_*` variables, and `depends_on`.

### Why Bind to 127.0.0.1?

Docker publishes ports by writing its own iptables rules, and those rules **bypass UFW**. A plain `"5678:5678"` exposes n8n to the internet over unencrypted HTTP even if UFW only allows 80 and 443. Binding to `127.0.0.1` makes Nginx the only way in.

## Step 3: Start n8n

```bash
cd /opt/n8n
docker compose up -d
docker compose ps
docker compose logs -f n8n
curl http://127.0.0.1:5678/healthz
```

The logs should show n8n is ready on port 5678, and `/healthz` returns `{"status":"ok"}`.

## Step 4: Configure Nginx as a Reverse Proxy for n8n

Create `/etc/nginx/sites-available/n8n`:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    listen [::]:80;
    server_name n8n.example.com;

    client_max_body_size 16M;

    location / {
        proxy_pass http://127.0.0.1:5678;
        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

| Directive | Why n8n needs it |
| --- | --- |
| `Upgrade` / `Connection` headers + `proxy_http_version 1.1` | The editor receives live execution updates over a WebSocket. Without these, you get "Connection lost" in the UI |
| `map $http_upgrade $connection_upgrade` | Sends `Connection: upgrade` only for WebSocket requests, `close` otherwise ([nginx WebSocket docs](https://nginx.org/en/docs/http/websocket.html)) |
| `proxy_read_timeout 3600s` | Keeps idle WebSocket connections and long webhook executions from being cut at the 60s default |
| `proxy_buffering off` | Streams responses immediately instead of buffering them |
| `client_max_body_size 16M` | Nginx's default is 1 MB, which rejects larger webhook payloads and file uploads with `413` |
| `X-Forwarded-*` headers | Lets n8n see the real client IP and protocol (paired with `N8N_PROXY_HOPS=1`) |

More on proxy headers and timeouts: [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy).

## Step 5: Add HTTPS with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d n8n.example.com
```

Certbot adds `listen 443 ssl`, the certificate paths, and an HTTP to HTTPS redirect. HTTPS is not optional here: n8n sets its session cookie as **secure** by default, so logging in over plain HTTP fails with a warning about the secure cookie. Certbot details and renewal checks are in [how to install Let's Encrypt SSL on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

## Step 6: Create the Owner Account and Lock It Down

Open `https://n8n.example.com`. On first visit, n8n asks you to create the **owner** account. Do it right away, because until you do, anyone who finds the URL can claim it.

Then harden it:

- **Turn on two-factor authentication** for the owner account in n8n's settings.
- **Rate limit the login endpoint** in Nginx so password guessing gets slow. See [how to set up rate limiting in Nginx](/blog/how-to-set-up-rate-limiting-in-nginx).
- **Restrict the editor by IP** if only your team uses it, but keep `/webhook/` open so external services can still trigger workflows:

```nginx
location /webhook/ {
    proxy_pass http://127.0.0.1:5678;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Add `allow <your-ip>; deny all;` to the main `location /` block. Test webhooks with the `/webhook-test/` path from inside the editor, which goes through `location /`.

## Step 7: Back Up and Update n8n

### Back Up

```bash
# PostgreSQL dump
cd /opt/n8n
docker compose exec -T postgres pg_dump -U n8n n8n | gzip > ~/n8n-db-$(date +%F).sql.gz

# Data volume (encryption key, settings)
docker run --rm -v n8n_n8n_data:/data -v ~/:/backup alpine \
  tar czf /backup/n8n-data-$(date +%F).tar.gz -C /data .
```

Compose prefixes volume names with the project directory, so the volume is `n8n_n8n_data` here; confirm with `docker volume ls`. Keep a copy of `/opt/n8n/.env` (with the encryption key) somewhere off the server. You can also export workflows with the n8n CLI: `docker compose exec n8n n8n export:workflow --all --output=/home/node/.n8n/workflows.json`.

### Update

```bash
cd /opt/n8n
# change the image tag in docker-compose.yml if you pinned one
docker compose pull
docker compose up -d
docker compose logs -f n8n
docker image prune -f
```

n8n runs database migrations on startup. Take a backup first, and read the release notes for breaking changes before jumping several versions.

## Troubleshooting Self-Hosted n8n

| Symptom | Cause | Fix |
| --- | --- | --- |
| Login page warns that n8n is "configured to use a secure cookie" but you are visiting via an insecure URL | Accessing over HTTP | Use the HTTPS URL; finish Step 5 |
| "Connection lost" banner in the editor | WebSocket upgrade not proxied | Add the `Upgrade`/`Connection` headers and `proxy_http_version 1.1` |
| Webhook node shows `http://localhost:5678/webhook/...` | `WEBHOOK_URL` not set | Set `WEBHOOK_URL=https://n8n.example.com/`, `docker compose up -d` |
| External service gets 404 on webhook | Workflow not activated, or using the test URL | Activate the workflow and use the production `/webhook/` URL |
| `413 Request Entity Too Large` | Nginx 1 MB body limit | Raise `client_max_body_size` |
| 502 Bad Gateway | Container stopped or still starting | `docker compose ps`, `docker compose logs n8n` |
| Credentials fail after migrating servers | Different `N8N_ENCRYPTION_KEY` | Restore the original key |
| `EACCES: permission denied, open '/home/node/.n8n/config'` | Volume or bind mount owned by root | `sudo chown -R 1000:1000` the bind-mounted directory (the container runs as `node`, UID 1000) |
| Schedules fire at the wrong hour | Time zone not set | Set `GENERIC_TIMEZONE` and `TZ` |
| n8n reachable on `:5678` from the internet | Port published on all interfaces | Use `127.0.0.1:5678:5678` |

## Monitor n8n Uptime

If n8n goes down, nothing tells you, because the tool that would send the alert is the one that's down. Scheduled workflows silently stop, and webhooks from Stripe, GitHub, or your forms start failing on the sender's side.

[GreenCrew](https://greencrew.space) monitors `https://n8n.example.com/healthz` from outside your server and sends alerts to Discord, Slack, or Telegram when it stops answering, independent of n8n itself. The check goes through DNS, Nginx, TLS, and the container, so any broken layer triggers it. You can also put it on a public status page if other teams depend on your automations. Setup: [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

If you restricted the editor by IP, add a `location = /healthz` block without the `deny` so the external check can reach it.

## Frequently Asked Questions

### Can I run n8n without Docker?

Yes. You can install it with `npm install -g n8n` on Node.js and run it under PM2 or systemd, as in [how to deploy a Node.js app with PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx). Docker is the method n8n recommends because it pins the Node.js version and makes updates a two-command job.

### Where does n8n store its data?

In `/home/node/.n8n` inside the container, which this setup maps to the `n8n_data` volume. With SQLite, that directory holds the whole database. With PostgreSQL, it still holds the config file with the encryption key (if you didn't set `N8N_ENCRYPTION_KEY`) and other local files.

### What is WEBHOOK_URL in n8n?

It's the public base URL n8n uses when it builds webhook addresses. Behind a reverse proxy, n8n can't work out its public URL on its own, so without `WEBHOOK_URL` it shows internal `localhost:5678` URLs, and services like Stripe or GitHub can't reach them.

### How much RAM does n8n need?

It depends on your workflows. Light use runs fine on a small VPS, but workflows that process large files or big JSON payloads hold that data in memory. Watch `docker stats` and move to a larger server or queue mode if executions get killed.

## Conclusion

Self-hosting n8n takes a Docker Compose file bound to `127.0.0.1:5678`, a few environment variables (`N8N_HOST`, `N8N_PROTOCOL=https`, `WEBHOOK_URL`, and a fixed `N8N_ENCRYPTION_KEY`), and an Nginx server block that proxies WebSockets. Add HTTPS, claim the owner account immediately, and schedule backups of the database and data volume. Because a dead n8n can't alert you about itself, put an outside check from [GreenCrew](https://greencrew.space) on `/healthz`.
