# How to Create a Self-Signed SSL Certificate for Nginx (openssl Command, Nginx Config, and Trusting It)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, SSL, TLS, OpenSSL, DevOps

Description: To create a self-signed certificate for Nginx, run openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout nginx-selfsigned.key -out nginx-selfsigned.crt with a subjectAltName, then point ssl_certificate and ssl_certificate_key at the files in a listen 443 ssl server block. This guide covers the full config, the HTTP-to-HTTPS redirect, trusting the certificate on Linux and macOS, mkcert, and fixing browser errors.

Date: 2026-10-13

**Short answer:** generate a key and certificate in one command with `sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt -subj "/CN=example.local" -addext "subjectAltName=DNS:example.local"`. In your Nginx server block, add `listen 443 ssl;`, `ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;`, and `ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;`, then run `sudo nginx -t && sudo systemctl reload nginx`. Traffic is now encrypted, but browsers will show a warning because no trusted authority signed the certificate. That's fine for internal tools, staging, and local development. For a public website, use a free [Let's Encrypt certificate](https://greencrew.space/blog/how-to-install-lets-encrypt-ssl-nginx-certbot) instead.

## What Is a Self-Signed SSL Certificate?

A self-signed certificate is a TLS certificate signed by its own private key rather than by a certificate authority (CA). It encrypts traffic exactly like a CA-issued certificate. The difference is **trust**: browsers and operating systems ship with a list of trusted CAs, and your self-signed certificate isn't on it. So the browser can't verify that the server is who it claims to be, and shows a warning like `NET::ERR_CERT_AUTHORITY_INVALID`.

| | Self-signed | Let's Encrypt | Commercial CA |
| --- | --- | --- | --- |
| Cost | Free | Free | Paid |
| Browser trust | No (unless you install it) | Yes | Yes |
| Needs a public domain | No | Yes | Yes |
| Works for IPs and `.local` / internal names | Yes | No | Rarely |
| Validity | Whatever you choose | Short-lived, auto-renewed | Set by the CA |
| Good for | Internal tools, staging, dev, homelabs | Public websites | Public sites with specific requirements |

Use self-signed certificates when you control every client that connects, or when the site has no public DNS name.

## Step 1: Generate the Key and Certificate with OpenSSL

On Ubuntu or Debian, OpenSSL is installed by default. Run:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/CN=example.local" \
  -addext "subjectAltName=DNS:example.local,DNS:www.example.local,IP:192.168.1.50"
```

Replace the names and IP with the hostnames and addresses people will actually use to reach the server.

| Option | Meaning |
| --- | --- |
| `req -x509` | Create a self-signed certificate instead of a certificate signing request |
| `-nodes` | Don't encrypt the private key with a passphrase, so Nginx can start without prompting |
| `-days 365` | Valid for one year |
| `-newkey rsa:2048` | Generate a new 2048-bit RSA key at the same time |
| `-keyout` / `-out` | Where to write the private key and the certificate |
| `-subj "/CN=..."` | Set the subject without the interactive questions |
| `-addext "subjectAltName=..."` | The names and IPs the certificate is valid for |

**Don't skip `subjectAltName`.** Modern browsers ignore the Common Name (CN) and match the hostname only against the Subject Alternative Name list. A certificate with just a CN fails with `NET::ERR_CERT_COMMON_NAME_INVALID` even after you trust it. The `-addext` flag needs OpenSSL 1.1.1 or newer, which every current Ubuntu LTS release has. Check with `openssl version`.

Prefer a smaller, faster key? Generate an ECDSA P-256 key instead:

```bash
sudo openssl req -x509 -nodes -days 365 \
  -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -keyout /etc/ssl/private/nginx-selfsigned.key \
  -out /etc/ssl/certs/nginx-selfsigned.crt \
  -subj "/CN=example.local" \
  -addext "subjectAltName=DNS:example.local"
```

Lock down the private key:

```bash
sudo chmod 600 /etc/ssl/private/nginx-selfsigned.key
```

Check what you generated:

```bash
openssl x509 -in /etc/ssl/certs/nginx-selfsigned.crt -noout -subject -dates -ext subjectAltName
```

## Step 2: Create a Reusable SSL Snippet

Keeping TLS settings in a snippet makes it easy to reuse them across server blocks. Create `/etc/nginx/snippets/self-signed.conf`:

```nginx
ssl_certificate     /etc/ssl/certs/nginx-selfsigned.crt;
ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
```

And `/etc/nginx/snippets/ssl-params.conf` with modern defaults:

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;
```

Older tutorials also generate a `dhparam.pem` file with `openssl dhparam`. That's only used by `DHE` cipher suites. With TLS 1.3 and the ECDHE suites that Nginx and OpenSSL prefer by default, you can skip it. Mozilla's [SSL Configuration Generator](https://ssl-config.mozilla.org/) gives tested settings if you want to tune ciphers further.

## Step 3: Configure Nginx to Use the Certificate

Edit your site, for example `/etc/nginx/sites-available/example.local`:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name example.local www.example.local;

    include snippets/self-signed.conf;
    include snippets/ssl-params.conf;

    root /var/www/example.local/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name example.local www.example.local;
    return 301 https://$host$request_uri;
}
```

The second block redirects all HTTP traffic to HTTPS. If you want HTTP/2, add `http2 on;` inside the 443 server block on Nginx 1.25.1 and newer. Older versions use `listen 443 ssl http2;` instead.

Avoid adding `Strict-Transport-Security` (HSTS) on a self-signed site unless every client trusts the certificate. HSTS removes the browser's "proceed anyway" option, which can lock people out. The [security headers guide](https://greencrew.space/blog/how-to-add-security-headers-in-nginx) explains HSTS in detail.

## Step 4: Open the Firewall and Reload Nginx

```bash
sudo ufw allow 'Nginx Full'
sudo nginx -t && sudo systemctl reload nginx
```

New to Nginx on Ubuntu? [How to install Nginx on Ubuntu](https://greencrew.space/blog/how-to-install-nginx-on-ubuntu) covers UFW profiles and server blocks.

## Step 5: Test the HTTPS Connection

`curl` refuses an untrusted certificate by default. Either skip verification with `-k`, or (better) tell it to trust your certificate:

```bash
curl -kI https://example.local
curl -I --cacert /etc/ssl/certs/nginx-selfsigned.crt https://example.local
```

To see exactly what the server presents:

```bash
openssl s_client -connect example.local:443 -servername example.local </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

For a self-signed certificate, the subject and issuer are identical.

## Step 6: Trust the Certificate on Your Machines (Optional)

To get rid of the browser warning, add the certificate to each client's trust store. Only do this for certificates you created yourself.

| Platform | Command or steps |
| --- | --- |
| Ubuntu / Debian | `sudo cp nginx-selfsigned.crt /usr/local/share/ca-certificates/nginx-selfsigned.crt && sudo update-ca-certificates` |
| RHEL / Fedora | `sudo cp nginx-selfsigned.crt /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust` |
| macOS | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain nginx-selfsigned.crt` |
| Windows | Open the `.crt` file, click **Install Certificate**, choose **Local Machine**, and place it in **Trusted Root Certification Authorities** |
| Firefox | Uses its own store by default: **Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import** |

On Debian-based systems, the file must end in `.crt` or `update-ca-certificates` ignores it. Restart the browser after trusting the certificate.

### Easier for Local Development: mkcert

If you just need trusted HTTPS on `localhost` or a dev hostname, [mkcert](https://github.com/FiloSottile/mkcert) automates all of this. It creates a local CA, installs it in your system and browser trust stores, and issues certificates signed by it:

```bash
mkcert -install
mkcert example.local localhost 127.0.0.1 ::1
# Creates example.local+3.pem and example.local+3-key.pem
```

Point `ssl_certificate` and `ssl_certificate_key` at those two files. Because the certificates chain to a CA your machine trusts, there's no warning, and you can issue new ones without re-trusting anything. Never share the mkcert root CA key. Anyone who has it can create certificates your machine will trust.

## Renewing a Self-Signed Certificate

Nothing renews a self-signed certificate automatically. When it expires, browsers and API clients start failing with `certificate has expired` errors. Check the expiry date:

```bash
openssl x509 -in /etc/ssl/certs/nginx-selfsigned.crt -noout -enddate
```

To renew, rerun the Step 1 command with the same paths, then reload Nginx. If you installed the old certificate in client trust stores, you'll need to replace it there too. That hassle is the main reason to move to Let's Encrypt as soon as a server gets a public domain.

## Troubleshooting Self-Signed Certificates in Nginx

| Symptom | Cause | Fix |
| --- | --- | --- |
| `NET::ERR_CERT_AUTHORITY_INVALID` | Expected for self-signed certificates | Trust the cert on the client (Step 6) or use mkcert |
| `NET::ERR_CERT_COMMON_NAME_INVALID` after trusting it | No `subjectAltName`, or hostname not in it | Regenerate with `-addext "subjectAltName=DNS:your.host"` |
| `nginx -t`: `key values mismatch` | Certificate and key are from different runs | Regenerate both in one command, or pair the matching files |
| `nginx -t`: `cannot load certificate ... No such file` | Wrong path in `ssl_certificate` | Check the path with `ls -l` |
| Nginx prompts for a PEM pass phrase on start | Key was created without `-nodes` | Recreate the key with `-nodes` |
| `curl: (60) SSL certificate problem: self-signed certificate` | curl doesn't trust it | Use `--cacert` or install the cert system-wide |
| HTTPS times out | Port 443 closed | `sudo ufw allow 'Nginx HTTPS'` and check your cloud firewall |
| Site worked, then suddenly fails | Certificate expired | Check `-enddate`, regenerate, and reload |

## Monitor the Sites Behind Your Certificates

Self-signed certificates usually protect internal dashboards, staging servers, and admin tools, which are exactly the systems nobody notices are down until someone needs them. When you later move the site to a public domain and a real certificate, keep an eye on it from the outside.

[GreenCrew](https://greencrew.space) monitors your public websites and HTTP endpoints from outside your network and alerts you on Discord, Slack, or Telegram when they stop responding. It also gives you a public status page for your users. See [how to get website downtime alerts on Discord, Slack, and Telegram](https://greencrew.space/blog/how-to-get-website-downtime-alerts-discord-slack-telegram) to set up alerts.

## Frequently Asked Questions

### Is a self-signed certificate secure?

The encryption is just as strong as a CA-issued certificate with the same key type. What's missing is identity verification. A client can't tell your certificate apart from an attacker's unless it was told to trust yours specifically. That's why self-signed certificates are fine for systems where you control the clients and unsuitable for public websites.

### Can I use a self-signed certificate for an IP address?

Yes. Add the IP to the SAN list with `IP:192.168.1.50` in the `-addext` value. Public CAs like Let's Encrypt generally don't issue certificates for private IPs, so self-signed or a private CA is the usual option there.

### How long should a self-signed certificate be valid?

One year is a common choice. Longer lifetimes mean fewer renewals but a longer window if the key leaks. Some clients, including Apple platforms, apply their own limits on very long-lived server certificates, so very long validity periods can cause errors even after you've trusted the certificate.

### Why does Nginx need -nodes?

Without `-nodes`, OpenSSL encrypts the private key with a passphrase. Nginx would then need that passphrase every time it starts, which breaks unattended restarts. Protect the unencrypted key with file permissions (`chmod 600`) instead.

## Conclusion

A self-signed certificate for Nginx is one `openssl req -x509` command, as long as you include a `subjectAltName`, plus two `ssl_certificate` lines in a `listen 443 ssl` server block. Trust it on the clients you control, or use mkcert for local development, and set a reminder before it expires. When the site goes public, switch to [Let's Encrypt with Certbot](https://greencrew.space/blog/how-to-install-lets-encrypt-ssl-nginx-certbot) and let [GreenCrew](https://greencrew.space) watch its uptime.
