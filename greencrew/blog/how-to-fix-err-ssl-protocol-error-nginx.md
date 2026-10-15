# How to Fix ERR_SSL_PROTOCOL_ERROR in Nginx ("This Site Can't Provide a Secure Connection")

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, SSL, TLS, Troubleshooting, DevOps

Description: ERR_SSL_PROTOCOL_ERROR usually means the server answered an HTTPS request with plain HTTP, most often because the Nginx server block says listen 443 without the ssl parameter. Change it to listen 443 ssl, point ssl_certificate and ssl_certificate_key at a valid certificate, and run nginx -t && systemctl reload nginx. This guide covers every server-side and client-side cause, how to diagnose each with curl and openssl, and how to prevent it.

Date: 2026-10-15

**Short answer:** Chrome shows **ERR_SSL_PROTOCOL_ERROR** ("This site can't provide a secure connection ... sent an invalid response") when the TLS handshake fails before any page loads. On an Nginx server, the most common cause is **plain HTTP being served on port 443**, usually because the server block has `listen 443;` instead of `listen 443 ssl;`. Run `curl -v https://example.com`: if you see `wrong version number`, that's the problem. Fix the `listen` line, make sure `ssl_certificate` and `ssl_certificate_key` are set for that server block, then `sudo nginx -t && sudo systemctl reload nginx`. If the server is fine, the cause is on the client side, such as an HSTS-forced `.dev` domain, antivirus HTTPS scanning, or a proxy.

## What Is ERR_SSL_PROTOCOL_ERROR?

Before your browser loads anything over HTTPS, it runs a **TLS handshake** with the server to agree on a protocol version and ciphers and to verify the certificate. ERR_SSL_PROTOCOL_ERROR means the handshake broke because the server's reply wasn't valid TLS, or the connection was cut mid-handshake.

It's different from certificate errors, where the handshake works but the certificate isn't trusted. Each browser words it differently:

| Browser / tool | What you see for the same problem |
| --- | --- |
| Chrome, Edge, Brave | `ERR_SSL_PROTOCOL_ERROR`, "This site can't provide a secure connection" |
| Firefox | `SSL_ERROR_RX_RECORD_TOO_LONG` (plain HTTP on 443) or `PR_END_OF_FILE_ERROR` |
| Safari | "Safari can't open the page because Safari can't establish a secure connection to the server" |
| curl (OpenSSL 3) | `SSL routines::wrong version number` or `unexpected eof while reading` |
| `openssl s_client` | `wrong version number`, or `no peer certificate available` |

### Related Errors That Aren't the Same Thing

| Error | Means | Where to look instead |
| --- | --- | --- |
| `ERR_SSL_VERSION_OR_CIPHER_MISMATCH` | Handshake ran but no protocol/cipher in common (often TLS 1.0/1.1 only) | `ssl_protocols`, `ssl_ciphers` |
| `NET::ERR_CERT_AUTHORITY_INVALID` | Certificate not trusted (self-signed, missing chain) | Use `fullchain.pem`, or a real CA |
| `NET::ERR_CERT_COMMON_NAME_INVALID` | Certificate doesn't cover this hostname | Reissue with the right `-d` names |
| `NET::ERR_CERT_DATE_INVALID` | Certificate expired, or the client clock is wrong | Renew, or fix the clock |
| `ERR_CONNECTION_REFUSED` | Nothing listening on 443 at all | Firewall, `listen 443` missing |
| `ERR_TOO_MANY_REDIRECTS` | HTTP ↔ HTTPS redirect loop | Redirect rules, CDN SSL mode |

## Step 1: Diagnose with curl and openssl

Test from your own machine first, then from the server itself to skip DNS, CDNs, and firewalls.

```bash
curl -vI https://example.com
```

Look at the lines just after `Connected to`:

