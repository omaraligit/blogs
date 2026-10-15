# How to Set Up Nginx as a Reverse Proxy (proxy_pass Config, Headers, and Health Checks)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Reverse Proxy, DevOps, Linux, Web Server

Description: To set up Nginx as a reverse proxy, add a server block with proxy_pass http://127.0.0.1:3000, forward the Host and X-Forwarded-* headers, run nginx -t, and reload. Includes a copy-paste config, WebSocket support, timeouts, a health check endpoint, and uptime monitoring.

Date: 2026-09-30

**Short answer:** install Nginx and create a server block for your domain. Point `proxy_pass` at the port your app listens on, forward the original `Host`, client IP and protocol headers, then check the config with `sudo nginx -t` and apply it with `sudo systemctl reload nginx`. The whole job takes about ten minutes. The rest of this guide covers the config you'll actually want in production: WebSockets, timeouts, a health check endpoint, and outside monitoring so you know when the proxy or the app behind it goes down.

## What Is an Nginx Reverse Proxy?

A reverse proxy is a server that sits in front of one or more applications. It takes client requests and passes them to the right backend. Nginx accepts traffic on ports 80 and 443, handles TLS, compression, and buffering, and sends each request to an app running on a private port, such as a Node.js server on `127.0.0.1:3000`. Clients never talk to the app directly.

Common reasons to put Nginx in front of an app:

| Benefit | What Nginx does | Why it matters |
| --- | --- | --- |
| TLS termination | Handles HTTPS certificates in one place | The app can stay plain HTTP on localhost |
| Single entry point | Routes `api.example.com` and `app.example.com` to different ports | One server can host many services |
| Buffering | Absorbs slow clients before they reach the app | Protects single-threaded runtimes |
| Security | Hides backend ports, limits request size and rate | Smaller attack surface |
| Zero-downtime reloads | `nginx -s reload` swaps config without dropping connections | Safe config changes |

## Step 1: Install Nginx

On Ubuntu or Debian:

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
nginx -v
```

On RHEL, Rocky, or AlmaLinux use `sudo dnf install -y nginx`. Then open the firewall if it's enabled:

```bash
sudo ufw allow 'Nginx Full'   # opens ports 80 and 443
```

Visit `http://your-server-ip`. If you see the "Welcome to nginx!" page, the server is running.

## Step 2: Make Sure Your App Listens on Localhost

Your backend should listen on the loopback interface, not on `0.0.0.0`, so the only way in is through Nginx. Check it:

```bash
curl -I http://127.0.0.1:3000
sudo ss -tlnp | grep 3000
```

If `ss` shows `0.0.0.0:3000` or `*:3000`, bind the app to `127.0.0.1` or block the port with your firewall.

## Step 3: Create the Reverse Proxy Server Block

Create `/etc/nginx/sites-available/example.com`:

```nginx
upstream app_backend {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    client_max_body_size 10m;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection        "";

        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }
}
```

Enable it and remove the default site:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

`nginx -t` has to print `syntax is ok` and `test is successful` before you reload. Reloading keeps existing connections alive. Restarting drops them.

### What Each Proxy Directive Does

| Directive | Purpose | What breaks without it |
| --- | --- | --- |
| `proxy_pass` | Sends the request to the backend | Nothing is proxied |
| `proxy_http_version 1.1` | Uses HTTP/1.1 to the upstream | Keepalive and WebSockets fail |
| `Host $host` | Passes the domain the client asked for | Virtual hosts and redirects in the app use the wrong host |
| `X-Real-IP` / `X-Forwarded-For` | Passes the real client IP | Logs and rate limits all show `127.0.0.1` |
| `X-Forwarded-Proto` | Tells the app whether the client used HTTPS | Redirect loops and insecure cookie flags |
| `Connection ""` | Clears the header so upstream keepalive works | A new TCP connection per request |
| `proxy_read_timeout` | Maximum time to wait for the backend response | Slow endpoints return `504 Gateway Timeout` |

### The Trailing-Slash Rule in proxy_pass

This is the most common reverse proxy mistake. Whether you put a URI after the upstream address changes the path the backend receives:

| Config | Request | Backend receives |
| --- | --- | --- |
| `location /api/ { proxy_pass http://app; }` | `/api/users` | `/api/users` |
| `location /api/ { proxy_pass http://app/; }` | `/api/users` | `/users` |
| `location /api/ { proxy_pass http://app/v2/; }` | `/api/users` | `/v2/users` |

If your app is mounted at the root but exposed at `/api/`, use the trailing slash to strip the prefix.

## Step 4: Add WebSocket Support

Socket.IO, Next.js dev servers, GraphQL subscriptions, and live dashboards need the `Upgrade` handshake passed through. Add a `map` in the `http` context, at the top of your site file:

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      "";
}
```

Then in the location:

```nginx
location /socket.io/ {
    proxy_pass http://app_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host       $host;
    proxy_read_timeout 3600s;
}
```

The long `proxy_read_timeout` stops Nginx from closing idle sockets after the 60-second default.

## Step 5: Add a Health Check Endpoint

A health endpoint gives your uptime monitor something cheap and reliable to request. Add two locations. One proves Nginx itself is answering. The other proves the whole path through to your app works:

```nginx
# Nginx is alive (does not touch the app)
location = /nginx-health {
    access_log off;
    default_type text/plain;
    return 200 "ok\n";
}

