# How to A/B Test with Nginx (split_clients, Sticky Cookies, and Variant Logging)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, A/B Testing, Analytics, Traffic Splitting, DevOps

Description: To A/B test with Nginx, use split_clients "${remote_addr}${http_user_agent}" $variant { 50% "a"; * "b"; } in the http block, then route each variant to a different root or upstream and log $variant in your access log. This guide covers sticky cookies, targeting users by device, country, or language, measuring conversions from logs, and avoiding SEO and caching mistakes.

Date: 2026-10-13

**Short answer:** Nginx has a built-in A/B testing tool, the `split_clients` directive. It hashes a value you choose (such as the client IP plus user agent) and assigns each visitor to a bucket by percentage: `split_clients "${remote_addr}${http_user_agent}" $variant { 50% "a"; * "b"; }`. Use `$variant` to pick a different `root`, `upstream`, or header, store it in a cookie so visitors keep the same version, and add it to your `log_format` so you can count conversions per variant. Because the split happens before your app runs, you can test whole page designs or backend versions with no JavaScript and no flicker.

## What Is A/B Testing in Nginx?

A/B testing shows two (or more) versions of a page to different visitors and compares how each one performs, for example how many visitors reach a signup confirmation page. Most A/B tools do the split in the browser with JavaScript. Doing it in Nginx means the split happens **at the server**, before any HTML is sent.

The [ngx_http_split_clients_module](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html) is compiled into standard Nginx builds. It hashes a string with MurmurHash2 and maps the result to a value based on percentages. The same input always lands in the same bucket.

| Approach | Where the split happens | Pros | Cons |
| --- | --- | --- | --- |
| JavaScript testing tool | Browser | Visual editors, built-in stats | Page flicker, blocked by ad blockers, adds page weight |
| App feature flags | Application code | Fine-grained, per-user targeting | Needs code changes in every app |
| **Nginx `split_clients`** | Web server | No flicker, works for any backend, can test entire releases | You build the measurement yourself |

Nginx-level testing works best for landing page redesigns, testing a new backend version (a canary release), or comparing two static builds of a site.

## Step 1: Define the Split with split_clients

Add this to the `http` block, for example in `/etc/nginx/conf.d/ab-test.conf`:

```nginx
split_clients "${remote_addr}${http_user_agent}" $split_variant {
    50%     "a";
    *       "b";
}
```

| Part | Meaning |
| --- | --- |
| `"${remote_addr}${http_user_agent}"` | The string that gets hashed. The same string always gets the same variant |
| `$split_variant` | The variable that receives the result |
| `50% "a";` | The first 50% of the hash range gets `a` |
| `* "b";` | Everything left over gets `b` |

Percentages can have decimals (`0.5%`) and must add up to 100% or less. Use `*` for the remainder. A three-way test looks like this:

```nginx
split_clients "${remote_addr}${http_user_agent}" $split_variant {
    33.3%   "a";
    33.3%   "b";
    *       "c";
}
```

### Pick the Right Hash Key

The key decides who sees what. Choose it based on what should stay stable:

| Key | Behavior | Good for |
| --- | --- | --- |
| `${remote_addr}` | Same variant per IP | Simple tests; everyone behind one office NAT gets the same version |
| `${remote_addr}${http_user_agent}` | Spreads users behind shared IPs | Most website tests |
| `${cookie_uid}` | Same variant per logged-in user or visitor ID | Apps that already set a user ID cookie |
| `${request_id}` | New random variant on every request | Load splitting only; useless for A/B tests because users flip versions |

IPs change, especially on mobile networks. To keep a visitor on the same variant reliably, store it in a cookie (Step 3).

## Step 2: Serve Different Content per Variant

### Option A: Two Static Builds

If each variant is a separate build of your site, switch the document root:

```nginx
server {
    listen 80;
    server_name example.com;

    root /var/www/example.com/variant-$ab_variant;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

With `/var/www/example.com/variant-a/` and `/var/www/example.com/variant-b/`, each visitor gets their own build. (`$ab_variant` is the sticky variable defined in Step 3.)

### Option B: Two Backend Versions

To compare two versions of an application, route to different upstreams:

```nginx
upstream app_a { server 127.0.0.1:3000; }
upstream app_b { server 127.0.0.1:3001; }

server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app_$ab_variant;
        proxy_set_header Host $host;
        proxy_set_header X-AB-Variant $ab_variant;
    }
}
```

When `proxy_pass` uses a variable that matches the name of an `upstream` group, Nginx uses that group directly, without needing a `resolver`. The [reverse proxy guide](/blog/how-to-set-up-nginx-reverse-proxy) covers the other `proxy_set_header` lines you'll want.

### Option C: One Backend, Variant as a Header

If your app can render both versions itself, just tell it which one to show:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header X-AB-Variant $ab_variant;
}
```

