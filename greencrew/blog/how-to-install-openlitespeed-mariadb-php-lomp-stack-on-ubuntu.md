# How to Install a LOMP Stack on Ubuntu (OpenLiteSpeed, MariaDB, and PHP Step by Step)

Author: [greencrew](https://greencrew.space/blog)

Tags: OpenLiteSpeed, MariaDB, PHP, LOMP, Linux, DevOps

Description: To install a LOMP stack on Ubuntu, add the LiteSpeed repository with wget -O - https://repo.litespeed.sh | sudo bash, run sudo apt install openlitespeed lsphp83 lsphp83-mysql mariadb-server, set the WebAdmin password with admpass.sh, and point the lsphp external app at the PHP version you installed. This guide covers every step, including virtual hosts, port 80, a MariaDB user, Let's Encrypt SSL, and troubleshooting.

Date: 2026-10-28

**Short answer:** A LOMP stack is Linux, OpenLiteSpeed, MariaDB (or MySQL), and PHP. On Ubuntu, add LiteSpeed's repository with `wget -O - https://repo.litespeed.sh | sudo bash`, then run `sudo apt install openlitespeed lsphp83 lsphp83-common lsphp83-mysql mariadb-server`. Set the admin password with `sudo /usr/local/lsws/admin/misc/admpass.sh`, log in to WebAdmin at `https://your-server-ip:7080`, and make sure the `lsphp` external app runs `/usr/local/lsws/lsphp83/bin/lsphp`. Then move the default listener from port 8088 to 80, map your domain to a virtual host, and secure MariaDB with `sudo mariadb-secure-installation`.

## What Is a LOMP Stack?

LOMP is a variation of LAMP and LEMP that replaces Apache or Nginx with **OpenLiteSpeed**, the open-source edition of LiteSpeed Web Server. It's popular for WordPress because of its built-in page cache (LSCache) and its support for Apache-style rewrite rules.

| Layer | Software | Job |
| --- | --- | --- |
| **L**inux | Ubuntu 24.04 LTS (or 22.04) | Operating system, packages, firewall |
| **O**penLiteSpeed | `openlitespeed` package | Event-driven web server with a browser-based admin panel |
| **M**ariaDB | `mariadb-server` | MySQL-compatible database |
| **P**HP | LSPHP (`lsphp83`) | PHP built for LiteSpeed's LSAPI, a faster alternative to FastCGI |

### LOMP vs LEMP vs LAMP

| | LOMP (OpenLiteSpeed) | LEMP (Nginx) | LAMP (Apache) |
| --- | --- | --- | --- |
| How PHP runs | LSPHP over LSAPI | PHP-FPM over FastCGI | `mod_php` or PHP-FPM |
| `.htaccess` | Rewrite rules only, read at restart | Not supported | Fully supported, per request |
| Configuration | WebAdmin GUI (port 7080) plus config files | Text config files | Text config files plus `.htaccess` |
| Built-in page cache | Yes (LSCache module) | `fastcgi_cache` / `proxy_cache` | Add-on modules |
| PHP packages | LiteSpeed repo (`lsphpXX`) | Ubuntu or ondrej PPA (`phpX.Y-fpm`) | Ubuntu or ondrej PPA |
| Docs and community | Smaller | Very large | Very large |

If you're already comfortable editing Nginx config, see [how to install a LEMP stack on Ubuntu](/blog/how-to-install-lemp-stack-on-ubuntu). Choose LOMP when you want a GUI, LSCache for WordPress, or `.htaccess` rewrite compatibility without running Apache.

## Prerequisites

- An Ubuntu 24.04 or 22.04 server with a non-root `sudo` user
- No other web server bound to port 80 or 443. Stop and disable Apache or Nginx first (`sudo systemctl disable --now nginx`).
- A domain pointing at the server's IP (needed for HTTPS)
- Your own public IP address, so you can restrict the admin panel to it

## Step 1: Install OpenLiteSpeed

Add LiteSpeed's official repository and install the server, as described in the [OpenLiteSpeed installation docs](https://docs.openlitespeed.org/installation/):

```bash
sudo apt update
wget -O - https://repo.litespeed.sh | sudo bash
sudo apt install -y openlitespeed
```

Everything lives under `/usr/local/lsws` (`$SERVER_ROOT` in the config). Check that it's running:

```bash
sudo /usr/local/lsws/bin/lswsctrl status
sudo systemctl status lsws
```

| Command | What it does |
| --- | --- |
| `sudo /usr/local/lsws/bin/lswsctrl start` | Start OpenLiteSpeed |
| `sudo /usr/local/lsws/bin/lswsctrl restart` | Graceful restart (reloads config without dropping connections) |
| `sudo /usr/local/lsws/bin/lswsctrl stop` | Stop the server |
| `sudo systemctl enable lsws` | Start on boot |

Out of the box, OpenLiteSpeed serves an **Example** site on port **8088** and the **WebAdmin** console on port **7080**.

## Step 2: Open the Firewall

Allow web traffic, and allow the admin panel only from your own IP:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp                                  # HTTP/3 (QUIC)
sudo ufw allow from 203.0.113.10 to any port 7080 proto tcp   # your IP
sudo ufw enable
sudo ufw status
```

Leave port 8088 closed. You'll move the site to port 80 in Step 5. Never open 7080 to the whole internet. It's a full control panel for your server.

## Step 3: Set the WebAdmin Password

The installer creates an `admin` user. Set your own credentials:

```bash
sudo /usr/local/lsws/admin/misc/admpass.sh
```

Then browse to `https://your-server-ip:7080`. The panel uses a self-signed certificate, so your browser will warn you the first time. That's expected. Log in with the username and password you just set.

## Step 4: Install PHP (LSPHP) and Point OpenLiteSpeed at It

OpenLiteSpeed doesn't use Ubuntu's `php-fpm` packages. It uses **LSPHP** builds from the same LiteSpeed repository. Install a current version with the MySQL extension:

```bash
sudo apt install -y lsphp83 lsphp83-common lsphp83-mysql
apt-cache search lsphp83    # list other extensions (curl, intl, imagick, redis, ...)
```

Each version installs into its own directory:

| Path | Purpose |
| --- | --- |
| `/usr/local/lsws/lsphp83/bin/lsphp` | The LSAPI binary OpenLiteSpeed runs |
| `/usr/local/lsws/lsphp83/bin/php` | CLI binary for cron jobs, WP-CLI, Composer |
| `/usr/local/lsws/lsphp83/etc/php/8.3/litespeed/php.ini` | `php.ini` for web requests |
| `/usr/local/lsws/lsphp83/etc/php/8.3/mods-available/` | Extension `.ini` files |

Now tell the server to use it. In WebAdmin:

1. Go to **Server Configuration → External App** and edit the app named **lsphp**.
2. Set **Command** to `/usr/local/lsws/lsphp83/bin/lsphp` (or `$SERVER_ROOT/lsphp83/bin/lsphp`).
3. Save, then check **Server Configuration → Script Handler**: the `php` suffix should use handler type **LiteSpeed SAPI** and handler name **lsphp**.
4. Click the **Graceful Restart** button in the top-right corner.

If you'd like the `php` command in your shell to use the same version:

```bash
sudo ln -sf /usr/local/lsws/lsphp83/bin/php /usr/local/bin/php
php -v
```

Running multiple PHP versions side by side works the same way: install `lsphpXX`, create a second external app, and give a virtual host its own script handler. That's the LOMP equivalent of [running multiple PHP-FPM versions](/blog/how-to-install-multiple-php-versions-on-ubuntu).

## Step 5: Serve on Port 80 and Create a Virtual Host

### Move the Listener to Port 80

In WebAdmin, open **Listeners**, view the **Default** listener, edit **General**, change **Port** from `8088` to `80`, save, and do a graceful restart. `http://your-server-ip` should now show the OpenLiteSpeed welcome page.

### Create the Site Directory

OpenLiteSpeed runs PHP as `nobody:nogroup` by default on Ubuntu:

```bash
sudo mkdir -p /var/www/example.com/{html,logs}
sudo chown -R nobody:nogroup /var/www/example.com/html
```

### Add the Virtual Host

Go to **Virtual Hosts → Add** and fill in:

| Field | Value |
| --- | --- |
| Virtual Host Name | `example.com` |
| Virtual Host Root | `/var/www/example.com/` |
| Config File | `$SERVER_ROOT/conf/vhosts/$VH_NAME/vhconf.conf` |
| Enable Scripts/ExtApps | Yes |
| Restrained | Yes |

After you save, WebAdmin offers to create the missing config file. Click the link to create it. Then edit the new virtual host:

- **General → Document Root:** `$VH_ROOT/html/`
- **General → Index Files:** `index.php, index.html`
- **Rewrite → Enable Rewrite:** Yes, and **Auto Load from .htaccess:** Yes (needed for WordPress permalinks and Laravel routing)

### Map Your Domain to the Virtual Host

Open **Listeners → Default → Virtual Host Mappings → Add**, pick `example.com`, and set **Domains** to `example.com, www.example.com`. Do a graceful restart.

Behind the scenes these settings are written to plain text files you can back up or put under version control:

| File | Contains |
| --- | --- |
| `/usr/local/lsws/conf/httpd_config.conf` | Server-wide settings, listeners, external apps |
| `/usr/local/lsws/conf/vhosts/example.com/vhconf.conf` | Per-site document root, rewrites, SSL overrides |
| `/usr/local/lsws/logs/error.log` | Server error log |
| `/usr/local/lsws/logs/access.log` | Default access log |

## Step 6: Install and Secure MariaDB

```bash
sudo apt install -y mariadb-server
sudo systemctl enable --now mariadb
sudo mariadb-secure-installation   # older releases: mysql_secure_installation
```

Accept the defaults: keep `unix_socket` authentication for root, remove anonymous users, disallow remote root login, and remove the test database. With socket auth, `sudo mariadb` logs you in as root without a password.

Create a database and a dedicated user for your app:

```bash
sudo mariadb <<'SQL'
CREATE DATABASE appdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'change-this-strong-password';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
SQL
```

The [MariaDB Knowledge Base](https://mariadb.com/kb/en/) covers authentication plugins and user management in depth.

## Step 7: Test PHP and the Database Together

`phpinfo()` only proves PHP runs. Test the whole chain with a real query:

```bash
sudo tee /var/www/example.com/html/health.php > /dev/null <<'EOF'
<?php
try {
    $pdo = new PDO('mysql:host=localhost;dbname=appdb', 'appuser', 'change-this-strong-password');
    $pdo->query('SELECT 1');
    http_response_code(200);
    echo 'OK PHP ' . PHP_VERSION . ' (' . PHP_SAPI . ')';
} catch (Throwable $e) {
    http_response_code(503);
    echo 'DB ERROR';
}
EOF
curl -i http://example.com/health.php
```

A `200` with `OK PHP 8.3.x (litespeed)` means OpenLiteSpeed, LSPHP, and MariaDB are all working. Keep this file (with real credentials loaded from config, not hard-coded) as a health endpoint for monitoring.

## Step 8: Add HTTPS with Let's Encrypt

OpenLiteSpeed doesn't have a Certbot plugin like Nginx does, so use webroot mode and point the listener at the certificate files:

```bash
sudo apt install -y certbot
sudo certbot certonly --webroot -w /var/www/example.com/html \
  -d example.com -d www.example.com \
  --deploy-hook "/usr/local/lsws/bin/lswsctrl restart"
```

The `--deploy-hook` restarts OpenLiteSpeed after every successful renewal so it loads the new certificate. In WebAdmin:

1. **Listeners → Add:** name `SSL`, IP `ANY`, port `443`, **Secure: Yes**.
2. On the new listener's **SSL** tab, set:
   - **Private Key File:** `/etc/letsencrypt/live/example.com/privkey.pem`
   - **Certificate File:** `/etc/letsencrypt/live/example.com/fullchain.pem`
   - **Chained Certificate:** Yes
3. Add a **Virtual Host Mapping** for `example.com` on the SSL listener, just like in Step 5.
4. Graceful restart and test `https://example.com`.

Redirect HTTP to HTTPS by adding these rules to the virtual host's **Rewrite → Rewrite Rules** box:

```apache
RewriteCond %{SERVER_PORT} 80
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]
```

Confirm automatic renewal works with `sudo certbot renew --dry-run`. Background on how ACME validation works is in the [Let's Encrypt docs](https://letsencrypt.org/docs/). If you hit browser errors after this step, our [ERR_SSL_PROTOCOL_ERROR guide](/blog/how-to-fix-err-ssl-protocol-error-nginx) covers the same certificate and port mistakes (the commands are for Nginx, but the causes are the same).

## OpenLiteSpeed Security Checklist

| Item | Why | How |
| --- | --- | --- |
| Restrict port 7080 | WebAdmin can change everything | UFW rule for your IP only, or an SSH tunnel |
| Remove the Example vhost | Ships demo scripts | Delete its listener mapping and virtual host |
| Delete test files | `phpinfo()` leaks server details | `sudo rm /var/www/example.com/html/info.php` |
| Keep packages updated | OLS, LSPHP, and MariaDB all come from apt | `sudo apt update && sudo apt upgrade` |
| Hide PHP version | Less information for scanners | `expose_php = Off` in the LSPHP `php.ini` |
| Add security headers | Clickjacking, MIME sniffing, HSTS | Virtual host → General → Header Operations (same headers as in our [security headers guide](/blog/how-to-add-security-headers-in-nginx)) |

## Troubleshooting OpenLiteSpeed and LOMP

| Symptom | Cause | Fix |
| --- | --- | --- |
| Site works on `:8088` but not port 80 | Listener still on 8088, or Apache/Nginx holds port 80 | Change listener port. Check `sudo ss -ltnp \| grep ':80 '` |
| Can't reach `https://ip:7080` | Firewall or cloud security group | Allow 7080 from your IP in UFW *and* in your provider's firewall |
| PHP files download instead of running | Script handler missing or wrong | Script Handler: suffix `php` → `lsphp`, then graceful restart |
| `503 Service Unavailable` on PHP pages | External app command path wrong, or LSPHP version not installed | Check the path exists: `ls /usr/local/lsws/lsphp83/bin/lsphp` |
| `phpinfo()` shows the wrong PHP version | External app still points at the old LSPHP | Update **Command**, graceful restart |
| Config change has no effect | No restart after saving | Click **Graceful Restart**, or run `lswsctrl restart` |
| `.htaccess` rewrite change ignored | OpenLiteSpeed reads `.htaccess` at startup | Restart the server after editing `.htaccess` |
| WordPress permalinks 404 | Rewrite disabled for the vhost | Rewrite → Enable Rewrite + Auto Load from .htaccess → Yes |
| `SQLSTATE[HY000] [1045] Access denied` | Wrong DB user/password or host | Recreate the user for `'appuser'@'localhost'` |
| `could not find driver` | `lsphp83-mysql` missing | `sudo apt install lsphp83-mysql`, restart |
| Old certificate after renewal | Server not restarted | Add `--deploy-hook` (Step 8) or restart manually |

Check `/usr/local/lsws/logs/error.log` and `/var/www/example.com/logs/` first. Most problems are spelled out there.

## Monitor Your LOMP Stack

OpenLiteSpeed's admin panel shows real-time stats, but only while you're looking at it, and only from inside the server. If LSPHP crashes, MariaDB runs out of disk, or a certificate fails to renew, your visitors find out first.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks from outside your network and alerts you on Discord, Slack, or Telegram when your site stops responding correctly. It also gives you a public status page to share with users. Point a check at the `health.php` endpoint from Step 7 so it fails whenever any layer (web server, PHP, or database) fails. Setup is covered in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Is OpenLiteSpeed faster than Nginx?

It depends on the workload, and published benchmarks vary widely. OpenLiteSpeed's main advantages are LSAPI for PHP and the built-in LSCache module, which help most with WordPress. Nginx with PHP-FPM and `fastcgi_cache` is just as capable when tuned. Choose based on how you want to manage the server and what your app needs.

### Can I use MySQL instead of MariaDB in a LOMP stack?

Yes. Install `mysql-server` instead of `mariadb-server`. The LSPHP `mysql` extension works with both. Only the secure-installation command name (`mysql_secure_installation`) and some authentication defaults differ.

### Does OpenLiteSpeed support .htaccess?

Partly. OpenLiteSpeed reads **rewrite rules** from `.htaccess` when the virtual host has Auto Load from .htaccess enabled, but it doesn't re-read the file on every request, and other Apache directives are ignored. Restart the server after changing `.htaccess`. The paid LiteSpeed Enterprise edition has fuller Apache compatibility.

### What's the difference between OpenLiteSpeed and LiteSpeed Enterprise?

OpenLiteSpeed is the free, open-source edition configured through WebAdmin and its own config format. LiteSpeed Enterprise is a commercial drop-in replacement for Apache that reads Apache config files and `.htaccess` directly. It's often bundled with cPanel or Plesk hosting.

## Conclusion

Installing a LOMP stack on Ubuntu comes down to a few steps. Add the LiteSpeed repo, install `openlitespeed` and an `lsphp` version, point the `lsphp` external app at that version, move the listener to port 80, map a virtual host, and secure MariaDB with a dedicated app user. Test the whole chain with a real database query, add Let's Encrypt with a restart hook, and lock port 7080 to your own IP. Then let [GreenCrew](https://greencrew.space) watch your health endpoint, so a stopped LSPHP process or a failed database reaches you as an alert, not as a complaint from a user.
