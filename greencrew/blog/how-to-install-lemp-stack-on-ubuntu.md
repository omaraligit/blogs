# How to Install a LEMP Stack on Ubuntu (Nginx, MySQL, and PHP-FPM Step by Step)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, MySQL, PHP, LEMP, Linux, DevOps

Description: To install a LEMP stack on Ubuntu, run sudo apt install nginx mysql-server php-fpm php-mysql, secure MySQL with mysql_secure_installation, and point an Nginx server block's fastcgi_pass at the PHP-FPM socket. This guide covers every step, creating a database user, testing the PHP to MySQL connection, and fixing common LEMP errors.

Date: 2026-10-15

**Short answer:** A LEMP stack is Linux, Nginx ("engine-x"), MySQL, and PHP. On Ubuntu you install it with `sudo apt install nginx mysql-server php-fpm php-mysql`. Next, run `sudo mysql_secure_installation`, create a database and a dedicated MySQL user for your app, and add a server block whose `location ~ \.php$` passes requests to `unix:/run/php/php8.3-fpm.sock` (use the socket name from `ls /run/php/`). Run `sudo nginx -t && sudo systemctl reload nginx`, then load a small PHP script that connects to MySQL to confirm all three layers work together. The whole setup takes about 15 minutes on a fresh server.

## What Is a LEMP Stack?

LEMP is the Nginx-based alternative to the classic LAMP stack (Linux, Apache, MySQL, PHP). It runs WordPress, Laravel, Drupal, Magento, and most other PHP applications. Each layer has one job:

| Layer | Software | Job |
| --- | --- | --- |
| **L**inux | Ubuntu 24.04 LTS (or 22.04) | Operating system, packages, firewall |
| **E**ngine-X | Nginx | Accepts HTTP/HTTPS requests, serves static files, forwards `.php` requests |
| **M**ySQL | MySQL Server (or MariaDB) | Stores application data |
| **P**HP | PHP-FPM | Runs PHP code in a pool of worker processes |

### LEMP vs LAMP

| | LEMP (Nginx) | LAMP (Apache) |
| --- | --- | --- |
| How PHP runs | Separate PHP-FPM service over FastCGI | Usually `mod_php` inside Apache, or PHP-FPM |
| `.htaccess` files | Not supported. Rules live in the server block | Supported per directory |
| Static files under load | Event-driven, low memory per connection | Process/thread-based by default |
| Config style | One server block per site | VirtualHost per site plus `.htaccess` |

If your app ships `.htaccess` rewrite rules, you'll translate them into Nginx `location` and `try_files` rules. For most frameworks that's a single line, shown in Step 4.

## Prerequisites

- An Ubuntu 24.04 or 22.04 server with a non-root user that has `sudo`
- A domain pointing at the server's IP (optional for testing, required for HTTPS)
- Ports 22, 80, and 443 reachable

Commands in this guide use the package versions Ubuntu 24.04 ships (PHP 8.3, MySQL 8.0). On other releases the version numbers in paths change, so check them with the commands shown rather than copying paths blindly.

## Step 1: Install Nginx

```bash
sudo apt update
sudo apt install nginx
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

Check that it's serving the default page:

```bash
curl -I http://localhost
# HTTP/1.1 200 OK
# Server: nginx/...
```

For the details (UFW profiles, the `sites-available` layout, and managing the service), see [how to install Nginx on Ubuntu](https://greencrew.space/blog/how-to-install-nginx-on-ubuntu).

## Step 2: Install and Secure MySQL

```bash
sudo apt install mysql-server
sudo systemctl status mysql
```

Run the hardening script. It removes anonymous users and the test database and can enforce a password policy:

```bash
sudo mysql_secure_installation
```

| Prompt | Recommended answer |
| --- | --- |
| VALIDATE PASSWORD component | Yes, if you want MySQL to reject weak passwords |
| Remove anonymous users? | Yes |
| Disallow root login remotely? | Yes |
| Remove test database? | Yes |
| Reload privilege tables now? | Yes |

On Ubuntu, the MySQL `root` account authenticates with the `auth_socket` plugin by default. You log in with `sudo mysql` and no password, because MySQL trusts the Linux root user. Leave it that way and **never let your application connect as root**.

### Create a Database and an App User

```bash
sudo mysql
```

```sql
CREATE DATABASE appdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'use-a-long-random-password';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Test the new account:

```bash
mysql -u appuser -p appdb -e "SELECT CURRENT_USER();"
```

Granting privileges on `appdb.*` (not `*.*`) means a compromised app can only touch its own database. `utf8mb4` is the character set that stores the full Unicode range, emoji included.

