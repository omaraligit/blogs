# How to Deploy a Node.js App in Production with PM2 and Nginx (Ubuntu, Startup Script, and Zero-Downtime Reloads)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Node.js, PM2, Linux, DevOps

Description: To run a Node.js app in production, start it with pm2 start app.js --name myapp, make it survive reboots with pm2 startup and pm2 save, and put Nginx in front with proxy_pass http://127.0.0.1:3000. This guide covers installing Node.js, an ecosystem.config.js file, cluster mode, zero-downtime reloads, logs, and HTTPS.

Date: 2026-10-15

**Short answer:** Don't run `node app.js` in an SSH session and hope for the best. Install Node.js from an LTS release, run the app with **PM2** (`pm2 start app.js --name myapp`), then run `pm2 startup` and `pm2 save` so it restarts after crashes and reboots. Bind the app to `127.0.0.1:3000` and put **Nginx** in front with `proxy_pass http://127.0.0.1:3000;`, so Nginx handles ports 80/443, TLS, and static files. Ship new code with `pm2 reload myapp` so users don't see downtime.

## Why You Need PM2 and Nginx for Node.js in Production

A Node.js process is a single program. If it throws an uncaught exception, it exits. If the server reboots, nothing starts it again. And if you bind it to port 80 directly, it has to run as root. A production setup fixes all three:

| Problem | `node app.js` alone | With PM2 + Nginx |
| --- | --- | --- |
| App crashes | Site is down until you notice | PM2 restarts it automatically |
| Server reboots | App doesn't start | `pm2 startup` registers a systemd service |
| Ports 80/443 | Needs root or extra capabilities | Nginx listens; Node stays on a high port as a normal user |
| HTTPS | Certificate handling in app code | Nginx terminates TLS with Certbot certificates |
| Multiple CPU cores | One core used | PM2 cluster mode runs one process per core |
| Deploys | Kill and restart, dropped requests | `pm2 reload` restarts processes one at a time |
| Logs | Lost when the terminal closes | Written to `~/.pm2/logs/` |

PM2 is a process manager for Node.js. It isn't the only option (a plain systemd unit works too, compared below), but it's the most common because it adds clustering, reloads, and log handling with very little setup.

## Prerequisites

- An Ubuntu server (24.04 or 22.04) with a sudo user. Don't run the app as root.
- Nginx installed ([how to install Nginx on Ubuntu](/blog/how-to-install-nginx-on-ubuntu))
- A domain pointed at the server if you want HTTPS

## Step 1: Install Node.js (LTS)

Ubuntu's own `nodejs` package often lags behind the current LTS release. Install from [NodeSource](https://github.com/nodesource/distributions) instead:

```bash
curl -fsSL https://deb.nodesource.com/setup_lts.x -o nodesource_setup.sh
sudo -E bash nodesource_setup.sh
sudo apt install nodejs
node -v
npm -v
```

