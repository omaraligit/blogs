# How to Set Up Rate Limiting in Nginx (limit_req, burst, and 429 Responses)

Author: [oaitbenali](https://www.github.com/oaitbenali)

Tags: Nginx, Rate Limiting, Security, DevOps, API

Description: To rate limit in Nginx, define a zone with limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s in the http block, apply it with limit_req zone=perip burst=20 nodelay, and return 429 with limit_req_status 429. Copy-paste configs for login pages, APIs, allowlists, connection limits, and safe testing.

Date: 2026-10-07

**Short answer:** add `limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;` to the `http` block. Put `limit_req zone=perip burst=20 nodelay;` in the `location` you want to protect, and set `limit_req_status 429;` so clients get a proper "Too Many Requests" response instead of the default 503. Run `sudo nginx -t`, then `sudo systemctl reload nginx`. That stops brute-force logins, scrapers, and runaway API clients from taking down your app. The rest of this guide explains how `burst` and `nodelay` really behave, gives configs for common cases, and shows how to roll out limits without blocking real users.

## What Is Rate Limiting in Nginx?

Rate limiting caps how many requests a client can make in a given time window. Nginx does this with the built-in [ngx_http_limit_req_module](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html), which uses the **leaky bucket** algorithm. Requests fill a bucket and drain out at a fixed rate. Requests that arrive while the bucket is full are rejected. No extra module or package is needed, because it's compiled into every standard Nginx build.

Nginx has two related modules:

| Module | Directive | Limits | Use it for |
| --- | --- | --- | --- |
| `limit_req` | `limit_req_zone`, `limit_req` | Requests per second or minute | Logins, APIs, search, expensive endpoints |
| `limit_conn` | `limit_conn_zone`, `limit_conn` | Simultaneous open connections | Downloads, slow clients, WebSockets |

## Step 1: Define a Rate Limit Zone

Zones go in the `http` context, usually `/etc/nginx/nginx.conf` or a file in `/etc/nginx/conf.d/`:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=perip:10m rate=10r/s;
    limit_req_status 429;
    limit_req_log_level warn;
}
```

| Part | Meaning |
| --- | --- |
| `$binary_remote_addr` | The key: the client IP in compact binary form (4 bytes for IPv4, 16 for IPv6) |
| `zone=perip:10m` | A shared memory zone named `perip`, 10 MB in size |
| `rate=10r/s` | 10 requests per second per key; use `r/m` for per-minute limits |
| `limit_req_status 429` | Return `429 Too Many Requests` instead of the default `503` |
| `limit_req_log_level warn` | Log rejections at `warn` level (default `error`) |

Per the Nginx docs, a 1 MB zone holds about 16,000 states with `$binary_remote_addr`, so `10m` tracks roughly 160,000 unique IPs. When the zone fills up, Nginx evicts the oldest entries. If it still can't make room, it rejects new requests, so size the zone to match your traffic.

Always use `$binary_remote_addr`, not `$remote_addr`. The text form uses more memory per entry.

## Step 2: Apply the Limit to a Location

```nginx
server {
    server_name example.com;

    location /api/ {
        limit_req zone=perip burst=20 nodelay;
        proxy_pass http://127.0.0.1:3000;
    }
}
```

Reload and confirm:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## How burst and nodelay Actually Work

Most rate limiting confusion comes from three settings. Nginx enforces the rate at millisecond granularity. `rate=10r/s` really means **one request every 100 ms**, not "10 requests at any time during the second". Browsers load pages in bursts, so a limit without `burst` breaks real users.

| Config | 25 requests arrive at once | Result |
| --- | --- | --- |
| `limit_req zone=perip;` | 1 served, 24 rejected | Too strict for browsers |
| `limit_req zone=perip burst=20;` | 1 served now, 20 queued and released every 100 ms, 4 rejected | Smooth but adds latency (up to 2 s) |
| `limit_req zone=perip burst=20 nodelay;` | 21 served immediately, 4 rejected; burst slots refill at 10/s | **Best default for most sites** |
| `limit_req zone=perip burst=20 delay=8;` | 9 served immediately, 12 queued, 4 rejected | Two-stage: fast for normal use, throttles heavy clients |

`burst=20 nodelay` lets normal page loads through at full speed and still caps sustained abuse at 10 requests per second. The `delay=` parameter needs Nginx 1.15.7 or newer.

## Copy-Paste Configs for Common Cases

### Protect a Login Page from Brute Force

Logins should be far stricter than the rest of the site:

```nginx
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    location = /login {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://127.0.0.1:3000;
    }
}
```

That allows 5 quick attempts, then one every 12 seconds, which makes password guessing impractical.

### Limit an API by API Key Instead of IP

Clients behind a corporate NAT or a mobile carrier share an IP address. For authenticated APIs, limit by key:

```nginx
map $http_x_api_key $api_limit_key {
    default $http_x_api_key;
    ""      $binary_remote_addr;   # no key: fall back to IP
}

limit_req_zone $api_limit_key zone=apikey:20m rate=50r/s;
```

### Stack Multiple Limits

You can apply several zones at once, and Nginx enforces the strictest one:

```nginx
limit_req_zone $binary_remote_addr zone=perip:10m  rate=10r/s;
limit_req_zone $server_name        zone=perhost:1m rate=500r/s;