| curl output | What it means |
| --- | --- |
| `SSL routines::wrong version number` | Server replied with plain HTTP. **Missing `ssl` on `listen 443`** or traffic forwarded to an HTTP port |
| `unexpected eof while reading` / `Connection reset by peer` | Server closed the connection mid-handshake (often no certificate for that server block) |
| `SSL certificate problem: ...` | Handshake worked, so it's a certificate trust issue, not this error |
| `HTTP/2 200` | Server is fine from here. Look at the client, CDN, or network |

Confirm with OpenSSL, using `-servername` so SNI is sent the way a browser sends it:

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

A healthy server prints a certificate chain and `Verify return code: 0 (ok)`. If you see `wrong version number`, prove the server is speaking HTTP on 443:

```bash
curl -i http://example.com:443
# HTTP/1.1 200 OK   <- plain HTTP on the HTTPS port
# or: HTTP/1.1 400 Bad Request ... "The plain HTTP request was sent to HTTPS port"
```

The second response means Nginx *is* expecting TLS on that port, so the server block is fine and something in front of it isn't. Then check what's listening and what Nginx actually loaded:

```bash
sudo ss -tlnp | grep ':443'
sudo nginx -T 2>/dev/null | grep -nE 'listen .*443|ssl_certificate|server_name'
sudo tail -n 50 /var/log/nginx/error.log
```

`nginx -T` prints the full merged config, including every file in `sites-enabled/` and `conf.d/`. That's the fastest way to spot a second server block on 443 that you forgot about.

## Step 2: Fix listen 443 Without ssl (Most Common Cause)

This config serves plain HTTP on port 443:

```nginx
server {
    listen 443;              # wrong: no "ssl"
    server_name example.com;
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

Having `ssl_certificate` lines isn't enough. TLS is only enabled by the `ssl` parameter on `listen`. Fix it:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;

    root /var/www/example.com;
}

server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -vI https://example.com
```