`setup_lts.x` tracks the current LTS line. To pin a major version, use its script instead (for example `setup_22.x`). Check the [Node.js release schedule](https://nodejs.org/en/about/previous-releases) and stick to **LTS** versions in production.

**Alternative:** [nvm](https://github.com/nvm-sh/nvm) installs Node per user and makes switching versions easy. It works with PM2, but note that `pm2 startup` records the Node path at the time you run it, so re-run it after you switch versions.

## Step 2: Prepare the Application

Clone your app into a directory owned by your deploy user and install production dependencies:

```bash
sudo mkdir -p /var/www/myapp
sudo chown $USER:$USER /var/www/myapp
git clone https://github.com/you/myapp.git /var/www/myapp
cd /var/www/myapp
npm ci --omit=dev
```

`npm ci` installs exactly what's in `package-lock.json`. Building TypeScript or a frontend usually needs dev dependencies, so in that case run `npm ci`, then `npm run build`, then `npm prune --omit=dev`.

Make the app listen on localhost only, so the only way in is through Nginx:

```js
// app.js
const express = require('express');
const app = express();

app.set('trust proxy', 'loopback'); // trust X-Forwarded-* from Nginx

app.get('/health', (req, res) => res.json({ status: 'ok' }));
app.get('/', (req, res) => res.send('Hello from Node.js behind Nginx'));

const port = process.env.PORT || 3000;
app.listen(port, '127.0.0.1', () => console.log(`Listening on ${port}`));
```

`trust proxy` makes Express read the real client IP and protocol from Nginx's `X-Forwarded-For` and `X-Forwarded-Proto` headers. Without it, `req.ip` is always `127.0.0.1` and `req.secure` is always false. See the [Express behind proxies guide](https://expressjs.com/en/guide/behind-proxies.html).

## Step 3: Install PM2 and Start the App

```bash
sudo npm install -g pm2
cd /var/www/myapp
pm2 start app.js --name myapp
pm2 list
curl http://127.0.0.1:3000/health
```

### Use an ecosystem.config.js File

Command-line flags are easy to forget on the next deploy. Put the config in the repo instead:

```js
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: 'myapp',
      script: './app.js',
      cwd: '/var/www/myapp',
      instances: 'max',          // one process per CPU core
      exec_mode: 'cluster',
      max_memory_restart: '400M',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
    },
  ],
};
```

```bash
pm2 delete myapp
pm2 start ecosystem.config.js
```

| Option | What it does |
| --- | --- |
| `name` | The name you use in `pm2 reload myapp`, `pm2 logs myapp` |
| `script` | Entry file (or `npm` with `args: 'start'`) |
| `instances` | Number of processes; `'max'` means one per CPU core |
| `exec_mode` | `fork` (single process) or `cluster` (load-balanced across instances) |
| `max_memory_restart` | Restart a process that grows past this memory, which contains leaks |
| `env` | Environment variables; keep real secrets out of git |
| `watch` | Restart on file changes. Useful in development, **leave it off in production** |

**Cluster mode caveat:** each instance is a separate process with its own memory. In-memory sessions, caches, or WebSocket state won't be shared, so move them to Redis or a database before you scale past one instance. The full option list is in the [PM2 ecosystem file docs](https://pm2.keymetrics.io/docs/usage/application-declaration/).

## Step 4: Start PM2 on Boot

```bash
pm2 startup systemd
```

PM2 prints a command that starts with `sudo env PATH=...`. **Copy and run that exact command.** It creates a systemd unit (`pm2-<user>.service`) for your user. Then save the current process list so it's restored at boot:

```bash
pm2 save
```

Run `pm2 save` again whenever you add or remove apps. Test it properly with `sudo reboot`, then `pm2 list` once you're back in.

## Step 5: Put Nginx in Front as a Reverse Proxy

Create `/etc/nginx/sites-available/myapp`:

```nginx
upstream myapp {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://myapp;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
curl -I http://example.com
```

`keepalive 32` together with `proxy_http_version 1.1` and an empty `Connection` header reuses connections to Node instead of opening a new one for every request. If your app uses WebSockets (Socket.IO, for example), add a location that passes the `Upgrade` and `Connection: upgrade` headers and raises `proxy_read_timeout`. The [Nginx WebSocket proxying docs](https://nginx.org/en/docs/http/websocket.html) show the exact lines.

Header details, timeouts, and buffering are covered in [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy).

### Serve Static Files with Nginx

Nginx is much better at serving files than `express.static`. If your app has a `public/` directory:

```nginx
location /static/ {
    alias /var/www/myapp/public/;
    expires 7d;
    access_log off;
}
```

## Step 6: Add HTTPS

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot adds the `listen 443 ssl` lines and an HTTP → HTTPS redirect. Because Nginx sends `X-Forwarded-Proto https` and Express trusts it, `req.secure` and secure cookies work without any TLS code in Node. Full guide: [Let's Encrypt on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

## Step 7: Deploy Updates with Zero Downtime

```bash
cd /var/www/myapp
git pull
npm ci --omit=dev
pm2 reload ecosystem.config.js --update-env
```

| Command | What happens | Downtime |
| --- | --- | --- |
| `pm2 restart myapp` | Kills all processes, then starts them | Brief, requests fail during the restart |
| `pm2 reload myapp` | In cluster mode, restarts instances one at a time | None if you run 2+ instances |
| `pm2 reload ... --update-env` | Same, and picks up changed environment variables | None |
| `pm2 stop myapp` / `pm2 delete myapp` | Stops it / removes it from the list | Full |

For truly graceful reloads, have the app stop accepting connections and finish in-flight requests when it receives `SIGINT`, then exit:

```js
const server = app.listen(port, '127.0.0.1');
process.on('SIGINT', () => {
  server.close(() => process.exit(0));
});
```

## Step 8: Manage Logs

```bash
pm2 logs myapp           # tail stdout and stderr
pm2 logs myapp --lines 200
pm2 monit                # live CPU/memory per process
pm2 install pm2-logrotate
```

Without rotation, `~/.pm2/logs/` grows until the disk fills, and a full disk takes down far more than your Node app. `pm2-logrotate` rotates files by size and keeps a limited number of old ones.

## PM2 vs systemd vs Docker

| Option | Good for | Trade-off |
| --- | --- | --- |
| **PM2** | Single servers, quick setup, cluster mode, easy reloads | One more global tool to keep updated |
| **systemd unit** | Minimal setups, one process, no extra dependencies | No built-in clustering or zero-downtime reload |
| **Docker / Compose** | Reproducible builds, multiple services | More moving parts; Nginx proxies to the container port |

All three work behind the same Nginx config. Only the `upstream` address changes.

## Troubleshooting Node.js Behind Nginx

| Symptom | Cause | Fix |
| --- | --- | --- |
| 502 Bad Gateway, `connect() failed (111: Connection refused)` | App isn't running or listens on another port | `pm2 list`, `pm2 logs`, check `PORT` matches `upstream` |
| App status `errored` in `pm2 list` | Crash on startup (missing env var, missing module) | `pm2 logs myapp --err --lines 100` |
| App gone after reboot | `pm2 startup` command not run, or `pm2 save` skipped | Run the printed `sudo env PATH=...` command, then `pm2 save` |
| `pm2: command not found` after a Node upgrade | Global packages belong to the old Node | `sudo npm install -g pm2`, then `pm2 update` |
| `req.ip` is always `127.0.0.1` | Express doesn't trust the proxy | `app.set('trust proxy', 'loopback')` |
| 504 Gateway Timeout on long requests | Request exceeds `proxy_read_timeout` (60s default) | Speed up the handler or raise the timeout for that location |
| WebSocket connections fail | `Upgrade` headers not forwarded | Add the WebSocket headers from the Nginx docs |
| Restart loop every few minutes | `max_memory_restart` too low or a real memory leak | `pm2 monit`, raise the limit, fix the leak |
| Sessions randomly lost | In-memory sessions with cluster mode | Use a shared store like Redis |

More causes of gateway errors are in [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor the App from Outside

PM2 restarts crashed processes, but it can't tell you when the app is running and broken: a dependency is down, every request returns 500, or a crash loop is restarting it faster than it can serve traffic. From inside the server, all of that looks "online."

[GreenCrew](https://greencrew.space) checks your site over HTTP from outside your network and sends alerts to Discord, Slack, or Telegram when it fails, plus a public status page for your users. Point the check at your `/health` route through Nginx (`https://example.com/health`), so a failure in Node, Nginx, TLS, or DNS all trigger it. The setup is in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Do I need Nginx if I use PM2?

They do different jobs. PM2 keeps Node running, and Nginx handles the internet-facing work: ports 80/443, TLS, static files, compression, [rate limiting](/blog/how-to-set-up-rate-limiting-in-nginx), and buffering slow clients. You can skip Nginx for internal tools, but for a public site the pair is the standard setup.

### How many PM2 instances should I run?

Start with `instances: 'max'` (one per core) if the app is stateless and CPU-bound. On a small VPS with 1–2 GB of RAM, memory often runs out before CPU does, so set a fixed number like `2` and watch `pm2 monit`. Running at least two instances is what makes `pm2 reload` zero-downtime.

### Should I run PM2 as root?

No. Run it as a normal deploy user that owns the app directory. Nginx binds the privileged ports, so Node never needs root, and a compromised app gets far less access to the system.

### Where do I put environment variables and secrets?

Use the `env` block in `ecosystem.config.js` for non-secret settings. For secrets, load a `.env` file that isn't committed (for example with Node's `--env-file` flag on versions that support it, or the `dotenv` package), restrict its permissions with `chmod 600`, and run `pm2 reload --update-env` after changing it.

## Conclusion

A production Node.js deployment on Ubuntu has three layers: an LTS Node.js runtime, PM2 to keep the app alive across crashes and reboots, and Nginx in front for TLS and traffic. Bind Node to localhost, register PM2 with `pm2 startup` and `pm2 save`, deploy with `pm2 reload`, and rotate your logs. Then add an external check with [GreenCrew](https://greencrew.space) so you find out the app is down before your users do.