# Full path: Nginx -> app -> dependencies
location = /healthz {
    access_log off;
    proxy_pass http://app_backend/healthz;
    proxy_set_header Host $host;
}
```

Your app's `/healthz` handler should check what really matters, such as the database connection or cache, and return a non-200 status when one of them fails. When `/nginx-health` is up but `/healthz` is down, you know right away the problem is behind the proxy.

For connection-level metrics, enable the built-in [stub_status module](https://nginx.org/en/docs/http/ngx_http_stub_status_module.html) and restrict it to localhost:

```nginx
location = /nginx_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```

```bash
curl http://127.0.0.1/nginx_status
# Active connections: 3
# server accepts handled requests
#  1204 1204 5310
# Reading: 0 Writing: 1 Waiting: 2
```

If `accepts` and `handled` stop matching, Nginx is dropping connections, usually because it has hit `worker_connections`.

## Step 6: Add HTTPS

A reverse proxy on plain HTTP is only halfway done. Get a free certificate from Let's Encrypt with Certbot, which edits this server block for you and sets up auto-renewal:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

For the full walkthrough, including renewal testing and rate limits, see our guide on [how to install a free Let's Encrypt SSL certificate on Nginx](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

## Troubleshooting Common Nginx Reverse Proxy Errors

| Error | Most likely cause | Fix |
| --- | --- | --- |
| `502 Bad Gateway` | App is down or listening on a different port | `curl 127.0.0.1:3000`, check `sudo tail -f /var/log/nginx/error.log` |
| `502` with `connect() ... (13: Permission denied)` | SELinux blocks Nginx from connecting out | `sudo setsebool -P httpd_can_network_connect 1` |
| `504 Gateway Timeout` | Backend slower than `proxy_read_timeout` | Raise the timeout or fix the slow endpoint |
| `413 Request Entity Too Large` | Upload larger than `client_max_body_size` (1 MB default) | Raise `client_max_body_size` |
| Redirect loop after HTTPS | App doesn't trust `X-Forwarded-Proto` | Enable "trust proxy" in your framework |
| All logs show `127.0.0.1` | Missing forwarded IP headers | Add `X-Real-IP` and `X-Forwarded-For` |

Two commands solve most problems:

```bash
sudo nginx -t                        # validate config
sudo tail -n 50 /var/log/nginx/error.log
```

## Step 7: Monitor the Proxy From Outside

A 502 page is still an HTTP response, and a server that has lost its network sends nothing at all. Neither will page you unless something outside the server is watching. A check running on the same machine can't report that the machine is gone.

Set up external monitoring with at least these checks:

| Monitor | URL | Catches |
| --- | --- | --- |
| HTTP status | `https://example.com/nginx-health` | Nginx down, DNS broken, server offline, TLS errors |
| HTTP status | `https://example.com/healthz` | App crashed, database unreachable, 502/504 |
| Keyword | `https://example.com/` expects your page title | Blank pages, error pages served with a 200 status |

[GreenCrew](https://greencrew.space) monitors these endpoints from outside your network. It alerts you on Discord, Slack, or Telegram the moment a check fails, and gives you a public status page to share with users during an incident. Create an HTTP monitor for each URL in the table above and you'll hear about the next 502 before your users do.

## Frequently Asked Questions

### What is the difference between a reverse proxy and a load balancer?

A reverse proxy forwards client requests to backend servers. A load balancer is a reverse proxy that spreads those requests across several backends. Nginx does both. Add more `server` lines to the `upstream` block and it load-balances with round-robin by default.

### Should I use sites-available or conf.d?

Both work. Debian and Ubuntu packages include `sites-enabled/*` from `nginx.conf`, while the official nginx.org packages and RHEL-based systems use `/etc/nginx/conf.d/*.conf`. Pick whichever your `nginx.conf` includes and stay consistent.

### Do I need to restart Nginx after changing the config?

No. Run `sudo nginx -t`, then `sudo systemctl reload nginx`. A reload starts new worker processes with the new config and shuts down the old workers gracefully, so users see no downtime. Restart only after upgrading the Nginx binary or changing modules.

### How do I proxy multiple apps on one server?

Create one `server` block per domain, each with its own `server_name` and `proxy_pass` port. For example, `app.example.com` goes to `127.0.0.1:3000` and `api.example.com` goes to `127.0.0.1:4000`. Nginx picks the block by matching the `Host` header.

## Conclusion

An Nginx reverse proxy comes down to a `server` block, a `proxy_pass` line, and the right forwarded headers. Getting it production-ready takes more: WebSocket upgrades, sensible timeouts, a health endpoint, HTTPS, and outside monitoring. Test every change with `nginx -t`, reload instead of restarting, and point a [GreenCrew](https://greencrew.space) monitor at `/healthz` so a broken backend gets fixed within minutes.
