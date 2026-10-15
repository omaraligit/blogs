# How to Set Up Nginx Load Balancing with Failover (upstream, least_conn, and Health Checks)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Load Balancing, High Availability, DevOps, Reliability

Description: To load balance with Nginx, list your backend servers in an upstream block, choose a method such as least_conn, set max_fails=3 fail_timeout=30s for passive health checks, and send traffic with proxy_pass http://backend. Covers failover with backup servers, sticky sessions, retries, and monitoring each backend.

Date: 2026-10-07

**Short answer:** define an `upstream backend { least_conn; server 10.0.0.11:3000 max_fails=3 fail_timeout=30s; server 10.0.0.12:3000 max_fails=3 fail_timeout=30s; }` block and point `proxy_pass http://backend;` at it. Add `proxy_next_upstream error timeout http_502 http_503;` so a failed request is retried on the next server. Nginx open source spreads traffic across healthy backends and stops sending requests to a server that keeps failing. It can't actively probe backends, though; that's an NGINX Plus feature. So you also need external monitoring on each backend to know when one is down. This guide covers every piece, with copy-paste configs.

## What Is Nginx Load Balancing?

Load balancing spreads incoming requests across several backend servers, so no single server is overloaded and one failure doesn't take your site down. Nginx load balances HTTP, HTTPS, gRPC, and, with the `stream` module, raw TCP and UDP traffic. It does this with the [ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html), which ships with every Nginx build.

A typical layout:

```text
                 ┌──────────────┐
 clients ──────► │    Nginx     │
                 │ (TLS, LB)    │
                 └──┬───┬───┬───┘
                    │   │   │
           ┌────────┘   │   └────────┐
           ▼            ▼            ▼
      app-1:3000   app-2:3000   app-3:3000 (backup)
```

## Step 1: Define the Upstream Group

Create `/etc/nginx/conf.d/upstream.conf`:

```nginx
upstream backend {
    least_conn;
    zone backend 64k;

    server 10.0.0.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.13:3000 backup;

    keepalive 32;
}
```

| Line | What it does |
| --- | --- |
| `least_conn` | Sends each request to the server with the fewest active connections |
| `zone backend 64k` | Shares backend state (failures, connection counts) across all worker processes |
| `max_fails=3 fail_timeout=30s` | After 3 failed attempts within 30 s, mark the server unavailable for 30 s |
| `backup` | Only receives traffic when all primary servers are unavailable |
| `keepalive 32` | Keeps up to 32 idle connections open per worker to reuse |

## Step 2: Send Traffic to the Upstream

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection        "";
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 3s;
        proxy_read_timeout    30s;

        proxy_next_upstream error timeout http_502 http_503 http_504;
        proxy_next_upstream_tries 2;
        proxy_next_upstream_timeout 10s;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

If you're new to the proxy headers above, our guide on [setting up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy) explains each one.

## Step 3: Choose a Load Balancing Method

| Method | Directive | How it picks a server | Best for |
| --- | --- | --- | --- |
| Round robin | *(default, no directive)* | Each server in turn, respecting `weight` | Identical, stateless servers |
| Least connections | `least_conn;` | Fewest active connections | Requests with very different durations |
| IP hash | `ip_hash;` | Hash of the client IPv4 /24 or full IPv6 | Simple sticky sessions |
| Generic hash | `hash $request_uri consistent;` | Hash of any variable | Cache affinity, sharding |
| Random | `random two least_conn;` | Picks two at random, sends to the less loaded | Many Nginx instances balancing the same pool |
| Least time | `least_time header;` | Lowest response time | **NGINX Plus only** |

### Weighted Servers

When backends have different capacity, give the larger one more traffic:

```nginx
upstream backend {
    server 10.0.0.11:3000 weight=3;   # gets ~3x the requests
    server 10.0.0.12:3000 weight=1;
}
```

### Sticky Sessions

If your app keeps sessions in local memory, a user must keep hitting the same server. Use `ip_hash`, or hash on a session cookie for more even spread:

```nginx
upstream backend {
    hash $cookie_sessionid consistent;
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
}
```

The better long-term fix is to move sessions into Redis or a database, so any server can handle any request and failover is seamless.

## How Failover Works: Passive Health Checks

Nginx open source uses **passive** health checks. It doesn't probe backends. It watches real requests, and when a request to a server fails (connection refused, timeout, or a status listed in `proxy_next_upstream`), it counts a failure against that server.

| Setting | Default | Effect |
| --- | --- | --- |
| `max_fails` | `1` | Failures within `fail_timeout` before the server is marked down |
| `fail_timeout` | `10s` | Window for counting failures **and** how long the server stays marked down |
| `max_fails=0` | n/a | Disables failure tracking for that server |
| `proxy_next_upstream` | `error timeout` | Which failures trigger a retry on the next server |
| `proxy_next_upstream_tries` | `0` (unlimited) | Maximum number of servers to try per request |

Two details matter in production:

1. **POST requests aren't retried by default.** Since Nginx 1.9.13, non-idempotent requests (POST, LOCK, PATCH) are only passed to the next server if you add `non_idempotent` to `proxy_next_upstream`. Leave it off unless the endpoints are safe to replay, or a payment could be charged twice.
2. **A "down" server comes back automatically.** After `fail_timeout`, Nginx sends a real user request to the server to test it. If the server is still broken, that user gets the error or a retry.

### Passive vs Active Health Checks

