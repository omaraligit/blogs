# How to Enable Gzip and Brotli Compression in Nginx (and Verify It Works)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Performance, Gzip, Brotli, Web Performance

Description: Enable gzip in Nginx with gzip on, gzip_comp_level 5, gzip_proxied any, gzip_vary on, and a gzip_types list for CSS, JavaScript, JSON, and SVG. Add Brotli with the ngx_brotli module, then verify with curl -H "Accept-Encoding: br" -I. A copy-paste config, a compression level table, and fixes for when compression doesn't work.

Date: 2026-10-07

**Short answer:** add `gzip on;`, `gzip_comp_level 5;`, `gzip_min_length 256;`, `gzip_proxied any;`, `gzip_vary on;`, and a `gzip_types` list (CSS, JavaScript, JSON, XML, SVG) to the `http` block, then reload Nginx. Install the `ngx_brotli` module for Brotli, which compresses text assets further than gzip, and add `brotli on;` with the same types. Check it with `curl -H "Accept-Encoding: br, gzip" -I https://example.com/app.js` and look for a `content-encoding` header. Text responses typically shrink by well over half, which speeds up page loads and cuts bandwidth costs.

## What Do Gzip and Brotli Do?

Gzip and Brotli are lossless compression formats. The browser lists the ones it supports in the `Accept-Encoding` request header. Nginx compresses the response with the best match and labels it with `Content-Encoding`, and the browser decompresses it transparently. Every modern browser supports both, though browsers only advertise Brotli over HTTPS.

| | Gzip | Brotli |
| --- | --- | --- |
| Built into Nginx | Yes (`ngx_http_gzip_module`) | No, needs the `ngx_brotli` module |
| Compression ratio on text | Good | Better, typically smaller output than gzip |
| CPU cost at high levels | Moderate | High at levels 10–11, so use precompression |
| Browser support | Universal | All modern browsers, HTTPS only |
| `Content-Encoding` value | `gzip` | `br` |
| Best use | Fallback for every client | Primary encoding for HTTPS sites |

Run both. Nginx serves Brotli to clients that ask for it and gzip to everyone else.

## Step 1: Check What Your Site Sends Today

Get a baseline before changing anything:

```bash
URL=https://example.com/assets/app.js
curl -s -o /dev/null -w "uncompressed: %{size_download} bytes\n" "$URL"
curl -s -o /dev/null -H "Accept-Encoding: gzip" -w "gzip: %{size_download} bytes\n" "$URL"
curl -s -o /dev/null -H "Accept-Encoding: br" -w "brotli: %{size_download} bytes\n" "$URL"
```

If all three numbers are the same, compression is off for that file type.

## Step 2: Enable Gzip Compression

Add this to the `http` block in `/etc/nginx/nginx.conf`, or create `/etc/nginx/conf.d/compression.conf`. Ubuntu's default `nginx.conf` already has `gzip on;` with most other options commented out, so edit it rather than adding a duplicate `gzip on`:

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 256;
gzip_proxied any;
gzip_vary on;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/javascript
    application/json
    application/ld+json
    application/manifest+json
    application/xml
    application/rss+xml
    application/atom+xml
    application/wasm
    image/svg+xml
    font/ttf
    font/otf;
```

Then:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### What Each Gzip Directive Does

| Directive | Default | Recommended | Why |
| --- | --- | --- | --- |
| `gzip` | `off` | `on` | Enables the module |
| `gzip_comp_level` | `1` | `4`–`6` | Above 6 costs much more CPU for very small gains |
| `gzip_min_length` | `20` | `256` | Tiny responses can get bigger when compressed |
| `gzip_proxied` | `off` | `any` | Without it, requests that come through a CDN or proxy (with a `Via` header) aren't compressed |
| `gzip_vary` | `off` | `on` | Sends `Vary: Accept-Encoding` so caches store compressed and uncompressed copies separately |
| `gzip_types` | `text/html` | See list above | `text/html` is always compressed; everything else must be listed |

### Which Files Not to Compress

Don't add already-compressed formats to `gzip_types`. You spend CPU and the file usually gets slightly larger:

| Compress | Don't compress |
| --- | --- |
| HTML, CSS, JavaScript, JSON, XML, SVG, TTF/OTF, WASM | JPEG, PNG, WebP, AVIF, GIF, MP4, WOFF/WOFF2, ZIP, PDF |

WOFF2 already uses Brotli internally, so compressing it again does nothing.

## Step 3: Add Brotli Compression

Nginx's open source build doesn't include Brotli, so you load Google's [ngx_brotli](https://github.com/google/ngx_brotli) module. The easiest route depends on how you installed Nginx:

| Nginx source | How to get Brotli |
| --- | --- |
| Debian 12+ / recent Ubuntu distro package | `sudo apt install libnginx-mod-http-brotli-filter libnginx-mod-http-brotli-static` |
| nginx.org official packages | Build `ngx_brotli` as a dynamic module that matches your exact Nginx version |
| Docker | Use an image that bundles the module, or build it in a multi-stage Dockerfile |

Check whether the module is loaded:

```bash
ls /etc/nginx/modules-enabled/ | grep -i brotli
nginx -V 2>&1 | grep -o brotli
```

Then add the config next to your gzip settings:

```nginx
brotli on;
brotli_comp_level 5;
brotli_min_length 256;
brotli_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/javascript
    application/json
    application/ld+json
    application/manifest+json
    application/xml
    application/rss+xml
    application/atom+xml
    application/wasm
    image/svg+xml
    font/ttf
    font/otf;