The app reads `X-AB-Variant` and can pass the value to its own analytics, so every event is tagged with the variant.

### Option D: Test Only One Page

Limit the experiment to a landing page and leave everything else alone:

```nginx
location = /pricing {
    try_files /pricing-$ab_variant.html =404;
}
```

## Step 3: Make Variants Sticky with a Cookie

Without a cookie, a visitor whose IP changes can jump between versions, which ruins your results. Store the assignment on the first visit and reuse it after that:

```nginx
# http block
split_clients "${remote_addr}${http_user_agent}" $split_variant {
    50%     "a";
    *       "b";
}

map $cookie_ab $ab_variant {
    "a"     "a";
    "b"     "b";
    default $split_variant;   # no cookie or invalid value: assign one
}
```

```nginx
# server block
add_header Set-Cookie "ab=$ab_variant; Path=/; Max-Age=2592000; SameSite=Lax; Secure" always;
```

The `map` only accepts known values, so a visitor can't put an arbitrary string into the cookie and have it end up in a file path or upstream name. `Max-Age=2592000` keeps the assignment for 30 days. Drop `Secure` if the site isn't on HTTPS yet.

To force a variant while you test, set the cookie yourself:

```bash
curl -s -H "Cookie: ab=b" https://example.com/ | grep -i "<title>"
```

## Step 4: Log the Variant for Analytics

Nginx doesn't calculate results for you, but it records everything you need. Add the variant to a custom log format in the `http` block:

```nginx
log_format ab_test '$remote_addr - [$time_local] "$request" $status '
                   '$body_bytes_sent "$http_referer" "$http_user_agent" '
                   'variant=$ab_variant rt=$request_time';

access_log /var/log/nginx/ab_test.log ab_test;
```

Now count how many unique visitors in each variant reached the goal page, for example `/signup/thanks`:

```bash
# Unique visitors (IP) who loaded the landing page, per variant
grep '"GET / ' /var/log/nginx/ab_test.log \
  | grep -o '^[^ ]*\|variant=[ab]' | paste - - | sort -u | cut -f2 | sort | uniq -c

# Unique visitors who reached the goal page, per variant
grep '"GET /signup/thanks' /var/log/nginx/ab_test.log \
  | grep -o '^[^ ]*\|variant=[ab]' | paste - - | sort -u | cut -f2 | sort | uniq -c
```

