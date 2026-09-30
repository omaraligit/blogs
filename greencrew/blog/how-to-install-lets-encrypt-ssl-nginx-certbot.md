# How to Install a Free Let's Encrypt SSL Certificate on Nginx with Certbot (and Auto-Renew It)

Author: [greencrew](https://greencrew.space/blog)

Tags: Let's Encrypt, SSL, Nginx, Certbot, HTTPS

Description: Install Certbot with snap, run sudo certbot --nginx -d example.com -d www.example.com, then confirm auto-renewal with sudo certbot renew --dry-run. A step-by-step Ubuntu guide with renewal troubleshooting, rate limits, wildcard certificates, and SSL expiry monitoring.

Date: 2026-09-30

**Short answer:** install Certbot, run `sudo certbot --nginx -d example.com -d www.example.com`, and let it edit your Nginx server block. It also installs a renewal timer. Then run `sudo certbot renew --dry-run` to prove renewal works. Your site is on HTTPS in under five minutes, for free, with no manual renewals. The rest of this guide covers what Certbot changes, how renewal really works, the rate limits that can lock you out, and how to catch a failed renewal before your visitors see a certificate warning.

## Why Let's Encrypt Certificate Expiry Matters More in 2026

Let's Encrypt certificates are free and trusted by every major browser, but they're short-lived on purpose. Two changes make it more important than ever to automate and monitor renewal:

| Change | What happened | Impact on you |
| --- | --- | --- |
| Expiry emails ended | Let's Encrypt stopped sending expiration reminder emails in 2025 | Nobody warns you when renewal silently breaks |
| Shorter lifetimes | Let's Encrypt announced a move from 90-day to 45-day certificates, phased in from 2026 to 2028 | Renewals happen about twice as often, so a broken renewal hurts sooner |
| Industry-wide limits | CA/Browser Forum ballot SC-081 cuts maximum public TLS certificate lifetime in steps down to 47 days by 2029 | Manual certificate management won't keep up |

Sources: [Let's Encrypt blog](https://letsencrypt.org/blog/), [CA/Browser Forum](https://cabforum.org/).

In short, a cron job you set up two years ago and never checked is now a real outage risk.

## Prerequisites

| Requirement | How to check |
| --- | --- |
| A domain with an A (and AAAA) record pointing to your server | `dig +short example.com` |
| Nginx installed with a `server_name` for the domain | `sudo nginx -T \| grep server_name` |
| Port 80 open to the internet (for HTTP-01 validation) | `sudo ufw status` |
| Root or sudo access | `sudo -v` |

If you haven't configured Nginx yet, follow [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy) first. Certbot needs an existing server block whose `server_name` matches the domain you request.

## Step 1: Install Certbot

The Certbot project recommends the snap package because it stays current and ships with a renewal timer. Remove any old apt version first:

```bash
sudo apt remove -y certbot
sudo snap install --classic certbot
sudo ln -sf /snap/bin/certbot /usr/bin/certbot
certbot --version
```

If you can't use snap, `sudo apt install certbot python3-certbot-nginx` works on Ubuntu and Debian. Distro packages often lag behind upstream releases, though.

## Step 2: Get the Certificate and Configure Nginx

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot will:

1. Ask for an email address and for you to accept the terms of service.
2. Create an ACME account and request a certificate for both names.
3. Complete the HTTP-01 challenge by temporarily serving a token at `/.well-known/acme-challenge/`.
4. Save the certificate to `/etc/letsencrypt/live/example.com/`.
5. Edit your server block to listen on 443 and redirect HTTP to HTTPS.

For unattended scripts, pass everything as flags:

```bash
sudo certbot --nginx --non-interactive --agree-tos \
  -m ops@example.com -d example.com -d www.example.com --redirect
```

### What Certbot Adds to Your Nginx Config

```nginx
server {
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = www.example.com) { return 301 https://$host$request_uri; } # managed by Certbot
    if ($host = example.com) { return 301 https://$host$request_uri; } # managed by Certbot
    listen 80;
    server_name example.com www.example.com;
    return 404; # managed by Certbot
}
```

### Certificate Files Explained

| File | Contains | Used by |
| --- | --- | --- |
| `fullchain.pem` | Your certificate plus the intermediate chain | `ssl_certificate` |
| `privkey.pem` | Your private key (keep it secret) | `ssl_certificate_key` |
| `cert.pem` | Your certificate only | Rarely needed; causes chain errors if used in Nginx |
| `chain.pem` | Intermediate certificates only | OCSP stapling configs, some Java apps |

Always point Nginx at `fullchain.pem`. Using `cert.pem` works in desktop Chrome but breaks on many mobile clients and API tools that can't fetch missing intermediates.

## Step 3: Verify HTTPS Works

```bash
curl -I https://example.com
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates
```

You should see a Let's Encrypt issuer and a `notAfter` date about 90 days away (shorter if you've opted into a short-lived profile).

## Step 4: Confirm Auto-Renewal

The snap package installs a systemd timer that runs twice a day and renews any certificate within 30 days of expiry, or earlier for shorter-lived certificates. Check it and test a renewal without touching your real certificates:

```bash
systemctl list-timers | grep -i certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

`certbot certificates` lists every certificate, the domains on it, and how many days it has left.

### Reload Nginx After Renewal

The `--nginx` installer reloads Nginx for you. If you obtained certificates with `certonly` or `--webroot`, add a deploy hook, or Nginx will keep serving the old certificate from memory until its next reload:

```bash
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh > /dev/null <<'EOF'
#!/bin/sh
nginx -t && systemctl reload nginx
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

This hook is behind most "the certificate renewed but the site still shows it expired" incidents.

## Wildcard Certificates with DNS-01

Wildcards like `*.example.com` require the DNS-01 challenge, because Let's Encrypt must confirm you control the DNS zone:

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  -d example.com -d '*.example.com'
```

Manual DNS certificates **do not auto-renew**. For automation, install a DNS plugin for your provider, such as `certbot-dns-cloudflare` or `certbot-dns-route53`, and pass its credentials file.

| Challenge | Port needed | Wildcards | Auto-renew | Best for |
| --- | --- | --- | --- | --- |
| HTTP-01 (`--nginx`) | 80 | No | Yes | Most single servers |
| DNS-01 (plugin) | None | Yes | Yes | Wildcards, servers behind firewalls |
| DNS-01 (`--manual`) | None | Yes | No | One-off testing only |
| TLS-ALPN-01 | 443 | No | Yes | Port 80 blocked (needs a supporting client) |

## Let's Encrypt Rate Limits to Know

Testing against production is the fastest way to get locked out. Use `--dry-run` or `--staging` while you experiment. The main limits, per the [Let's Encrypt rate limit docs](https://letsencrypt.org/docs/rate-limits/):

| Limit | Value | Typical trigger |
| --- | --- | --- |
| Certificates per registered domain | 50 per week | Many subdomains on separate certificates |
| Duplicate certificates (same exact names) | 5 per week | Reinstalling the server over and over |
| Failed validations | 5 per hour per account per hostname | Broken DNS or blocked port 80 |
| New orders | 300 per 3 hours per account | Bulk automation |

Renewals of an existing certificate get special treatment, so routine auto-renewal doesn't hit these limits.

## Troubleshooting Certbot Errors

| Error message | Cause | Fix |
| --- | --- | --- |
| `Could not automatically find a matching server block` | No `server_name` matches the requested domain | Add the domain to `server_name`, then `nginx -t && systemctl reload nginx` |
| `Timeout during connect (likely firewall problem)` | Port 80 blocked | `sudo ufw allow 80`, check your cloud security group |
| `Invalid response from http://.../.well-known/acme-challenge/` | DNS points elsewhere, or a redirect or proxy intercepts the path | `dig +short example.com`, disable CDN proxying during issuance |
| `too many certificates already issued` | Rate limit hit | Wait for the window or reuse your existing certificate |
| Browser still shows the old certificate | Nginx not reloaded after renewal | Add the deploy hook above |

Logs are in `/var/log/letsencrypt/letsencrypt.log`.

## Step 5: Monitor Your HTTPS Endpoint

Auto-renewal can still fail quietly. DNS changes, a firewall rule, a CDN toggle, or a removed server block can each break the HTTP-01 challenge, and since Let's Encrypt no longer emails you, the first alert is often a customer screenshot of `NET::ERR_CERT_DATE_INVALID`.

Add two safety nets:

**1. A local expiry check** that you can run from cron:

```bash
#!/bin/sh
DOMAIN=example.com
END=$(echo | openssl s_client -connect "$DOMAIN:443" -servername "$DOMAIN" 2>/dev/null \
  | openssl x509 -noout -enddate | cut -d= -f2)
DAYS=$(( ( $(date -d "$END" +%s) - $(date +%s) ) / 86400 ))
echo "$DOMAIN expires in $DAYS days"
[ "$DAYS" -lt 14 ] && exit 1 || exit 0
```

**2. External monitoring** of the HTTPS URL. [GreenCrew](https://greencrew.space) checks your site from outside your server, and an expired or invalid certificate makes the HTTPS check fail. You get a Discord, Slack, or Telegram alert right away, and your public status page shows the incident to users automatically. That also covers the case the cron script can't: the server itself being offline. To wire up the alert channels, see [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Is Let's Encrypt really free for commercial sites?

Yes. Let's Encrypt is a nonprofit certificate authority run by the Internet Security Research Group (ISRG). Its certificates are free for any use, commercial sites included. It issues domain-validated (DV) certificates only, not OV or EV.

### How long does a Let's Encrypt certificate last?

The standard lifetime has been 90 days, and Let's Encrypt is phasing in 45-day certificates over 2026 to 2028. Certbot renews ahead of expiry automatically, so the lifetime only becomes a problem when renewal breaks, which is why monitoring matters.

### How do I force a Certbot renewal?

Run `sudo certbot renew --force-renewal`, or `sudo certbot renew --cert-name example.com --force-renewal` for one certificate. Use it sparingly: forced renewals count toward the duplicate-certificate limit of 5 per week.

### Can I use Let's Encrypt behind Cloudflare?

Yes. Either pause the Cloudflare proxy (grey cloud) while issuing with HTTP-01, or use the `certbot-dns-cloudflare` plugin with DNS-01, which works with the proxy on and supports wildcards.

## Conclusion

Installing Let's Encrypt on Nginx takes one Certbot command. Keeping it working takes three habits: confirm the renewal timer with `certbot renew --dry-run`, reload Nginx after every renewal, and watch your HTTPS endpoint from outside. With expiry emails gone and lifetimes getting shorter, an external monitor such as [GreenCrew](https://greencrew.space) is the easiest way to catch a failed renewal before your visitors do.