**Old guides use `ssl on;`.** That directive was deprecated in Nginx 1.15.0 and removed in 1.25.1, so on current versions `nginx -t` fails with an unknown directive error. Use `listen 443 ssl;` instead. Likewise, `listen 443 ssl http2;` still works but is deprecated since 1.25.1 in favor of the separate `http2 on;` directive. See the [ngx_http_ssl_module docs](https://nginx.org/en/docs/http/ngx_http_ssl_module.html) and [configuring HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html).

### Check Every Server Block on Port 443

Nginx decides which server block answers a connection by IP:port first, then by SNI hostname. A port either speaks TLS or it doesn't, so don't mix `listen 443 ssl` in one block with plain `listen 443` in another, or with a stray `listen 443` on a specific IP. A leftover block from an old site or a copied snippet is enough to break HTTPS for some hostnames. Make every `listen 443` line in the output of `nginx -T` include `ssl`.

## Step 3: Make Sure Every SSL Server Block Has a Certificate

When a block listens with `ssl` but has no certificate (and none is inherited from the `http` level), Nginx accepts the TCP connection and then fails the handshake. The error log says:

```text
no "ssl_certificate" is defined for the "listen ... ssl" directive in ...
```

or, at request time:

```text
no "ssl_certificate" is defined in server listening on SSL port while SSL handshaking
```

The usual culprit is a **default server** on 443 (for example a catch-all `server_name _;` block) with no certificate. Clients whose SNI doesn't match another block, or that send no SNI, land there and the handshake fails. Give the catch-all a certificate, even a [self-signed one](https://greencrew.space/blog/how-to-create-a-self-signed-ssl-certificate-for-nginx), or reject those handshakes outright (supported since Nginx 1.19.4):

```nginx
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    ssl_reject_handshake on;
}
```

Also confirm the files exist and match:

```bash
sudo ls -l /etc/letsencrypt/live/example.com/
sudo openssl x509 -noout -modulus -in /etc/letsencrypt/live/example.com/fullchain.pem | openssl md5
sudo openssl rsa  -noout -modulus -in /etc/letsencrypt/live/example.com/privkey.pem  | openssl md5
```

The two hashes must match. (This modulus comparison works for RSA keys. Certbot issues ECDSA keys by default in recent versions, so for those compare `openssl x509 -noout -pubkey` with `openssl pkey -pubout` instead.) If you don't have a certificate yet, follow [how to install Let's Encrypt SSL on Nginx with Certbot](https://greencrew.space/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

## Step 4: Check What's in Front of Nginx

If Nginx is correct when tested on the server (`curl -vI --resolve example.com:443:127.0.0.1 https://example.com`) but fails from outside, something in between is breaking TLS.

| Setup | Cause | Fix |
| --- | --- | --- |
| Docker | Port mapped as `-p 443:80`, so container HTTP is exposed on host 443 | Map `443:443` and configure TLS in the container, or terminate TLS on the host |
| Cloud load balancer | Listener on 443 forwards TLS straight to an HTTP target (or the reverse) | Match the listener protocol to the target protocol |
| Another service on 443 | Apache, a VPN, or a panel bound to 443 first | `sudo ss -tlnp | grep :443`, stop or move it |
| Router port forwarding | WAN 443 forwarded to LAN port 80 | Forward 443 → 443 |
| CDN/proxy in TCP passthrough | Origin isn't speaking TLS on the port the CDN connects to | Enable TLS on the origin or change the CDN's origin port |
| Reverse proxy to an app | `proxy_pass https://...` to an app that only speaks HTTP | Use `proxy_pass http://127.0.0.1:PORT` (see [reverse proxy guide](https://greencrew.space/blog/how-to-set-up-nginx-reverse-proxy)) |

The last row shows up in the Nginx error log as `SSL_do_handshake() failed ... wrong version number while SSL handshaking to upstream`, and the browser usually gets a 502 rather than ERR_SSL_PROTOCOL_ERROR. It's the same mistake in the other direction.

## Step 5: Rule Out Protocol and Cipher Problems

Modern browsers require TLS 1.2 or later. If your config forces old protocols only, the error changes to ERR_SSL_VERSION_OR_CIPHER_MISMATCH, but some clients report a generic protocol error. Use:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

Test each version:

```bash
openssl s_client -connect example.com:443 -servername example.com -tls1_2 </dev/null | grep -E 'Protocol|Cipher'
openssl s_client -connect example.com:443 -servername example.com -tls1_3 </dev/null | grep -E 'Protocol|Cipher'
```

Mozilla's [SSL Configuration Generator](https://ssl-config.mozilla.org/) produces a tested `ssl_protocols` and `ssl_ciphers` set for your Nginx and OpenSSL versions.

## Step 6: Client-Side Causes

If curl and openssl succeed from several networks, the server is fine and the problem is on the visitor's machine:

| Cause | How to tell | Fix |
| --- | --- | --- |
| HSTS-preloaded TLD (`.dev`, `.app`) on a local HTTP server | Chrome forces HTTPS for these domains, so your local HTTP-only server fails | Use `localhost` or `.test`, or serve local HTTPS |
| Domain previously sent HSTS | Site worked over HTTPS once; now HTTPS is broken and the browser won't fall back | Fix HTTPS; for local testing, delete the domain at `chrome://net-internals/#hsts` |
| Antivirus "HTTPS scanning" | Fails on one machine only; works in a VM or on mobile | Disable web/HTTPS scanning or update the antivirus |
| Corporate proxy or captive portal | Fails on one network only | Test on mobile data; ask IT about TLS inspection |
| Typed `https://` for a dev server | `https://localhost:3000` while the dev server only speaks HTTP | Use `http://localhost:3000` or enable HTTPS in the dev server |
| Wrong system clock | Usually certificate date errors, sometimes generic handshake failures | Enable automatic time sync |
| Stale cached state | Error persists after a fix | Try an incognito window, clear the SSL state, restart the browser |

Also be careful with [HSTS headers](https://greencrew.space/blog/how-to-add-security-headers-in-nginx). Once browsers have seen `Strict-Transport-Security`, they refuse plain HTTP for that domain for the whole `max-age`. A broken TLS config then means the site is completely unreachable for returning visitors, with no HTTP fallback.

## ERR_SSL_PROTOCOL_ERROR Troubleshooting Table

| Symptom | Cause | Fix |
| --- | --- | --- |
| `curl`: `wrong version number` | Plain HTTP on port 443 | `listen 443 ssl;` in every 443 block |
| `curl -i http://host:443` returns a normal page | Same as above, confirmed | Add `ssl` to `listen`, reload |
| `nginx -t`: `unknown directive "ssl"` | `ssl on;` removed in Nginx 1.25.1 | Delete it, use `listen 443 ssl;` |
| Error log: `no "ssl_certificate" is defined` | SSL server block without a certificate | Add a certificate or `ssl_reject_handshake on;` |
| Works for `example.com`, fails for another hostname | That name falls through to a default server without a certificate | Add the name to the right block, give the default a cert |
| Works on the server, fails from outside | Docker/LB/router forwarding 443 to an HTTP port | Fix the port mapping or listener protocol |
| Fails only on one machine or network | Antivirus, proxy, or HSTS on a local domain | See the client-side table |
| Error appeared after a Certbot run | Certbot edited the wrong block, or a duplicate block exists | `nginx -T | grep -n 443`, remove the duplicate |
| Error appeared after renewal | Nginx not reloaded or a file path changed | `sudo systemctl reload nginx`, check the deploy hook |

## Get Alerted When HTTPS Breaks

TLS problems usually come from a change: a new server block, a Certbot run that edited the wrong file, a Docker port mapping, a load balancer tweak. They're also easy to miss when you're testing from a machine that has cached state, or over HTTP.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks from outside your network. Point a check at your `https://` URL, and when a handshake failure makes the site unreachable, you get an alert on Discord, Slack, or Telegram, while your public status page keeps users informed. Add the check before you change TLS config, not after. See [how to get website downtime alerts on Discord, Slack, and Telegram](https://greencrew.space/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### What causes ERR_SSL_PROTOCOL_ERROR?

The browser received something that isn't a valid TLS handshake, or the connection was closed mid-handshake. On Nginx servers it's most often `listen 443` without `ssl`, a server block on 443 with no certificate, or a port forward that sends HTTPS traffic to an HTTP port.

### Is ERR_SSL_PROTOCOL_ERROR a server or a client problem?

Usually the server. Test with `curl -vI https://yourdomain` and `openssl s_client` from a couple of networks. If they fail, it's the server or something in front of it. If they succeed and only one browser or machine fails, it's client-side (antivirus, proxy, HSTS, cached state).

### Why does it only happen on localhost or a .dev domain?

Chrome has `.dev` and `.app` on the HSTS preload list, so it always uses HTTPS for them. If your local server only speaks HTTP, the handshake fails. Use `localhost` or a `.test` domain for local work, or run local HTTPS with a [self-signed certificate](https://greencrew.space/blog/how-to-create-a-self-signed-ssl-certificate-for-nginx).

### Does clearing the browser cache fix ERR_SSL_PROTOCOL_ERROR?

Only when the server was already fixed and the browser is holding stale state. Clearing the cache doesn't fix a misconfigured server, so check with curl first and only then clear the cache or try an incognito window.

## Conclusion

ERR_SSL_PROTOCOL_ERROR means the TLS handshake failed before any HTTP happened. On Nginx, start with `curl -vI https://yourdomain`: `wrong version number` points at `listen 443` without `ssl` or a port forward to HTTP, and a reset connection points at a server block with no certificate. Fix the `listen` lines, give every SSL block (including the default) a certificate or `ssl_reject_handshake`, keep `ssl_protocols TLSv1.2 TLSv1.3`, and check what sits in front of Nginx. Then set up an external HTTPS check with [GreenCrew](https://greencrew.space) so the next broken handshake reaches you as an alert.