**Prefer MariaDB?** `sudo apt install mariadb-server` is a drop-in alternative. The commands above work the same way, and `php-mysql` talks to both.

## Step 3: Install PHP-FPM and Extensions

```bash
sudo apt install php-fpm php-mysql php-curl php-mbstring php-xml php-zip php-gd php-intl
php -v
ls /run/php/
# php8.3-fpm.pid  php8.3-fpm.sock  php-fpm.sock
```

| Package | Why you need it |
| --- | --- |
| `php-fpm` | The FastCGI process manager Nginx talks to |
| `php-mysql` | `mysqli` and `pdo_mysql` drivers for MySQL/MariaDB |
| `php-curl` | Outgoing HTTP requests (APIs, updates) |
| `php-mbstring` | Multibyte string handling, required by most frameworks |
| `php-xml` | XML/DOM parsing, required by WordPress and Laravel |
| `php-zip`, `php-gd`, `php-intl` | Archives, image resizing, and localization |

Don't install the plain `php` metapackage on a LEMP server unless you check what it pulls in. Depending on the release it can bring in Apache's `libapache2-mod-php`, and a running Apache will fight Nginx for port 80.

## Step 4: Configure the Nginx Server Block for PHP

Create the web root and a server block:

```bash
sudo mkdir -p /var/www/example.com/public
sudo chown -R $USER:www-data /var/www/example.com
sudo nano /etc/nginx/sites-available/example.com
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/example.com/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.(?!well-known) {
        deny all;
    }

    client_max_body_size 32m;
}
```

Enable it, remove the default site, and reload:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

| Directive | What it does in a LEMP stack |
| --- | --- |
| `try_files $uri $uri/ /index.php?$query_string` | Serves real files directly and routes everything else to your PHP front controller (the Nginx version of the usual `.htaccess` rewrite) |
| `include snippets/fastcgi-php.conf` | Sets `SCRIPT_FILENAME` and blocks the path-info exploit |
| `fastcgi_pass` | Where PHP-FPM listens; must match the socket in `/run/php/` |
| `location ~ /\.(?!well-known)` | Hides `.env`, `.git`, and other dotfiles but still allows Let's Encrypt challenges |
| `client_max_body_size` | Max upload size; the Nginx default is 1 MB |

