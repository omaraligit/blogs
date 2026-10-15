# How to Add Security Headers in Nginx (HSTS, CSP, X-Frame-Options, and More)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Security, HTTP Headers, TLS, DevOps

Description: To add security headers in Nginx, use add_header with the always flag, for example add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always; put them in a shared snippet so add_header inheritance doesn't drop them, then reload and verify with curl -I. This guide covers HSTS, Content-Security-Policy, X-Content-Type-Options, frame protection, Referrer-Policy, and Permissions-Policy.

Date: 2026-10-08

**Short answer:** create a snippet file such as `/etc/nginx/snippets/security-headers.conf` that contains `add_header` lines for `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, and `Permissions-Policy`, each ending in `always`. Then `include` that snippet in every HTTPS `server` block, and again in any `location` that has its own `add_header`. Run `sudo nginx -t && sudo systemctl reload nginx` and check the result with `curl -I https://example.com`. Roll out HSTS with a short `max-age` first, and start CSP in report-only mode so you don't break your own site.

## What Are HTTP Security Headers?

Security headers are response headers that tell the browser to enable protections it supports but doesn't turn on by default. They don't fix vulnerabilities in your code. They make whole classes of attacks harder: protocol downgrades, clickjacking, MIME sniffing, and the impact of cross-site scripting (XSS).

Here are the headers worth setting on almost every site:

| Header | Protects against | Recommended starting value |
| --- | --- | --- |
| `Strict-Transport-Security` | HTTPS downgrade and cookie theft over HTTP | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | XSS, injected scripts, clickjacking (`frame-ancestors`) | Site-specific; start in report-only |
| `X-Content-Type-Options` | MIME sniffing turning uploads into scripts | `nosniff` |
| `X-Frame-Options` | Clickjacking in older browsers | `SAMEORIGIN` or `DENY` |
| `Referrer-Policy` | Leaking full URLs to other sites | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Unwanted access to camera, mic, location | `camera=(), microphone=(), geolocation=()` |

And two that you should *not* add:

| Header | Why to skip it |
| --- | --- |
| `X-XSS-Protection` | The browser XSS filters it controlled have been removed, and the filter itself could introduce issues. OWASP recommends not setting it, or setting it to `0`. Use CSP instead. |
| `Expect-CT` | Deprecated. Certificate Transparency is now enforced by browsers without it. |

## Step 1: Understand add_header Inheritance (The Most Common Mistake)