Divide goal hits by landing page hits for each variant to get a conversion rate. For dashboards, tools like [GoAccess](https://goaccess.io/) can read a custom log format, or you can ship the logs to your existing log pipeline and group by the `variant` field.

### Send the Variant to Your Analytics Tool

If you already use a web analytics tool, expose the variant to the page so your tracking script can record it:

```nginx
add_header X-AB-Variant $ab_variant always;
```

Or have your app (Option C) render it into the page, for example as a `data-variant` attribute that your analytics snippet reads and sends as a custom dimension.

## Step 5: Target Specific Users

Not every test should include every visitor. Combine `split_clients` with `map` and `geo` to target segments.

### Test Only on Mobile

```nginx
map $http_user_agent $is_mobile {
    default                       0;
    "~*(android|iphone|ipad|mobile)" 1;
}

map "$is_mobile:$split_variant" $ab_variant {
    "~^0:"      "a";              # desktop: always control
    "1:a"       "a";
    "1:b"       "b";
}
```

User agent matching is approximate. It's good enough for routing an experiment, not for strict device detection.

### Test by Language

```nginx
map $http_accept_language $lang {
    default   en;
    "~*^fr"   fr;
    "~*^de"   de;
}
```

Then use `$lang` in a `root` path or pass it to your app, or include it in the `map` that decides who's in the experiment.

### Test by Network or Country

The built-in [geo module](https://nginx.org/en/docs/http/ngx_http_geo_module.html) maps IP ranges to values. That's handy for internal testers:

```nginx
geo $internal {
    default        0;
    10.0.0.0/8     1;
    203.0.113.0/24 1;
}
```

For country targeting, use a GeoIP2 database (such as MaxMind's GeoLite2) with the third-party `ngx_http_geoip2_module`. The older built-in `ngx_http_geoip_module` reads the legacy GeoIP format, which MaxMind no longer updates.

## Avoid Common A/B Testing Mistakes

### Don't Break Caching

If you use [proxy_cache](/blog/how-to-set-up-nginx-proxy-cache) or a CDN, variant A's cached HTML could be served to variant B. Add the variant to the cache key:

```nginx
proxy_cache_key "$scheme$request_method$host$request_uri$ab_variant";
```

For a CDN, either skip caching the tested pages or configure it to vary on the `ab` cookie.

### Don't Hurt SEO

Google's guidance on [website testing](https://developers.google.com/search/docs/crawling-indexing/website-testing) says to treat search crawlers like regular visitors (showing them different content is cloaking), to use `rel="canonical"` on variant URLs that point to the original, to use 302 rather than 301 for redirect-based tests, and to end the test when you have results.

### Run It Long Enough

Small samples produce misleading winners. Decide the goal metric and the sample size before you start, and avoid stopping the test the moment one variant pulls ahead.

## Nginx A/B Testing Directive Reference

| Directive or variable | Context | Purpose |
| --- | --- | --- |
| `split_clients` | `http` | Assign a value by hashed percentage |
| `map` | `http` | Derive one variable from another (sticky cookie, targeting) |
| `geo` | `http` | Map client IP ranges to values |
| `$cookie_NAME` | Variable | Read a request cookie |
| `add_header Set-Cookie` | `http`, `server`, `location` | Store the assigned variant |
| `log_format` | `http` | Add `$ab_variant` to access logs |
| `proxy_cache_key` | `http`, `server`, `location` | Keep cached variants separate |

## Troubleshooting Nginx A/B Tests

| Symptom | Cause | Fix |
| --- | --- | --- |
| Users switch between versions | No cookie, or the hash key includes something that changes | Use the sticky `map $cookie_ab` pattern |
| Split is far from 50/50 | Low traffic, or many users share one IP with an IP-only key | Add `$http_user_agent` to the key, wait for more traffic |
| Everyone gets the same variant | `split_clients` key is empty or constant (for example `$remote_addr` behind a load balancer) | Restore the real client IP with the realip module |
| Variant B shows variant A's page | Cache key doesn't include the variant | Add `$ab_variant` to `proxy_cache_key` |
| `no live upstreams` or 502 on one variant | That variant's backend is down | Check the upstream; see [fixing 502 Bad Gateway](/blog/how-to-fix-502-bad-gateway-nginx) |
| Cookie never set | `add_header` overridden in a `location` that has its own `add_header` | Repeat the header in that location, and use `always` |
| `unknown "ab_variant" variable` | `map` or `split_clients` not in the `http` block | Move them to `http` context |

## Monitor Both Variants

An A/B test doubles the number of things that can break. If variant B's backend crashes, half your visitors get errors while variant A looks perfectly healthy, and your conversion numbers for B drop for the wrong reason.

Monitor each variant on its own. With the cookie pattern, you can run a dedicated check for each version, or monitor the upstream ports directly. [GreenCrew](https://greencrew.space) checks your URLs from outside your network and alerts you on Discord, Slack, or Telegram the moment one stops responding, so a broken variant doesn't silently ruin an experiment. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram) to set it up.

## Frequently Asked Questions

### Can Nginx do A/B testing without any plugins?

Yes. `split_clients`, `map`, `geo`, and custom `log_format` are all part of standard Nginx builds. You only need a third-party module for country-level targeting with GeoIP2.

### How do I make sure users always see the same variant?

Hash a stable value in `split_clients`, then save the result in a cookie and read it back with `map $cookie_ab`. The cookie keeps the assignment even if the visitor's IP address changes.

### Is split_clients good for canary releases?

Yes. Send a small share of traffic to the new version, for example `5% "b"; * "a";`, with `proxy_pass http://app_$ab_variant`. Watch error rates for the new upstream, then increase the percentage. For weighted routing without user stickiness, the `weight` parameter in an `upstream` block is another option, covered in [Nginx load balancing](/blog/how-to-set-up-nginx-load-balancing-with-failover).

### How do I end an A/B test in Nginx?

Replace `$ab_variant` with the winning value (or point `root`/`proxy_pass` straight at the winner), remove the `Set-Cookie` header, and reload Nginx. Old cookies are harmless once nothing reads them.

## Conclusion

Nginx's `split_clients` gives you server-side A/B testing with no extra software: hash a stable key, keep the assignment in a cookie, route by variant, and log `$ab_variant` so you can measure each version. Keep cache keys variant-aware, follow Google's testing guidance, and monitor each variant with [GreenCrew](https://greencrew.space) so an outage doesn't get mistaken for a losing design.