| | Passive (Nginx open source) | Active (NGINX Plus `health_check`) |
| --- | --- | --- |
| Detection | Only when real traffic fails | Background probes on a schedule |
| Users affected | Some requests fail or retry before the server is marked down | Usually none |
| Detects a broken idle backend | No | Yes |
| Cost | Free | Commercial license |

Passive checks have a blind spot. A backend that's down but receives no traffic, such as a `backup` server, can sit broken for weeks without anyone noticing. Then the day you need failover, it fails too.

## Step 4: Close the Gap with External Backend Monitoring

Fill the gap left by passive checks by giving every backend its own health URL and monitoring each one from outside. One option is a separate port per backend in Nginx:

```nginx
# Health check passthrough per backend (restrict or protect as needed)
server {
    listen 8081;
    location = /healthz { proxy_pass http://10.0.0.11:3000/healthz; }
}
server {
    listen 8082;
    location = /healthz { proxy_pass http://10.0.0.12:3000/healthz; }
}
server {
    listen 8083;
    location = /healthz { proxy_pass http://10.0.0.13:3000/healthz; }
}
```

Then monitor the load-balanced site **and** each backend:

| Monitor | URL | Tells you |
| --- | --- | --- |
| Public site | `https://example.com/healthz` | What users actually experience |
| Backend 1 | `http://lb.example.com:8081/healthz` | app-1 is healthy |
| Backend 2 | `http://lb.example.com:8082/healthz` | app-2 is healthy |
| Backup | `http://lb.example.com:8083/healthz` | Your failover target will work when needed |

[GreenCrew](https://greencrew.space) runs these checks from outside your infrastructure. It alerts you on Discord, Slack, or Telegram when any backend fails, even while the public site still looks fine because Nginx is routing around the problem. That's the alert you want: fix app-2 while app-1 is still carrying the load, before you're down to zero healthy servers. Your public status page can show a "degraded performance" incident instead of a full outage. For the alert setup, see [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Step 5: See Which Backend Served Each Request

Add upstream details to your access log so you can debug uneven load or a failing node:

```nginx
log_format upstream_log '$remote_addr [$time_local] "$request" $status '
                        'upstream=$upstream_addr ustatus=$upstream_status '
                        'rt=$request_time urt=$upstream_response_time';

access_log /var/log/nginx/access.log upstream_log;
```

```bash
# Requests per backend in the last 1,000 lines
tail -n 1000 /var/log/nginx/access.log | grep -o 'upstream=[^ ]*' | sort | uniq -c
```

When a retry happens, `$upstream_addr` lists both servers, for example `10.0.0.11:3000, 10.0.0.12:3000`, which shows failover in action.

## Zero-Downtime Deploys: Drain a Server

To take a backend out for maintenance, mark it `down` and reload:

```nginx
server 10.0.0.12:3000 down;
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Existing requests finish on the old workers, and new requests skip the server. Deploy, remove `down`, reload again, then repeat for the next server. This is the simplest form of a rolling deploy.

## Troubleshooting Nginx Load Balancing

| Symptom | Cause | Fix |
| --- | --- | --- |
| All traffic goes to one server | `ip_hash` with most users behind one NAT or CDN | Use `least_conn`, or hash on a cookie |
| Users randomly logged out | Sessions in local memory, no stickiness | Add `hash $cookie_...` or move sessions to Redis |
| `no live upstreams while connecting to upstream` | Every server marked failed | Check backends; raise `max_fails` if they're flapping |
| Duplicate orders or payments | `non_idempotent` retries enabled | Remove `non_idempotent` from `proxy_next_upstream` |
| `invalid parameter "backup"` | `backup` used with `hash`, `ip_hash`, or `random` | Use round robin or `least_conn` with backup servers |
| Backup server broken during failover | Passive checks never tested it | Monitor the backup's health URL externally |

## Frequently Asked Questions

### Does Nginx open source support active health checks?

No. The `health_check` directive is only in the commercial NGINX Plus. Nginx open source uses passive checks through `max_fails` and `fail_timeout`. To catch failures before users do, monitor each backend's health endpoint with an external uptime monitor.

### What is the default load balancing method in Nginx?

Weighted round robin. With no method directive, Nginx sends requests to each server in turn, adjusted by any `weight` values. For workloads where some requests take much longer than others, `least_conn` usually spreads the load better.

### How many backend servers can Nginx load balance?

There's no practical hard limit. Nginx can balance across hundreds of upstream servers. Your limits are the backend capacity, network bandwidth, and the size of the shared `zone`, which you should increase for large upstream groups.

### Is Nginx a good alternative to HAProxy or a cloud load balancer?

For HTTP workloads, yes. Nginx combines load balancing with TLS termination, caching, compression, and rate limiting in one process. HAProxy has active health checks in its free version and more detailed load balancing stats. Cloud load balancers remove server management entirely but cost more and offer less control.

## Conclusion

Nginx load balancing needs an `upstream` block, a method such as `least_conn`, sensible `max_fails` and `fail_timeout` values, and careful `proxy_next_upstream` retries. Its one real gap in the open source version is that it can't see an idle or backup backend failing. Close that gap by monitoring every backend from outside with [GreenCrew](https://greencrew.space), so you get an alert and a status page update while you still have healthy servers left. To harden the same entry point, add [rate limiting](/blog/how-to-set-up-rate-limiting-in-nginx) and [Gzip and Brotli compression](/blog/how-to-enable-gzip-and-brotli-compression-in-nginx).
