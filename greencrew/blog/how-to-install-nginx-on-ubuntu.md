# How to Install Nginx on Ubuntu (apt install nginx, UFW Firewall, and Your First Server Block)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Ubuntu, Linux, Web Server, DevOps

Description: To install Nginx on Ubuntu, run sudo apt update && sudo apt install nginx, allow web traffic with sudo ufw allow 'Nginx Full', and check it with systemctl status nginx and curl -I http://localhost. This guide also covers server blocks, the config file layout, the official nginx.org repository, service commands, and common install errors.

Date: 2026-10-13

**Short answer:** run `sudo apt update && sudo apt install nginx`. Ubuntu starts and enables the service automatically, so it's already running. Open the firewall with `sudo ufw allow 'Nginx Full'` (allow `OpenSSH` first if UFW isn't enabled yet), then visit `http://your_server_ip` and you should see the "Welcome to nginx!" page. To host your own site, create a server block in `/etc/nginx/sites-available/`, symlink it into `sites-enabled/`, and run `sudo nginx -t && sudo systemctl reload nginx`. The steps below work on current Ubuntu LTS releases such as 22.04 and 24.04.

## What Is Nginx?

[Nginx](https://nginx.org/en/) is an open source web server that also works as a reverse proxy, load balancer, and HTTP cache. It uses an event-driven architecture, so a small number of worker processes can handle a large number of concurrent connections with modest memory. That's why it's the default choice for serving static sites, fronting Node.js, Python, and PHP apps, and terminating TLS.

You have two ways to install it on Ubuntu:

| Source | Command | Version | Config layout | Best for |
| --- | --- | --- | --- | --- |
| Ubuntu repository | `apt install nginx` | Fixed per Ubuntu release, gets security patches | `sites-available/` + `sites-enabled/` | Most servers, simplest updates |
| nginx.org repository | Add repo, then `apt install nginx` | Latest stable or mainline | `conf.d/*.conf` | When you need newer features |

Start with the Ubuntu package unless you know you need a feature from a newer release.

## Prerequisites

- An Ubuntu server with a non-root user that has `sudo`
- SSH access
- Optional: a domain name with an A record pointing to the server's IP (needed later for HTTPS)

## Step 1: Install Nginx from the Ubuntu Repository

```bash
sudo apt update
sudo apt install -y nginx
```

Check the installed version and confirm the service is running:

```bash
nginx -v
systemctl status nginx --no-pager
```

You should see `Active: active (running)`. Ubuntu enables the service at boot by default. You can confirm that with `systemctl is-enabled nginx`.

## Step 2: Allow Nginx Through the UFW Firewall

Nginx registers three UFW application profiles when it installs:

```bash
sudo ufw app list
# Available applications:
#   Nginx Full
#   Nginx HTTP
#   Nginx HTTPS
#   OpenSSH
```

| Profile | Opens | Use it when |
| --- | --- | --- |
| `Nginx HTTP` | Port 80 | Plain HTTP only, before you add TLS |
| `Nginx HTTPS` | Port 443 | HTTPS only |
| `Nginx Full` | Ports 80 and 443 | Most sites: HTTPS plus the HTTP-to-HTTPS redirect |

Allow SSH **before** enabling the firewall so you don't lock yourself out:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

If your server runs on a cloud provider, also check its security group or cloud firewall. UFW rules don't help if the provider is blocking port 80 upstream.

## Step 3: Check That Nginx Is Serving Pages

From the server:

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
# Server: nginx/...
```

From your own machine, open `http://your_server_ip` in a browser. To find the public IP:

```bash
curl -4 -s https://icanhazip.com
```

If you see the "Welcome to nginx!" page, the install worked.

## Step 4: Manage the Nginx Service

| Task | Command |
| --- | --- |
| Test the configuration | `sudo nginx -t` |
| Reload config without dropping connections | `sudo systemctl reload nginx` |
| Restart (full stop and start) | `sudo systemctl restart nginx` |
| Stop | `sudo systemctl stop nginx` |
| Start | `sudo systemctl start nginx` |
| Disable start at boot | `sudo systemctl disable nginx` |
| Show the full effective config | `sudo nginx -T` |

Use `reload` for config changes. It starts new workers with the new config and lets old workers finish their requests, so visitors don't see errors. Always run `nginx -t` first. If the test fails, a reload is refused and the old config keeps running, but a `restart` with a broken config leaves Nginx stopped.

## Step 5: Learn the Nginx File Layout on Ubuntu

| Path | What lives there |
| --- | --- |
| `/etc/nginx/nginx.conf` | Main config: worker settings, `http` block, includes |
| `/etc/nginx/sites-available/` | One file per site (server block); not active by itself |
| `/etc/nginx/sites-enabled/` | Symlinks to the sites that are live |
| `/etc/nginx/conf.d/` | Extra `http`-level config, loaded automatically (`*.conf`) |
| `/etc/nginx/snippets/` | Reusable fragments, such as `fastcgi-php.conf` |
| `/var/www/html/` | The default site's web root |
| `/var/log/nginx/access.log` | Every request |
| `/var/log/nginx/error.log` | Errors, failed upstream connections, config warnings |

The `sites-available` / `sites-enabled` split is a Debian and Ubuntu convention. It lets you disable a site by removing a symlink without deleting its config.

## Step 6: Create Your First Server Block

A server block (Nginx's version of an Apache virtual host) tells Nginx which domain to answer for and where its files live. Create a web root and a test page:

```bash
sudo mkdir -p /var/www/example.com/html
sudo chown -R $USER:$USER /var/www/example.com/html
echo '<h1>example.com is working</h1>' > /var/www/example.com/html/index.html
```

Create `/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name example.com www.example.com;

    root /var/www/example.com/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log;
}
```

Enable it, and optionally disable the default site:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default   # optional
sudo nginx -t && sudo systemctl reload nginx
```

Test it before DNS has propagated by sending the right `Host` header:

```bash
curl -H "Host: example.com" http://your_server_ip
```

### Key Server Block Directives

| Directive | Meaning |
| --- | --- |
| `listen 80;` / `listen [::]:80;` | Accept IPv4 and IPv6 connections on port 80 |
| `server_name` | Domain names this block answers for |
| `root` | Directory that files are served from |
| `index` | File to serve for a directory request |
| `try_files $uri $uri/ =404;` | Serve the file or directory if it exists, otherwise return 404 |

If you add many long domain names, `nginx -t` may complain with `could not build server_names_hash`. Uncomment `server_names_hash_bucket_size 64;` in `/etc/nginx/nginx.conf`.

## Optional: Install the Latest Nginx from nginx.org

The [official nginx.org packages](https://nginx.org/en/linux_packages.html#Ubuntu) track the current stable release. Install the prerequisites and the signing key:

```bash
sudo apt install -y curl gnupg2 ca-certificates lsb-release ubuntu-keyring
curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
```

Add the stable repository and pin it above Ubuntu's own package:

```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list

printf "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
  | sudo tee /etc/apt/preferences.d/99nginx

sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Replace `packages/ubuntu` with `packages/mainline/ubuntu` for the mainline branch. Before switching, check the nginx.org docs for the exact steps for your release, and note the differences: this package uses `/etc/nginx/conf.d/*.conf` for sites (no `sites-enabled`), runs workers as the `nginx` user instead of `www-data`, and doesn't ship Ubuntu's `snippets/` directory.

## Next Steps After Installing Nginx

A fresh install serves plain HTTP. For a production site, work through these in order:

1. **Add HTTPS:** [install a free Let's Encrypt certificate with Certbot](https://greencrew.space/blog/how-to-install-lets-encrypt-ssl-nginx-certbot). For a server without a public domain, [create a self-signed certificate](https://greencrew.space/blog/how-to-create-a-self-signed-ssl-certificate-for-nginx).
2. **Proxy to your app:** [set up Nginx as a reverse proxy](https://greencrew.space/blog/how-to-set-up-nginx-reverse-proxy) for Node.js, Python, or Go, or [configure PHP-FPM](https://greencrew.space/blog/how-to-configure-php-fpm-with-nginx) for PHP.
3. **Harden it:** [add security headers](https://greencrew.space/blog/how-to-add-security-headers-in-nginx) and [rate limiting](https://greencrew.space/blog/how-to-set-up-rate-limiting-in-nginx).
4. **Speed it up:** [enable Gzip and Brotli compression](https://greencrew.space/blog/how-to-enable-gzip-and-brotli-compression-in-nginx).

Also hide the exact version from error pages and the `Server` header by adding this to the `http` block in `nginx.conf`:

```nginx
server_tokens off;
```

## Troubleshooting Nginx Installation on Ubuntu

| Symptom | Cause | Fix |
| --- | --- | --- |
| `bind() to 0.0.0.0:80 failed (98: Address already in use)` | Apache or another service holds port 80 | `sudo ss -tlnp \| grep ':80'`, then stop and disable it: `sudo systemctl disable --now apache2` |
| Browser times out, `curl localhost` works | Firewall or cloud security group blocks port 80 | `sudo ufw allow 'Nginx Full'` and open the port in your provider's firewall |
| Still seeing "Welcome to nginx!" for your domain | Default site matched first or your site isn't enabled | Check the symlink in `sites-enabled/`, remove `default`, reload |
| `403 Forbidden` | No index file, or Nginx can't read the web root | Add `index.html`, check permissions with `namei -l /var/www/example.com/html/index.html` |
| `404 Not Found` for every page | Wrong `root` path | Fix `root` and run `sudo nginx -t` |
| `nginx: [emerg] unknown directive` | Typo, or a module that isn't installed | Check spelling, and check the module with `nginx -V` |
| `apt` says `Unable to locate package nginx` | Package lists out of date | Run `sudo apt update` first |
| Changes don't take effect | Config not reloaded, or edited the wrong file | `sudo nginx -T \| grep server_name` to see what's loaded |

The error log almost always tells you exactly what's wrong:

```bash
sudo tail -n 50 /var/log/nginx/error.log
sudo journalctl -u nginx --no-pager -n 50
```

## Monitor Your New Nginx Server

Once your site is live, you need to know when it stops responding, whether that's a bad config reload, a full disk, or the server going away. Checking from the server itself won't help when the server is the thing that's down.

[GreenCrew](https://greencrew.space) checks your site over HTTP from outside your network and alerts you on Discord, Slack, or Telegram when it goes down. You also get a public status page to share with your users. Add a check as soon as your first server block is live, and see [how to get website downtime alerts on Discord, Slack, and Telegram](https://greencrew.space/blog/how-to-get-website-downtime-alerts-discord-slack-telegram) for the full setup.

## Frequently Asked Questions

### How do I check if Nginx is installed on Ubuntu?

Run `nginx -v` to print the version, and `systemctl status nginx` to see whether it's running. If the command isn't found, it isn't installed. `dpkg -l | grep nginx` lists the installed Nginx packages.

### Should I use the Ubuntu package or the nginx.org repository?

Use the Ubuntu package for most servers. It gets security fixes through normal `apt upgrade` and fits Ubuntu's config layout. Use the nginx.org repository when you need a feature or fix that's only in a newer Nginx release.

### What's the difference between reload and restart?

`reload` applies a new config gracefully. Existing connections finish on old workers, and new ones go to new workers. `restart` stops and starts the whole service, which briefly drops connections. Use `reload` for config changes.

### How do I uninstall Nginx on Ubuntu?

Run `sudo apt remove nginx nginx-common` to remove the packages but keep config files, or `sudo apt purge nginx nginx-common` to also delete `/etc/nginx`. Then run `sudo apt autoremove` to clean up dependencies. Your files in `/var/www` aren't removed.

## Conclusion

Installing Nginx on Ubuntu takes one `apt install`, one UFW rule, and a quick `curl` check. From there, each site gets its own server block in `sites-available`, enabled with a symlink and applied with `nginx -t && systemctl reload nginx`. Add HTTPS next, then put [GreenCrew](https://greencrew.space) in front of it so you hear about downtime from an alert instead of from your users.
