# How to Fix 502 Bad Gateway in Nginx (Error Log Messages, Causes, and Fixes)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Troubleshooting, Reverse Proxy, Linux, DevOps

Description: To fix a 502 Bad Gateway in Nginx, run sudo tail -n 50 /var/log/nginx/error.log, find the upstream error (connection refused, permission denied, upstream sent too big header), and fix the backend, port, socket, or buffer setting it points to. This guide maps every common error log message to its cause and a copy-paste fix.

Date: 2026-10-08

**Short answer:** a 502 Bad Gateway means Nginx is up but couldn't get a valid response from the backend it proxies to (Node.js, PHP-FPM, Gunicorn, a Docker container). Run `sudo tail -n 50 /var/log/nginx/error.log` and read the line that ends in `while connecting to upstream` or `while reading response header from upstream`. Most of the time it says `Connection refused`, which means the app is down or listening on a different port than your `proxy_pass`. Start or fix the app, confirm it answers with `curl -i http://127.0.0.1:3000`, and the 502 goes away. The rest of this guide covers every other error message and how to fix it.

## What Does 502 Bad Gateway Mean in Nginx?

When Nginx acts as a [reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy), it sits between the browser and your application. A 502 is Nginx telling the client: "I'm working, but the server behind me failed." The problem is almost never Nginx itself. It's the connection to the upstream, or what the upstream sent back.

That makes the Nginx error log the single most useful tool here. Nginx records exactly why the upstream request failed, along with the upstream address it tried.

### 502 vs 503 vs 504

These three get confused often, and each points to a different fix:

| Status | What Nginx is saying | Typical cause |
| --- | --- | --- |
| `502 Bad Gateway` | The upstream refused, reset, or sent an invalid response | App crashed, wrong port or socket, headers too large, upstream TLS failure |
| `503 Service Unavailable` | Nginx (or the app) is refusing to serve right now | Rate limiting with `limit_req_status 503` (the default), maintenance mode, app overload |
| `504 Gateway Timeout` | The upstream accepted the connection but didn't answer in time | Slow query, `proxy_read_timeout` too low, upstream unreachable on the network |

If you're seeing 504s instead, the fix is usually timeouts or a slow endpoint, not a dead process. If you're seeing 503s on busy endpoints, check your [Nginx rate limiting](/blog/how-to-set-up-rate-limiting-in-nginx) config first.

## Step 1: Read the Nginx Error Log

Open the log and reproduce the error in your browser:

```bash
sudo tail -f /var/log/nginx/error.log
```

If your server block sets its own `error_log`, check that file instead:

```bash
sudo nginx -T 2>/dev/null | grep -E "error_log|access_log"
```

A typical 502 line looks like this:

```text
2026/10/08 09:14:02 [error] 2201#2201: *57 connect() failed (111: Connection refused) while connecting to upstream, client: 203.0.113.9, server: example.com, request: "GET / HTTP/2.0", upstream: "http://127.0.0.1:3000/", host: "example.com"
```

The two parts that matter are the error text (`connect() failed (111: Connection refused)`) and the `upstream:` address. Together they tell you what failed and where.

## Step 2: Match the Error Message to the Cause

Find your log message in this table:

| Error log message | Cause | Fix |
| --- | --- | --- |
| `connect() failed (111: Connection refused) while connecting to upstream` | Nothing is listening on that host and port | Start the app; make `proxy_pass` match the port it really uses |
| `connect() to unix:/run/php/php-fpm.sock failed (2: No such file or directory)` | Socket path in Nginx doesn't match PHP-FPM's `listen` | Point `fastcgi_pass` at the real socket path |
| `connect() to unix:... failed (13: Permission denied)` | Nginx's user can't open the socket | Fix `listen.owner` / `listen.group` in the FPM pool |
| `connect() to 127.0.0.1:3000 failed (13: Permission denied)` | SELinux blocks Nginx from making network connections | `sudo setsebool -P httpd_can_network_connect 1` |
| `upstream prematurely closed connection while reading response header from upstream` | App crashed or killed the request mid-response | Check app logs for crashes, out-of-memory kills, worker timeouts |
| `recv() failed (104: Connection reset by peer)` | Upstream reset the connection | Same as above; also check keepalive mismatches |
| `upstream sent too big header while reading response header from upstream` | Response headers (often cookies) exceed the proxy buffer | Raise `proxy_buffer_size` / `fastcgi_buffer_size` |
| `no live upstreams while connecting to upstream` | Every server in the `upstream` block is marked failed | Fix the backends; review `max_fails` and `fail_timeout` |
| `SSL_do_handshake() failed ... while SSL handshaking to upstream` | HTTPS upstream needs SNI or rejects the handshake | Add `proxy_ssl_server_name on;` |
| `... could not be resolved (3: Host not found)` | Upstream hostname in a variable can't be resolved at runtime | Set a working `resolver` or use an IP |