```

Brotli at levels 4–6 is a good balance for on-the-fly compression. Save levels 10–11 for precompressed files, because they're too slow to run on every request.

## Step 4: Precompress Static Files with gzip_static and brotli_static

On-the-fly compression runs on every request. For files that don't change between deploys, such as build output, compress once at the highest level and let Nginx serve the `.gz` or `.br` file directly:

```bash
# Run at build/deploy time
find /var/www/example.com -type f \
  \( -name '*.js' -o -name '*.css' -o -name '*.svg' -o -name '*.json' -o -name '*.html' \) \
  -exec gzip -k -9 -f {} \; \
  -exec brotli -k -q 11 -f {} \;
```

```nginx
location /assets/ {
    root /var/www/example.com;
    gzip_static on;     # serves app.js.gz if present
    brotli_static on;   # serves app.js.br if present
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

This gives you the smallest files with almost no CPU cost per request. `gzip_static` needs `ngx_http_gzip_static_module`, which most distro packages include. Check with `nginx -V 2>&1 | grep -o gzip_static`.

## Compression Levels: Size vs CPU

| Level | Gzip | Brotli | When to use |
| --- | --- | --- | --- |
| 1–3 | Fast, decent ratio | Fast, ratio similar to gzip 6 | Very high traffic, CPU-limited servers |
| 4–6 | Balanced (**recommended**) | Balanced (**recommended**) | On-the-fly compression for most sites |
| 7–9 | Slow, marginal gains | Slower, better ratio | Rarely worth it on the fly |
| 10–11 | n/a | Very slow, best ratio | Precompressed static files only |

## Step 5: Verify Compression Works

```bash
curl -sI -H "Accept-Encoding: br, gzip" https://example.com/assets/app.js \
  | grep -iE "content-encoding|vary|content-type"
# content-type: application/javascript
# content-encoding: br
# vary: Accept-Encoding
```

Repeat with `-H "Accept-Encoding: gzip"` to confirm the gzip fallback. Then re-run the Step 1 size comparison to measure the actual savings on your own assets. Results depend heavily on content, so measure rather than guess.

In the browser, open DevTools, go to **Network**, and add the **Content-Encoding** column. Each text asset should show `br` or `gzip`.

## Troubleshooting: Nginx Compression Not Working

| Symptom | Cause | Fix |
| --- | --- | --- |
| No `content-encoding` header on JS or CSS | MIME type missing from `gzip_types` | Check the response `content-type` and add the exact type |
| Works directly, not through a CDN | `gzip_proxied` is `off` | Set `gzip_proxied any;` |
| Proxied app responses never compressed | Backend already sends `Content-Encoding`, or the app compresses itself | Compress in one place only, preferably Nginx |
| `unknown directive "brotli"` | Module not installed or not loaded | Install the module, or add `load_module modules/ngx_http_brotli_filter_module.so;` |
| Brotli never served | Testing over plain HTTP, or the client didn't send `br` | Browsers only advertise `br` over HTTPS; test with an explicit `Accept-Encoding` |
| Duplicate `gzip` directive error | `gzip on` in both `nginx.conf` and your include | Keep the settings in one place |

## A Note on Security: BREACH

Compressing HTTPS responses that include both secrets (CSRF tokens, session data) and attacker-controlled input can, in theory, leak those secrets through the [BREACH attack](https://www.breachattack.com/). The usual mitigations are per-request masking of CSRF tokens (most modern frameworks do this) and not reflecting user input on pages that contain secrets. Compressing static assets carries no BREACH risk.

## Make Sure Faster Doesn't Mean Broken

Compression changes affect every response, and mistakes show up in unexpected places. A misconfigured `gzip_static` can serve stale `.gz` files after a deploy, a double-compressed API response breaks clients, and a bad module install stops Nginx from reloading. Monitor your key pages and API endpoints from outside so you catch problems right after the change ships. [GreenCrew](https://greencrew.space) checks your URLs continuously and alerts you on Discord, Slack, or Telegram when a check fails. Pair it with a hardened [Nginx reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy) and HTTPS from [Let's Encrypt](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot), which you need anyway for browsers to accept Brotli.

## Frequently Asked Questions

### Should I use gzip or Brotli in Nginx?

Use both. Brotli usually produces smaller text files and is supported by every modern browser over HTTPS, while gzip is the universal fallback for older clients, plain HTTP, and many API tools. Nginx picks the right one for each request based on `Accept-Encoding`.

### What is the best gzip_comp_level for Nginx?

Use levels 4 to 6. Level 1 is fastest but compresses less. Beyond 6, CPU use climbs steeply while files barely get smaller. For maximum compression, precompress static assets at build time and serve them with `gzip_static on;`.

### Does Nginx compress HTML by default?

Only if `gzip on;` is set. Once it's on, `text/html` is always compressed, and every other type must be listed in `gzip_types`. That's why many sites compress HTML but accidentally serve CSS and JavaScript uncompressed.

### Does compression help SEO?

Indirectly, yes. Smaller responses download faster, which improves page load metrics such as Largest Contentful Paint. Those count toward Core Web Vitals, which Google uses as a page experience signal. Lighthouse also flags "Enable text compression" as a performance issue when assets aren't compressed.

## Conclusion

Turning on gzip takes six directives. Adding Brotli takes one module and the same type list. Use levels 4 to 6 on the fly, precompress static builds at maximum level, set `gzip_proxied any` and `gzip_vary on`, and check every change with `curl`. Then keep an eye on the result with external monitoring from [GreenCrew](https://greencrew.space), so faster pages never come at the cost of broken ones.
