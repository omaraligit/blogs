# How to Set Up Nginx Caching with proxy_cache (Config, Cache Bypass, and Serving Stale Content)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Caching, Performance, Reverse Proxy, DevOps

Description: To set up Nginx caching, define a cache with proxy_cache_path /var/cache/nginx/app levels=1:2 keys_zone=app_cache:10m max_size=1g inactive=60m; in the http block, then add proxy_cache app_cache; and proxy_cache_valid 200 10m; to your proxied location. This guide covers the full config, cache bypass for logged-in users, microcaching, serving stale content when the backend is down, and purging.

Date: 2026-10-08

**Short answer:** add a `proxy_cache_path` line to the `http` block to create a cache on disk with a shared memory zone. Then, in the `location` that uses `proxy_pass`, turn it on with `proxy_cache app_cache;` and set how long to keep responses with `proxy_cache_valid 200 301 10m;`. Add `add_header X-Cache-Status $upstream_cache_status;` so you can see `HIT` and `MISS` with `curl -I`. Bypass the cache for logged-in users with `proxy_cache_bypass` and `proxy_no_cache`, and use `proxy_cache_use_stale` so Nginx keeps serving cached pages when the backend fails.

## What Is Nginx proxy_cache?

When Nginx works as a [reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy), every request normally goes through to your application. With `proxy_cache`, Nginx saves the backend's response to disk and serves later requests for the same URL itself, without touching the app until the cached copy expires.

| Benefit | What happens | Why it matters |
| --- | --- | --- |
| Faster responses | Cached hits skip the app and database | Lower time to first byte |
| Less backend load | Only misses reach the app | Survive traffic spikes on smaller servers |
| Resilience | Stale content is served when the backend fails | Visitors see pages instead of a 502 |
| Simple setup | Built into Nginx, no extra service | No Varnish or Redis layer to run |

It works best for pages that are the same for every visitor: blogs, docs, marketing pages, product listings, and public API responses. It's a bad fit for anything personalized unless you bypass the cache for those requests.

## Step 1: Define the Cache with proxy_cache_path

`proxy_cache_path` is only allowed in the `http` context. On most installs, put it in a file under `/etc/nginx/conf.d/`, which is included inside `http`:

```nginx
# /etc/nginx/conf.d/cache.conf
proxy_cache_path /var/cache/nginx/app
                 levels=1:2
                 keys_zone=app_cache:10m
                 max_size=1g
                 inactive=60m
                 use_temp_path=off;
```

Create the directory and make sure the Nginx worker user owns it:

```bash
sudo mkdir -p /var/cache/nginx/app
sudo chown -R www-data:www-data /var/cache/nginx   # use "nginx" on RHEL-based systems
```

### What Each proxy_cache_path Parameter Does

| Parameter | Meaning | Guidance |
| --- | --- | --- |
| `/var/cache/nginx/app` | Directory where cached responses are stored | Fast local disk; not a network mount |
| `levels=1:2` | Two-level subdirectory structure | Keeps any single directory from holding too many files |
| `keys_zone=app_cache:10m` | Name and size of the shared memory zone for keys | Per the [nginx docs](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_path), 1 MB holds about 8,000 keys |
| `max_size=1g` | Upper limit on disk usage | The cache manager evicts least-recently-used items above this |
| `inactive=60m` | Remove items not requested for this long | Applies even if the item is still "valid" |
| `use_temp_path=off` | Write files directly into the cache directory | Avoids an extra copy between filesystems |

`inactive` and validity are different things. `proxy_cache_valid` decides when a cached item is *stale*. `inactive` decides when an unused item is *deleted*. A response cached for 10 minutes that nobody requests for 60 minutes is removed from disk.

## Step 2: Enable Caching in Your Location