location /api/ {
    limit_req zone=perip   burst=20  nodelay;
    limit_req zone=perhost burst=200 nodelay;
    proxy_pass http://127.0.0.1:3000;
}
```

The per-IP limit stops a single abuser. The per-server limit protects the backend from many IPs at once.

### Allowlist Trusted IPs

Requests with an **empty key are not counted**, so map trusted networks to an empty string:

```nginx
geo $rate_limited {
    default        1;
    10.0.0.0/8     0;   # internal network
    203.0.113.10   0;   # office IP
}

map $rate_limited $limit_key {
    0 "";
    1 $binary_remote_addr;
}

limit_req_zone $limit_key zone=perip:10m rate=10r/s;
```

## Limit Simultaneous Connections with limit_conn

Request rate doesn't cover clients that open many slow, long-lived connections. Add `limit_conn` for those:

```nginx
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn_status 429;

location /downloads/ {
    limit_conn addr 2;      # max 2 parallel downloads per IP
    limit_rate 2m;          # 2 MB/s per connection
}
```

## Rate Limiting Behind a CDN or Load Balancer

If Nginx sits behind Cloudflare, AWS ALB, or another proxy, `$binary_remote_addr` is the **proxy's** IP. You'd end up rate limiting everyone as a single client. Restore the real client IP with the [realip module](https://nginx.org/en/docs/http/ngx_http_realip_module.html) before applying limits:

```nginx
set_real_ip_from 10.0.0.0/8;          # your load balancer range
real_ip_header   X-Forwarded-For;
real_ip_recursive on;
# For Cloudflare: list Cloudflare's published IP ranges and use
# real_ip_header CF-Connecting-IP;
```

Only trust headers from IP ranges you control or have verified. Otherwise attackers can spoof `X-Forwarded-For` to dodge your limits.

## Roll Out Safely with Dry Run Mode

Turning on a strict limit in production without testing is a good way to block your own customers. Nginx 1.17.1 added a dry run mode that logs what would be rejected without rejecting anything:

```nginx
location /api/ {
    limit_req zone=perip burst=20 nodelay;
    limit_req_dry_run on;
    proxy_pass http://127.0.0.1:3000;
}
```

Watch the error log for a few days:

```bash
sudo grep "limiting requests" /var/log/nginx/error.log | tail
# ... limiting requests, dry run, excess: 20.540 by zone "perip", client: 198.51.100.7 ...
```

If only bots and scrapers show up, remove `limit_req_dry_run` and reload.

## Test Your Rate Limit

Send a quick burst and count the status codes:

```bash
for i in $(seq 1 40); do
  curl -s -o /dev/null -w "%{http_code}\n" https://example.com/api/health
done | sort | uniq -c
#   21 200
#   19 429
```

With `rate=10r/s burst=20 nodelay`, you should see about 21 `200` responses and the rest `429`.

## Troubleshooting Nginx Rate Limits

| Symptom | Cause | Fix |
| --- | --- | --- |
| Real users get 429s on page load | No `burst`, or burst too small | Add `burst=20 nodelay` or larger |
| Everyone is limited together | Nginx sees the CDN or load balancer IP | Configure `set_real_ip_from` and `real_ip_header` |
| Limit has no effect | `limit_req` set in a parent block but overridden in the location | Directives are inherited only if the child block has none; repeat them |
| Clients get 503, not 429 | `limit_req_status` not set | Add `limit_req_status 429;` |
| `zone "perip" is unknown` | `limit_req_zone` missing or outside `http` | Define the zone in the `http` block |
| Error log full of rejections | Logging at `error` level | Set `limit_req_log_level warn;` or `notice` |

## Don't Rate Limit Your Own Monitoring

Your uptime monitor and health checks should never get a 429, or you'll get false "down" alerts. Keep health endpoints in their own `location` with no `limit_req`:

```nginx
location = /healthz {
    access_log off;
    proxy_pass http://127.0.0.1:3000/healthz;
}
```

Rate limits protect your uptime, but a misconfigured limit can cause an outage of its own when it blocks a real traffic spike or a partner integration. [GreenCrew](https://greencrew.space) monitors your site and API endpoints from outside your network. If a new limit starts returning errors to clients, you get an alert on Discord, Slack, or Telegram right away, and your public status page keeps customers informed. For the full alerting setup, see [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### What is a good rate limit for Nginx?

For general website traffic, `rate=10r/s` with `burst=20 nodelay` per IP is a safe starting point. For login and password-reset endpoints use around `5r/m`. For APIs, base the limit on your documented quota and measure real usage with `limit_req_dry_run` before enforcing it.

### Should Nginx return 429 or 503 for rate-limited requests?

Return 429. `429 Too Many Requests` tells clients and crawlers that they're being throttled and should slow down, while `503 Service Unavailable` suggests your server is down. Set `limit_req_status 429;` and `limit_conn_status 429;`.

### Does Nginx rate limiting stop DDoS attacks?

It reduces the impact of small, application-layer floods from a limited number of IPs. It doesn't stop large volumetric or distributed attacks, because the traffic still reaches your server and uses its bandwidth. For those, use an upstream DDoS protection service or CDN.

### Is the rate limit shared between Nginx worker processes?

Yes. The zone lives in shared memory, so every worker on the same server sees the same counters. It isn't shared between separate Nginx servers. Each node behind a load balancer enforces its own limit.

## Conclusion

Nginx rate limiting comes down to a zone, a `limit_req` line, and the right `burst` value. Use `burst` with `nodelay` so browsers aren't punished, return 429 instead of 503, restore real client IPs behind a CDN, and try new limits in dry run mode first. Pair it with a hardened [Nginx reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy) and external monitoring from [GreenCrew](https://greencrew.space) so you know right away when a limit, or an attack, affects real users.