If you raise `client_max_body_size`, raise `upload_max_filesize` and `post_max_size` in `/etc/php/8.3/fpm/php.ini` to match. Pool sizing (`pm.max_children`) and FastCGI timeouts are covered in [how to configure PHP-FPM with Nginx](https://greencrew.space/blog/how-to-configure-php-fpm-with-nginx).

## Step 5: Test PHP and the MySQL Connection

A `phpinfo()` page proves Nginx can reach PHP. It doesn't prove PHP can reach MySQL, so test both with one script:

```bash
nano /var/www/example.com/public/dbtest.php
```

```php
<?php
$dsn = 'mysql:host=localhost;dbname=appdb;charset=utf8mb4';
try {
    $pdo = new PDO($dsn, 'appuser', 'use-a-long-random-password', [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    ]);
    $version = $pdo->query('SELECT VERSION()')->fetchColumn();
    echo "PHP " . PHP_VERSION . " connected to MySQL $version via " . php_sapi_name();
} catch (PDOException $e) {
    http_response_code(500);
    echo "DB connection failed: " . $e->getMessage();
}
```

```bash
curl http://example.com/dbtest.php
# PHP 8.3.x connected to MySQL 8.0.x via fpm-fcgi
rm /var/www/example.com/public/dbtest.php
```

`fpm-fcgi` confirms the request went Nginx → PHP-FPM, and the version string confirms PHP → MySQL. **Delete the test file afterward.** It contains credentials.

In a real app, keep credentials in an environment file (for example Laravel's `.env` or WordPress's `wp-config.php`) outside the public web root, or at least behind the dotfile `deny` rule above.

## Step 6: Add HTTPS

With DNS pointed at the server, Certbot gets a free certificate and edits the server block for you:

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

The full walkthrough, including auto-renewal checks, is in [how to install a Let's Encrypt SSL certificate on Nginx with Certbot](https://greencrew.space/blog/how-to-install-lets-encrypt-ssl-nginx-certbot). After that, add [security headers](https://greencrew.space/blog/how-to-add-security-headers-in-nginx) and [Gzip/Brotli compression](https://greencrew.space/blog/how-to-enable-gzip-and-brotli-compression-in-nginx).

## LEMP File and Service Cheat Sheet

| What | Path or command (Ubuntu 24.04) |
| --- | --- |
| Nginx site configs | `/etc/nginx/sites-available/`, enabled via symlink in `sites-enabled/` |
| Nginx logs | `/var/log/nginx/access.log`, `/var/log/nginx/error.log` |
| PHP-FPM config | `/etc/php/8.3/fpm/php.ini`, pools in `/etc/php/8.3/fpm/pool.d/` |
| PHP-FPM log | `/var/log/php8.3-fpm.log` |
| MySQL config | `/etc/mysql/mysql.conf.d/mysqld.cnf` |
| MySQL error log | `/var/log/mysql/error.log` |
| Reload Nginx | `sudo nginx -t && sudo systemctl reload nginx` |
| Reload PHP-FPM | `sudo php-fpm8.3 -t && sudo systemctl reload php8.3-fpm` |
| Restart MySQL | `sudo systemctl restart mysql` |

## Troubleshooting a LEMP Stack

| Symptom | Cause | Fix |
| --- | --- | --- |
| Browser downloads `.php` files | No `location ~ \.php$` block, or the site isn't enabled | Add the block, check the symlink, reload Nginx |
| 502 Bad Gateway | PHP-FPM stopped or wrong socket path | `ls /run/php/`, fix `fastcgi_pass`, `sudo systemctl start php8.3-fpm` |
| "File not found." | `root` is wrong or `SCRIPT_FILENAME` isn't set | Check `root`, use `snippets/fastcgi-php.conf` |
| `SQLSTATE[HY000] [2002] No such file or directory` | MySQL is down or PHP can't find its socket | `sudo systemctl status mysql`; try `host=127.0.0.1` to use TCP |
| `SQLSTATE[HY000] [1045] Access denied` | Wrong user, password, or host part (`'appuser'@'localhost'`) | Recreate the user or fix credentials |
| `could not find driver` | `php-mysql` not installed for FPM | `sudo apt install php-mysql && sudo systemctl reload php8.3-fpm` |
| `413 Request Entity Too Large` | Upload bigger than `client_max_body_size` | Raise it in Nginx, and match it in `php.ini` |
| Port 80 already in use | Apache was installed alongside | `sudo systemctl disable --now apache2` |
| Blank white page | PHP fatal error with `display_errors` off | Check `/var/log/nginx/error.log` and the PHP-FPM log |

For gateway errors in depth, see [how to fix 502 Bad Gateway in Nginx](https://greencrew.space/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Your LEMP Stack

A LEMP stack has three services that can fail independently. Nginx can stay up while PHP-FPM runs out of workers or MySQL crashes after running out of memory, and the static homepage keeps loading while every dynamic page errors.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks against your site from outside your network and alerts you on Discord, Slack, or Telegram when it stops responding correctly, with a public status page you can share with users. Point the check at a URL that runs PHP *and* queries the database (a lightweight `/health` route works well). That way the check fails when any layer fails. See [how to get website downtime alerts on Discord, Slack, and Telegram](https://greencrew.space/blog/how-to-get-website-downtime-alerts-discord-slack-telegram) to set it up.

## Frequently Asked Questions

### What does LEMP stand for?

Linux, "Engine-X" (Nginx), MySQL (or MariaDB), and PHP. The "E" comes from how Nginx is pronounced. It's the same idea as LAMP with Apache swapped for Nginx.

### Should I use MySQL or MariaDB?

Either works for most PHP apps, and the setup steps are nearly identical. Use MySQL if your app or host documents MySQL 8 specifically. Use MariaDB if you prefer its community governance or your distribution defaults to it. Check your app's requirements before choosing, since the two have diverged in some features.

### Do I need phpMyAdmin?

No. `sudo mysql` and a desktop client over an SSH tunnel cover most needs. If you do install phpMyAdmin, protect it with HTTPS, a non-obvious path, and HTTP basic auth or an IP allowlist, since it's a common target for scanners.

### Can I run multiple sites on one LEMP server?

Yes. Give each site its own server block, web root, database, and MySQL user. For stronger isolation, give each site its own PHP-FPM pool running as a separate Linux user, with its own socket for `fastcgi_pass`.

## Conclusion

Installing a LEMP stack on Ubuntu comes down to four packages, one hardening script, one database user, and one server block. Test the full chain (Nginx → PHP-FPM → MySQL) with a real connection, not just `phpinfo()`, then add HTTPS. Tune the [PHP-FPM pool](https://greencrew.space/blog/how-to-configure-php-fpm-with-nginx) once you have real traffic, and let [GreenCrew](https://greencrew.space) watch the site from the outside so a stopped PHP-FPM or a crashed database reaches you as an alert, not as a complaint from a user.