The sections below walk through the fixes for each one.

## Step 3: Fix "Connection Refused" (The App Isn't Listening)

This causes the large majority of 502s. Check whether anything is listening on the port from your `proxy_pass`:

```bash
sudo ss -ltnp | grep 3000
curl -i http://127.0.0.1:3000/
```

If `ss` shows nothing, the app is down. Check its service and logs:

```bash
sudo systemctl status myapp
sudo journalctl -u myapp -n 100 --no-pager
```

If the app *is* listening but on a different address, make Nginx match it. Common mismatches:

| App listens on | Your `proxy_pass` | Result |
| --- | --- | --- |
| `127.0.0.1:8080` | `http://127.0.0.1:3000` | Refused: wrong port |
| `127.0.0.1:3000` | `http://localhost:3000` | Usually works, but Nginx may try `[::1]` first and log refused errors |
| `[::1]:3000` only | `http://127.0.0.1:3000` | Refused: app is IPv6-only |
| `127.0.0.1:3000` inside a Docker container | `http://127.0.0.1:3000` on the host | Refused: container localhost isn't host localhost |

Use an explicit IP in `proxy_pass` instead of `localhost` to avoid the IPv4/IPv6 ambiguity:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
}
```

### Docker and Docker Compose

Inside a container, the app must listen on `0.0.0.0`, not `127.0.0.1`, or nothing outside the container can reach it. If Nginx runs on the host, publish the port (`-p 127.0.0.1:3000:3000`) and proxy to `127.0.0.1:3000`. If Nginx runs in the same Compose network, proxy to the service name:

```nginx
location / {
    proxy_pass http://app:3000;   # "app" is the Compose service name
}
```

Keep in mind that Nginx resolves that hostname when it starts. If the `app` container gets a new IP after a restart, Nginx keeps using the old one until you reload it.

## Step 4: Fix PHP-FPM Socket Errors

For PHP sites, the 502 usually comes from the FastCGI socket. First find the socket path PHP-FPM actually uses:

```bash
grep -R "^listen" /etc/php/*/fpm/pool.d/ 2>/dev/null   # Debian/Ubuntu
grep -R "^listen" /etc/php-fpm.d/ 2>/dev/null          # RHEL/Rocky/Alma
ls -l /run/php/
```

Then make `fastcgi_pass` match it exactly:

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;      # Debian/Ubuntu helper
    fastcgi_pass unix:/run/php/php-fpm.sock; # must match the pool's "listen"
}
```

If the error is `Permission denied`, set the socket owner in the pool file to the user Nginx runs as (`www-data` on Debian/Ubuntu, `nginx` on RHEL-based systems):

```ini
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

Restart PHP-FPM, then reload Nginx. Also check the FPM log for `server reached pm.max_children setting`. If all PHP workers are busy, requests queue up and can fail or time out under load. Raise `pm.max_children` only if the server has the memory for it.

## Step 5: Fix "Upstream Sent Too Big Header"

Large cookies, long JWTs, or many `Set-Cookie` headers can overflow the buffer Nginx uses for the response header. The default is one memory page (4 KB or 8 KB depending on the platform). Raise it in the `location` or `server` block:

```nginx
# Reverse proxy (proxy_pass)
proxy_buffer_size 16k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;

# PHP-FPM (fastcgi_pass)
fastcgi_buffer_size 16k;
fastcgi_buffers 8 16k;
```

`proxy_busy_buffers_size` has to be at least as large as `proxy_buffer_size` and smaller than the total of `proxy_buffers` minus one buffer, or `nginx -t` will refuse the config. If you need more than 32k for headers, look at why the app is sending that much in the first place.

## Step 6: Fix Crashes, Resets, and Premature Closes

`upstream prematurely closed connection` and `Connection reset by peer` mean the backend accepted the request and then dropped it. Nginx is just the messenger. Look at the app side:

```bash
sudo journalctl -u myapp --since "10 min ago"
sudo dmesg -T | grep -i -E "killed process|out of memory"
```

| What you find | What it means | Fix |
| --- | --- | --- |
| `Out of memory: Killed process` | The kernel's OOM killer stopped your app | Add memory or swap, limit worker count, fix the leak |
| Gunicorn `WORKER TIMEOUT` | Worker exceeded Gunicorn's `--timeout` | Raise the timeout or make the endpoint faster |
| Stack trace, then process exit | Unhandled exception crashed the app | Fix the bug; run under systemd with `Restart=on-failure` |
| Errors only after idle periods | Upstream closes keepalive connections before Nginx does | Keep the app's keepalive timeout longer than Nginx's |

## Step 7: Fix Load Balancer and HTTPS Upstream 502s

`no live upstreams` means Nginx has marked every server in the `upstream` block as unavailable after repeated failures. Fix the backends first. Then check that `max_fails` and `fail_timeout` aren't so aggressive that one slow second takes everything out. Our [Nginx load balancing with failover](/blog/how-to-set-up-nginx-load-balancing-with-failover) guide covers those settings in detail.

If you proxy to an HTTPS backend, such as a managed platform or another load balancer, Nginx doesn't send SNI by default. Many hosts reject the handshake without it:

```nginx
location / {
    proxy_pass https://backend.example.net;
    proxy_ssl_server_name on;
    proxy_set_header Host backend.example.net;
}
```

## Step 8: Reload and Verify the Fix

After any change:

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -sS -o /dev/null -w "%{http_code}\n" https://example.com/
```

You want `200` (or whatever your homepage normally returns). Keep `tail -f` running on the error log for a few minutes to confirm the upstream errors have stopped, not just paused.

## Troubleshooting Checklist for Nginx 502 Errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| 502 on every page right after deploy | App failed to start or bound a new port | `systemctl status`, check the port in the app config |
| 502 only on some pages | Those routes crash or return huge headers | Check app logs and buffer settings |
| 502 only under load | Worker pool exhausted or OOM kills | Scale workers, add memory, add caching |
| 502 after server reboot | App service not enabled at boot | `sudo systemctl enable myapp` |
| 502 only on RHEL/Rocky/Alma | SELinux blocking network connections | `setsebool -P httpd_can_network_connect 1` |
| 502 intermittently, every few minutes | Keepalive mismatch or a crash loop | Compare timeouts; check restart counts in journald |
| Error log is empty | Wrong log file, or the 502 comes from a CDN | `nginx -T` to find the log; check the `Server` header in the response |

That last row catches people out. If you use Cloudflare or another CDN, the 502 page might come from the CDN failing to reach Nginx, not from Nginx failing to reach your app. Check the response headers and the CDN's error page to see which hop failed.

## Monitor for 502s Before Your Users Report Them

A 502 is the most common way a site "goes down" while the server itself stays up. Ping checks and CPU graphs look fine. Nginx is running. Only a real HTTP request shows the problem. That's why you need an HTTP check that runs from outside your network and alerts on the status code.

Set up these monitors:

| Monitor | URL | Catches |
| --- | --- | --- |
| HTTP status | `https://example.com/` | 502/504 on the homepage, Nginx down, DNS or TLS failures |
| HTTP status | `https://example.com/healthz` | App crashed or database unreachable, even if the homepage is cached |
| HTTP status | Each critical route (`/login`, `/api/health`) | Route-specific crashes and oversized headers |

[GreenCrew](https://greencrew.space) runs HTTP checks against these URLs from outside your server. It alerts you on Discord, Slack, or Telegram when one starts returning 502, and gives you a public status page so users know you're on it. If you haven't wired up chat alerts yet, see [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Is a 502 Bad Gateway my fault or the server's?

If you run the site, it's a server-side problem. The browser and the visitor's network aren't involved. Nginx couldn't get a valid response from your backend. If you're a visitor, the site owner has to fix it. Refreshing helps only when the cause is a brief restart.

### Why does restarting Nginx not fix the 502?

Because Nginx usually isn't what's broken. Restarting it doesn't start a crashed app, fix a wrong socket path, or free up PHP workers. Restart or fix the upstream service and use `sudo systemctl reload nginx` only after you change Nginx's config.

### Can a 502 be caused by a firewall?

Yes, when the upstream is on another machine. If `curl http://10.0.0.12:3000` from the Nginx host hangs or is refused, check the backend's firewall (`ufw`, `firewalld`, cloud security groups). A blocked connection often shows up as a timeout (504), and an actively rejected one shows up as connection refused (502).

### How do I show a custom 502 error page in Nginx?

Use `error_page` with an internal location:

```nginx
error_page 502 503 504 /50x.html;
location = /50x.html {
    root /usr/share/nginx/html;
    internal;
}
```

It improves the user experience, but it doesn't fix anything. Keep monitoring the real status code.

## Conclusion

Fixing a 502 Bad Gateway in Nginx always starts in the same place: `/var/log/nginx/error.log`. The upstream error message tells you whether the app is down, on the wrong port or socket, blocked by permissions or SELinux, crashing mid-request, or sending headers that are too big. Fix the backend, run `nginx -t`, reload, and verify with `curl`. Then put a [GreenCrew](https://greencrew.space) HTTP monitor on your homepage and health endpoint so the next 502 reaches your Discord, Slack, or Telegram before your users notice it. For the official reference on every proxy directive mentioned here, see the [ngx_http_proxy_module docs](https://nginx.org/en/docs/http/ngx_http_proxy_module.html).