Before writing any headers, you need to know how Nginx inherits `add_header`. The [official docs](https://nginx.org/en/docs/http/ngx_http_headers_module.html#add_header) put it this way: directives are inherited from the previous level *only if there are no `add_header` directives defined on the current level*.

So this config silently drops your security headers on `/assets/`:

```nginx
server {
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    location /assets/ {
        add_header Cache-Control "public, max-age=31536000, immutable";
        # Only Cache-Control is sent here. The two headers above are gone.
    }
}
```

The fix is to keep all security headers in one snippet and include it at every level that defines its own `add_header`. The `always` parameter matters too. Without it, Nginx adds the header only to successful and redirect responses (200, 201, 204, 206, 301, 302, 303, 304, 307, 308). With `always`, it's also added to error pages like 404 and 500.

## Step 2: Create a Security Headers Snippet

Create `/etc/nginx/snippets/security-headers.conf` (on RHEL-based systems, `/etc/nginx/conf.d/` is included automatically, so use a different directory such as `/etc/nginx/snippets/` to keep it from loading globally):

```nginx
# /etc/nginx/snippets/security-headers.conf

# Force HTTPS for one year, including subdomains
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Block MIME sniffing
add_header X-Content-Type-Options "nosniff" always;

# Clickjacking protection for older browsers (CSP frame-ancestors covers modern ones)
add_header X-Frame-Options "SAMEORIGIN" always;

# Send only the origin to other sites, nothing over HTTP
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Turn off powerful browser features you don't use
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;

# Content Security Policy: start in report-only (see Step 4)
add_header Content-Security-Policy-Report-Only "default-src 'self'; img-src 'self' data:; object-src 'none'; base-uri 'self'; frame-ancestors 'self'" always;
```

Then include it in your HTTPS server block and in any location with its own headers:

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    include snippets/security-headers.conf;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }

    location /assets/ {
        include snippets/security-headers.conf;   # re-include: this level has add_header
        add_header Cache-Control "public, max-age=31536000, immutable";
        root /var/www/example;
    }
}
```

Apply it:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

If you don't have HTTPS set up yet, do that first. Our guide on [installing a free Let's Encrypt certificate on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot) takes about five minutes. HSTS on a site without working HTTPS will lock visitors out.

## Step 3: Roll Out HSTS Safely

`Strict-Transport-Security` tells browsers to use HTTPS only for your domain for `max-age` seconds. Browsers cache that decision, so a mistake here is hard to undo. Roll it out in stages:

| Stage | Header value | Duration before next stage |
| --- | --- | --- |
| 1. Test | `max-age=300` | A day; confirm every page and subdomain works over HTTPS |
| 2. Short | `max-age=86400; includeSubDomains` | A week |
| 3. Long | `max-age=31536000; includeSubDomains` | Ongoing |
| 4. Preload (optional) | `max-age=31536000; includeSubDomains; preload` | Then submit at hstspreload.org |

Some things to know before you go further:

- **Only send HSTS over HTTPS.** Browsers ignore it on plain HTTP responses, so put it in the `listen 443` server block, not the port 80 redirect block.
- **`includeSubDomains` covers every subdomain.** If `intranet.example.com` or a legacy service only works over HTTP, it will break.
- **Preload is close to permanent.** The [HSTS preload list](https://hstspreload.org/) is built into browsers, and removal takes a long time to reach users. Only submit when you're sure every subdomain will be HTTPS forever.

## Step 4: Build a Content-Security-Policy Without Breaking Your Site

CSP is the most powerful header and the easiest to get wrong. It lists the sources the browser may load scripts, styles, images, fonts, and frames from. Anything not on the list is blocked.

Start with `Content-Security-Policy-Report-Only` (as in the snippet above). The browser reports violations in DevTools but blocks nothing. Browse every part of your site, read the console, and add the sources you really use:

```nginx
add_header Content-Security-Policy-Report-Only "default-src 'self'; script-src 'self' https://cdn.example.net; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self' https://api.example.com; object-src 'none'; base-uri 'self'; frame-ancestors 'self'; form-action 'self'" always;
```

The CSP directives you'll use most:

| Directive | Controls | Typical value |
| --- | --- | --- |
| `default-src` | Fallback for every type not listed | `'self'` |
| `script-src` | JavaScript sources | `'self'` plus your CDN; avoid `'unsafe-inline'` |
| `style-src` | CSS sources | `'self'`; many frameworks need `'unsafe-inline'` at first |
| `img-src` | Images | `'self' data:` |
| `connect-src` | `fetch`, XHR, WebSocket endpoints | `'self'` plus your API |
| `frame-ancestors` | Who can embed your site in a frame | `'self'` or `'none'` |
| `object-src` | Plugins (`<object>`, `<embed>`) | `'none'` |
| `base-uri` | `<base>` tag targets | `'self'` |
| `form-action` | Where forms may submit | `'self'` |

When the console has been clean for a while, rename the header to `Content-Security-Policy` to enforce it. Leave out `'unsafe-inline'` and `'unsafe-eval'` for `script-src` wherever you can. Allowing them removes most of CSP's protection against XSS. The [MDN CSP reference](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Security-Policy) lists every directive.

## Step 5: Hide Version Information

Nginx sends its version in the `Server` header and on default error pages. Turning that off doesn't make you secure, but it gives scanners less to work with:

```nginx
# In the http block of /etc/nginx/nginx.conf
server_tokens off;
```

With that set, the header shows just `Server: nginx`. Removing it entirely requires the third-party `headers-more` module. Also strip headers your app leaks through the proxy:

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_hide_header X-Powered-By;
}
```

## Step 6: Verify Your Security Headers

Check the headers from the command line:

```bash
curl -sI https://example.com | grep -iE "strict-transport|content-security|x-content-type|x-frame|referrer-policy|permissions-policy|server"
```

Test an error page and a static asset too, because that's where inheritance bugs show up:

```bash
curl -sI https://example.com/this-does-not-exist | grep -i strict-transport
curl -sI https://example.com/assets/app.css | grep -i x-content-type
```

For a graded report, run your site through the [MDN HTTP Observatory](https://developer.mozilla.org/en-US/observatory), which scores each header and explains what's missing.

## Troubleshooting Nginx Security Headers

| Symptom | Cause | Fix |
| --- | --- | --- |
| Headers missing on some paths | A `location` has its own `add_header`, which cancels inheritance | Include the snippet in that `location` |
| Headers missing on 404/500 pages | No `always` parameter | Add `always` to every security header |
| Header appears twice | Both the app and Nginx set it | `proxy_hide_header <Name>;` or remove it from one side |
| Site broken after enabling CSP | Policy blocks a script, style, or API you use | Go back to report-only, read the console, add the sources |
| Embedded widget stopped loading | `X-Frame-Options` or `frame-ancestors` on the embedded site | Allow the parent origin in `frame-ancestors` on that site |
| Subdomain unreachable after HSTS | `includeSubDomains` with a subdomain that has no HTTPS | Add a certificate to that subdomain; wait out or lower `max-age` |
| HSTS header ignored | Sent over plain HTTP only | Add it to the `listen 443` server block |

## Watch for Breakage After Header Changes

Security header changes fail quietly. A strict CSP can block your JavaScript bundle, and the server still returns `200 OK` for a blank page. A bad HSTS rollout can lock users out of a subdomain while your main site looks fine from your laptop.

After each change, check more than the status code:

| Check | What to monitor | Catches |
| --- | --- | --- |
| HTTP status | `https://example.com/` | Nginx config errors, TLS failures, 5xx from a broken reload |
| HTTP status | Every subdomain covered by `includeSubDomains` | Subdomains without valid HTTPS |
| Keyword | Text that only appears once the page renders | Pages that return 200 but are broken |

[GreenCrew](https://greencrew.space) runs these HTTP checks from outside your network and alerts you on Discord, Slack, or Telegram when one fails. Your public status page shows users what's happening if a change goes wrong. If you're also tuning other parts of the stack, like [compression](/blog/how-to-enable-gzip-and-brotli-compression-in-nginx) or [rate limiting](/blog/how-to-set-up-rate-limiting-in-nginx), the same monitors will catch regressions there.

## Frequently Asked Questions

### Where should I put add_header in Nginx: http, server, or location?

Any of them work, but because of the inheritance rule, it's easiest to `include` a snippet in each HTTPS `server` block and in every `location` that sets its own `add_header`. Putting headers only in the `http` block works until the first `server` or `location` adds a header of its own, and then they all disappear for that block.

### Do I still need X-Frame-Options if I use CSP frame-ancestors?

Modern browsers use `frame-ancestors` and ignore `X-Frame-Options` when both are present. Keeping `X-Frame-Options` costs nothing and covers older browsers, so most sites send both with matching values.

### Do security headers help SEO?

Not directly. Search engines don't rank pages higher for having CSP or HSTS. HTTPS is a known ranking signal, though, and HSTS makes sure visitors always reach the HTTPS version. Headers also protect your users and your site's reputation, which matters more.

### Should I set security headers in Nginx or in my application?

Set them in one place. Nginx is a good default because every response passes through it, including static files and error pages your app never sees. Set them in the app only for page-specific values, like a CSP with per-request nonces, and use `proxy_hide_header` so you don't send duplicates.

## Conclusion

Adding security headers in Nginx takes one snippet file, a few `include` lines, and an understanding of how `add_header` inheritance works. Use `always` so error pages get the headers too, roll out HSTS in stages, run CSP in report-only mode until the console is clean, and verify with `curl -I` and the HTTP Observatory. Then add a [GreenCrew](https://greencrew.space) monitor to your main domain and subdomains, so a header change that breaks something reaches you on Discord, Slack, or Telegram before your users notice.