Now use the zone in your server block:

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    # ssl_certificate lines here

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache app_cache;
        proxy_cache_valid 200 301 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_key $scheme$host$request_uri;
        proxy_cache_lock on;

        add_header X-Cache-Status $upstream_cache_status always;
    }
}
```

Test and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### What Each Caching Directive Does

| Directive | What it does | Default |
| --- | --- | --- |
| `proxy_cache` | Turns caching on with the named zone | `off` |
| `proxy_cache_valid` | How long to cache responses, per status code | Not set; caching relies on upstream headers |
| `proxy_cache_key` | What makes two requests "the same" | `$scheme$proxy_host$request_uri` |
| `proxy_cache_lock` | Only one request fills a missing entry; others wait | `off` |
| `proxy_cache_methods` | Which request methods can be cached | `GET HEAD` |
| `proxy_cache_min_uses` | Requests needed before a response is cached | `1` |

The default key uses `$proxy_host`, which is the upstream address (`127.0.0.1:3000`), not your domain. If one backend serves several domains, use `$host` in the key as shown above, or pages from different sites can get mixed up.

## Step 3: Check That the Cache Works

Request the same URL twice and look at the `X-Cache-Status` header:

```bash
curl -sI https://example.com/ | grep -i x-cache-status
# x-cache-status: MISS
curl -sI https://example.com/ | grep -i x-cache-status
# x-cache-status: HIT
```

Here's what each `$upstream_cache_status` value means:

| Value | Meaning |
| --- | --- |
| `MISS` | Not in cache; fetched from the backend (and stored, if cacheable) |
| `HIT` | Served from cache |
| `EXPIRED` | Cached copy was stale; fetched a fresh one |
| `STALE` | Served a stale copy because of `proxy_cache_use_stale` |
| `UPDATING` | Served stale while another request refreshes the entry |
| `REVALIDATED` | Stale copy confirmed still valid via `proxy_cache_revalidate` |
| `BYPASS` | Cache skipped because of `proxy_cache_bypass` |

If you keep getting `MISS`, the backend is probably telling Nginx not to cache. See the troubleshooting table below.

## Step 4: Respect (or Override) Upstream Cache Headers

Nginx listens to your application. By default it won't cache a response that has:

- A `Set-Cookie` header
- `Cache-Control: private`, `no-cache`, or `no-store`
- `Vary: *`

When the upstream sends `Cache-Control: max-age`, `Expires`, or `X-Accel-Expires`, those take priority over `proxy_cache_valid`. That's usually what you want, because the app knows which pages are safe to cache.

If your app sends `no-cache` on everything and you know the pages are public, you can tell Nginx to ignore those headers for a specific location:

```nginx
location /blog/ {
    proxy_pass http://127.0.0.1:3000;
    proxy_cache app_cache;
    proxy_cache_valid 200 10m;
    proxy_ignore_headers Cache-Control Expires Set-Cookie;
    proxy_hide_header Set-Cookie;
}
```

Be careful here. Ignoring `Set-Cookie` and caching the response can send one visitor's session cookie to everyone. Use this only on paths that never set per-user cookies, and hide the header as shown.

## Step 5: Bypass the Cache for Logged-In Users and APIs

Logged-in users, carts, dashboards, and authenticated APIs must never be served from a shared cache. Use a `map` (in the `http` context) to flag those requests:

```nginx
# /etc/nginx/conf.d/cache.conf (http context)
map $http_cookie $skip_cache {
    default         0;
    ~*session       1;
    ~*wordpress_logged_in 1;
}
```

Then skip both reading and writing the cache for them:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_cache app_cache;
    proxy_cache_valid 200 10m;

    proxy_cache_bypass $skip_cache $http_authorization;
    proxy_no_cache     $skip_cache $http_authorization;

    add_header X-Cache-Status $upstream_cache_status always;
}

location /admin/ {
    proxy_pass http://127.0.0.1:3000;
    proxy_cache off;
}
```

| Directive | Effect when its value is non-empty and not "0" |
| --- | --- |
| `proxy_cache_bypass` | Don't *read* from the cache; go to the backend |
| `proxy_no_cache` | Don't *save* this response into the cache |

You almost always want both. With only `proxy_cache_bypass`, a logged-in user's personalized response can still be saved and later served to anonymous visitors.

## Step 6: Serve Stale Content When the Backend Is Down

This is where caching turns into a reliability feature. With `proxy_cache_use_stale`, Nginx returns the last good copy instead of an error when the backend fails:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_cache app_cache;
    proxy_cache_valid 200 10m;

    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_background_update on;
    proxy_cache_lock on;
    proxy_cache_revalidate on;
}
```

| Directive | What it adds |
| --- | --- |
| `proxy_cache_use_stale error timeout http_502 ...` | Serve stale pages when the backend errors or times out |
| `updating` | Serve stale while a fresh copy is being fetched |
| `proxy_cache_background_update on` | Refresh expired items in the background, so visitors get the stale copy instantly |
| `proxy_cache_lock on` | Only one request goes to the backend for a missing item |
| `proxy_cache_revalidate on` | Use `If-Modified-Since` / `If-None-Match` to refresh cheaply |

Stale content is only served for URLs that are already in the cache. Pages nobody visited recently will still return a 502. To fix those errors at the source, see [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Step 7: Microcaching for Dynamic Sites

If your pages change often but get heavy traffic, cache them for just one second. Under load, that turns hundreds of identical requests per second into about one backend request per URL per second:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_cache app_cache;
    proxy_cache_valid 200 1s;
    proxy_cache_use_stale updating;
    proxy_cache_background_update on;
    proxy_cache_lock on;

    proxy_cache_bypass $skip_cache;
    proxy_no_cache     $skip_cache;
}
```

Visitors see content that's at most a second or two old, and a traffic spike hits Nginx instead of your database. Microcaching pairs well with [Nginx load balancing](/blog/how-to-set-up-nginx-load-balancing-with-failover) and [rate limiting](/blog/how-to-set-up-rate-limiting-in-nginx) for handling bursts.

## Step 8: Clear (Purge) the Nginx Cache

Open source Nginx doesn't include a purge API. The `proxy_cache_purge` directive is an NGINX Plus feature. Your options:

| Method | How | When to use |
| --- | --- | --- |
| Clear everything | `sudo rm -rf /var/cache/nginx/app/*` | After a deploy; Nginx refills the cache as requests come in |
| Short validity | Keep `proxy_cache_valid` short (1–10 minutes) | Content updates often, purging isn't worth the effort |
| Bypass header | `proxy_cache_bypass $http_x_refresh;` restricted to your IPs | Force-refresh one URL: the fresh response replaces the cached one |
| Third-party module | `ngx_cache_purge` | You need per-URL purges and can build modules |

The bypass trick works because a bypassed request still gets saved to the cache (unless `proxy_no_cache` also matches). Restrict it so random visitors can't hammer your backend:

```nginx
geo $can_refresh {
    default 0;
    203.0.113.10 1;   # your office or CI runner
}
map $can_refresh$http_x_refresh $force_refresh {
    default 0;
    ~^1.+   1;
}
# in the location:
proxy_cache_bypass $force_refresh;
```

```bash
curl -sI -H "X-Refresh: 1" https://example.com/blog/my-post | grep -i x-cache-status
# x-cache-status: BYPASS
```

## Troubleshooting Nginx proxy_cache

| Symptom | Cause | Fix |
| --- | --- | --- |
| Always `MISS`, never `HIT` | Upstream sends `Set-Cookie` or `Cache-Control: no-cache/private` | Fix headers in the app, or `proxy_ignore_headers` on safe paths |
| No `X-Cache-Status` header at all | `add_header` in a nested block canceled inheritance, or `proxy_cache` isn't on for that location | Put `add_header` in the same location as `proxy_cache` |
| One user's content shown to others | Personalized pages cached | Add `proxy_cache_bypass` and `proxy_no_cache` for session cookies and `Authorization` |
| Wrong site's page served | Several domains share one backend and the key lacks `$host` | `proxy_cache_key $scheme$host$request_uri;` |
| `nginx -t` error: "proxy_cache_path" directive is not allowed here | Put inside `server` or `location` | Move it to the `http` context |
| Permission errors in error.log | Cache directory not owned by the worker user | `chown` the directory to `www-data` or `nginx` |
| Disk filling up | No `max_size`, or it's too large for the disk | Set `max_size` below free space |
| POST requests not cached | Only `GET` and `HEAD` are cached by default | Intended; don't cache POST |

## Monitor the Origin, Not Just the Cache

Caching creates a monitoring blind spot. With `proxy_cache_use_stale`, your homepage can keep returning `200 OK` from cache for hours while the app behind it is completely down. Visitors on popular pages see nothing wrong, while everyone trying to log in, check out, or load an uncached page gets errors.

Monitor both layers:

| Monitor | URL | Catches |
| --- | --- | --- |
| HTTP status | `https://example.com/` | Nginx down, TLS or DNS failures |
| HTTP status | `https://example.com/healthz` (with `proxy_cache off`) | App down while the cache hides it |
| HTTP status | A key uncached route like `/login` | Failures in the parts of the site cache can't cover |

Make sure the health endpoint is never cached:

```nginx
location = /healthz {
    proxy_pass http://127.0.0.1:3000;
    proxy_cache off;
}
```

[GreenCrew](https://greencrew.space) checks these URLs from outside your network and alerts you on Discord, Slack, or Telegram as soon as one fails, even while the cache is still keeping the homepage up. A public status page lets you tell users which parts of the site are affected. To set up the chat side, see [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Does Nginx cache responses by default?

No. Proxy caching is off until you define a zone with `proxy_cache_path` and turn it on with `proxy_cache` in a `server` or `location` block. Nginx will happily serve static files from disk without caching, but it doesn't store responses from proxied apps unless you configure it.

### What's the difference between proxy_cache and browser caching?

`proxy_cache` stores responses on your server and shares them across all visitors. Browser caching, controlled by `Cache-Control` and `expires`, stores files in each visitor's browser. Use both: long browser caching for versioned static assets, and `proxy_cache` for HTML and API responses generated by your app.

### Should I use Nginx caching or Varnish?

For most sites, Nginx's built-in cache does the job without adding another service. Varnish offers a more flexible configuration language and built-in purging, which helps on large sites with complex invalidation rules. If you already run Nginx and your rules are simple, start with `proxy_cache`.

### Is fastcgi_cache the same as proxy_cache?

They work the same way and use almost identical directives (`fastcgi_cache_path`, `fastcgi_cache_valid`, `fastcgi_cache_bypass`). Use `fastcgi_cache` when Nginx talks to PHP-FPM directly with `fastcgi_pass`, and `proxy_cache` when it talks to an HTTP backend with `proxy_pass`.

## Conclusion

Setting up Nginx caching takes one `proxy_cache_path` line in the `http` block and a few directives in your proxied location. Add `X-Cache-Status` so you can see hits and misses, bypass the cache for sessions and `Authorization` headers, and use `proxy_cache_use_stale` with background updates so visitors get pages even when the backend fails. Then monitor an uncached `/healthz` endpoint with [GreenCrew](https://greencrew.space), so the cache can't hide an outage from you. For the full directive reference, see the [ngx_http_proxy_module documentation](https://nginx.org/en/docs/http/ngx_http_proxy_module.html).
