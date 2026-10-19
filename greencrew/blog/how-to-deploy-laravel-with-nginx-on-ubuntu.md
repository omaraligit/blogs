# How to Deploy Laravel with Nginx on Ubuntu (Server Block, Permissions, Queues, and Scheduler)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Laravel, PHP, Linux, DevOps

Description: To deploy Laravel with Nginx, set the server block's root to your app's public/ directory, add try_files $uri $uri/ /index.php?$query_string; to the location / block, pass index.php to PHP-FPM, and give www-data write access to storage/ and bootstrap/cache. This guide covers PHP extensions, Composer, the .env file, caching, migrations, queue workers, the scheduler, HTTPS, and common Laravel 404/500 errors.

Date: 2026-10-19

**Short answer:** Install PHP-FPM with the extensions Laravel needs, install Composer, and put your app in `/var/www/laravel`. Run `composer install --no-dev --optimize-autoloader`, fill in `.env` with `APP_ENV=production` and `APP_DEBUG=false`, and run `php artisan key:generate` and `php artisan migrate --force`. Point Nginx's `root` at `/var/www/laravel/public` (not the project root), use `try_files $uri $uri/ /index.php?$query_string;`, and send `index.php` to the PHP-FPM socket. Finally, make `storage/` and `bootstrap/cache/` writable by `www-data`, run the queue worker under systemd, and add the scheduler cron entry.

## What Does a Laravel Production Setup on Nginx Look Like?

Laravel is a PHP framework, so it runs on the same LEMP base as any PHP app: Nginx receives requests, serves files from `public/` directly, and hands everything else to PHP-FPM, which runs Laravel's front controller, `public/index.php`. On top of that base, a real Laravel app needs a few extra pieces:

| Component | Job | Covered in |
| --- | --- | --- |
| Nginx | HTTP/HTTPS, static files, routes everything else to `index.php` | Step 5 |
| PHP-FPM | Runs Laravel's PHP code | Step 1 |
| Composer | Installs PHP dependencies from `composer.lock` | Step 2 |
| MySQL / PostgreSQL | Application database | Step 3 |
| `.env` file | Secrets and environment settings | Step 3 |
| Queue worker | Runs queued jobs (mail, notifications, exports) | Step 7 |
| Scheduler | Runs `schedule:run` every minute | Step 8 |

If you haven't set up Nginx, MySQL, and PHP-FPM yet, follow [how to install a LEMP stack on Ubuntu](/blog/how-to-install-lemp-stack-on-ubuntu) first and come back here.

## Prerequisites

- Ubuntu 24.04 or 22.04 with a sudo user
- Nginx, MySQL (or PostgreSQL), and PHP-FPM installed
- A domain pointed at the server for HTTPS
- Your Laravel app in a Git repository

Commands use PHP 8.3, which Ubuntu 24.04 ships. Check which PHP version your Laravel release needs in the [Laravel deployment and installation docs](https://laravel.com/docs/deployment), and replace `8.3` in package names and socket paths if yours differs.

## Step 1: Install the PHP Extensions Laravel Needs

```bash
sudo apt update
sudo apt install php8.3-fpm php8.3-cli php8.3-mysql php8.3-mbstring \
  php8.3-xml php8.3-bcmath php8.3-curl php8.3-zip php8.3-intl unzip git
php -v
php -m | grep -E 'mbstring|xml|pdo_mysql|bcmath|curl|zip|intl'
```

| Package | Why Laravel needs it |
| --- | --- |
| `php8.3-fpm` | Runs PHP for Nginx |
| `php8.3-cli` | Runs `php artisan` and Composer |
| `php8.3-mysql` | PDO driver for MySQL/MariaDB (`php8.3-pgsql` for PostgreSQL) |
| `php8.3-mbstring` | Multibyte strings, required by the framework |
| `php8.3-xml` | DOM and XML functions, required by the framework and many packages |
| `php8.3-bcmath`, `php8.3-intl` | Number and locale formatting, used by many packages |
| `php8.3-curl` | HTTP client (`Http::get()`), external APIs |
| `php8.3-zip`, `unzip` | Lets Composer unpack packages quickly |

Extensions like Ctype, Fileinfo, OpenSSL, PDO, and Tokenizer are compiled into Ubuntu's PHP packages already. If `composer install` complains about a missing extension, install the matching `php8.3-<name>` package and restart PHP-FPM.

## Step 2: Install Composer and Your App

Install Composer by following the official instructions at [getcomposer.org/download](https://getcomposer.org/download/) (they include a checksum check), then move it onto your `PATH`:

```bash
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

Clone the app into a directory your deploy user owns:

```bash
sudo mkdir -p /var/www/laravel
sudo chown $USER:www-data /var/www/laravel
git clone https://github.com/you/your-laravel-app.git /var/www/laravel
cd /var/www/laravel
composer install --no-dev --optimize-autoloader
```

`--no-dev` skips test and debug packages, and `--optimize-autoloader` builds a class map so PHP doesn't search the filesystem for classes on every request. If you build frontend assets with Vite, run `npm ci && npm run build` here too, which writes the compiled files to `public/build`.

## Step 3: Create the Database and the .env File

Create a database and a dedicated user (MySQL shown):

```sql
CREATE DATABASE laravel CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'laravel'@'localhost' IDENTIFIED BY 'a-long-random-password';
GRANT ALL PRIVILEGES ON laravel.* TO 'laravel'@'localhost';
FLUSH PRIVILEGES;
```

Then create the environment file:

```bash
cp .env.example .env
php artisan key:generate
chmod 640 .env
```

Edit `.env` with the production values:

```ini
APP_NAME="My App"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

LOG_CHANNEL=daily
LOG_LEVEL=warning

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=a-long-random-password

QUEUE_CONNECTION=database
```

| Setting | Production value | Why |
| --- | --- | --- |
| `APP_ENV` | `production` | Enables production behavior; `migrate` asks for confirmation unless you pass `--force` |
| `APP_DEBUG` | `false` | **Critical.** `true` shows stack traces and environment details to visitors |
| `APP_KEY` | Set by `key:generate` | Encrypts cookies and sessions. Don't change it on a live app, or every session breaks |
| `APP_URL` | Your HTTPS URL | Used for generated links, emails, and the storage URL |
| `LOG_CHANNEL` | `daily` | One file per day in `storage/logs/` instead of one ever-growing file |

## Step 4: Fix Laravel Storage Permissions

Laravel writes logs, compiled views, sessions, and caches to `storage/` and `bootstrap/cache/`. PHP-FPM runs as `www-data`, so those two directories must be writable by that user. Everything else should stay read-only to the web server.

```bash
cd /var/www/laravel
sudo chown -R $USER:www-data storage bootstrap/cache
sudo chmod -R 775 storage bootstrap/cache
php artisan storage:link
```

`storage:link` creates the `public/storage` symlink so files saved on the `public` disk are reachable at `/storage/...`.

**Don't** fix permission errors with `chmod -R 777`. It makes every file writable by every process on the server, and a single upload bug becomes remote code execution. Group ownership plus `775` on the two writable directories is enough.

## Step 5: Configure the Nginx Server Block for Laravel

Create `/etc/nginx/sites-available/laravel`. This is based on the server block in the [Laravel deployment docs](https://laravel.com/docs/deployment#nginx):

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    root /var/www/laravel/public;

    index index.php;
    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

| Directive | What it does |
| --- | --- |
| `root .../public` | Only `public/` is web-accessible. `.env`, `vendor/`, and `storage/` stay unreachable |
| `try_files $uri $uri/ /index.php?$query_string` | Serves real files directly, sends every other URL to Laravel's router with the query string intact |
| `location ~ ^/index\.php(/\|$)` | Only the front controller is executed. A stray `.php` file in `public/` won't run |
| `$realpath_root` | Resolves symlinks, so symlink-based deploys (like `current -> releases/42`) pick up the new release |
| `error_page 404 /index.php` | Lets Laravel render its own 404 page |
| `location ~ /\.(?!well-known)` | Blocks dotfiles such as `.git` and `.env`, but allows Let's Encrypt challenges |

The FastCGI details, socket vs TCP, and pool tuning (`pm.max_children`) are covered in [how to configure PHP-FPM with Nginx](/blog/how-to-configure-php-fpm-with-nginx).

## Step 6: Run Migrations and Cache the Config

```bash
cd /var/www/laravel
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
```

On recent Laravel versions, `php artisan optimize` runs these cache commands in one go.

| Command | What it caches | Gotcha |
| --- | --- | --- |
| `config:cache` | All `config/*.php` into one file | After caching, `env()` returns `null` outside config files. Read values with `config()` in app code |
| `route:cache` | The compiled route list | Re-run after every deploy that changes routes |
| `view:cache` | Compiled Blade templates | None, just faster first requests |
| `optimize:clear` | Clears all of the above | Use it when "my change isn't showing up" |

Test it: `curl -I http://example.com` should return `200 OK`. Recent Laravel versions (11 and later) register a built-in health route at `/up`, so `curl http://example.com/up` is a quick check that the framework boots.

## Step 7: Run the Laravel Queue Worker with systemd

If your app dispatches jobs, something has to process them. Create `/etc/systemd/system/laravel-worker.service`:

```ini
[Unit]
Description=Laravel queue worker
After=network.target mysql.service

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/laravel
ExecStart=/usr/bin/php artisan queue:work --sleep=3 --tries=3 --max-time=3600
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now laravel-worker
sudo systemctl status laravel-worker
```

`--max-time=3600` makes the worker exit after an hour and systemd starts a fresh one, which keeps memory leaks in check. The [Laravel queue docs](https://laravel.com/docs/queues#supervisor-configuration) use Supervisor instead; either works. Workers keep the old code in memory, so run `php artisan queue:restart` on every deploy.

## Step 8: Add the Laravel Scheduler Cron Entry

Laravel's scheduler needs one cron entry that runs every minute. Add it to the `www-data` crontab so it writes files with the right owner:

```bash
sudo crontab -u www-data -e
```

```cron
* * * * * cd /var/www/laravel && php artisan schedule:run >> /dev/null 2>&1
```

Check it with `php artisan schedule:list`. See the [Laravel scheduling docs](https://laravel.com/docs/scheduling#running-the-scheduler) for details.

## Step 9: Enable HTTPS

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot edits the server block, adds the certificate, and sets up the HTTP to HTTPS redirect. Make sure `APP_URL` uses `https://`, then run `php artisan config:cache` again. The full walkthrough is in [how to install Let's Encrypt SSL on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

If you put Cloudflare or a load balancer in front of Nginx, configure Laravel's trusted proxies so `request()->secure()` and generated URLs see HTTPS.

## Deploying Laravel Updates

A simple in-place deploy script:

```bash
#!/usr/bin/env bash
set -e
cd /var/www/laravel
php artisan down --retry=15
git pull
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan optimize
php artisan queue:restart
sudo systemctl reload php8.3-fpm
php artisan up
```

Reloading PHP-FPM clears OPcache so the new code is served. `artisan down` shows a 503 maintenance page for the few seconds the deploy takes. For true zero-downtime deploys, use release directories with a `current` symlink (which is why the server block uses `$realpath_root`), or a tool like Deployer or Envoyer.

## Troubleshooting Laravel on Nginx

| Symptom | Cause | Fix |
| --- | --- | --- |
| Home page works, every other route returns 404 | `try_files` missing or wrong in `location /` | Use `try_files $uri $uri/ /index.php?$query_string;` |
| Nginx shows a directory listing or 403 | `root` points at the project root, not `public/` | Set `root /var/www/laravel/public;` |
| Blank 500 error | Exception with `APP_DEBUG=false` | Read `storage/logs/laravel-*.log` |
| `The stream or file ".../laravel.log" could not be opened` | `storage/` not writable by `www-data` | Step 4 permissions; check `ls -la storage` |
| `No application encryption key has been specified` | `APP_KEY` empty | `php artisan key:generate`, then `config:cache` |
| `.env` changes ignored | Config is cached | `php artisan config:cache` again |
| 502 Bad Gateway | Wrong PHP-FPM socket or PHP-FPM is down | `ls /run/php/`, `systemctl status php8.3-fpm` |
| `419 Page Expired` on forms | Session/cookie issue, often `SESSION_DOMAIN` or HTTPS mismatch | Check `APP_URL`, `SESSION_DOMAIN`, `SESSION_SECURE_COOKIE` |
| Jobs never run | Worker not running, or old code in memory | `systemctl status laravel-worker`, `php artisan queue:restart` |
| `413 Request Entity Too Large` on uploads | Nginx `client_max_body_size` (1 MB default) | Raise it, plus `upload_max_filesize` and `post_max_size` in `php.ini` |

More on gateway errors in [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Your Laravel App

Most Laravel outages don't take Nginx down. The server keeps answering while every request returns a 500 because the database is full, a cached config points at the wrong host, or a deploy left the app in maintenance mode. You need a check that loads the site the way a visitor does.

[GreenCrew](https://greencrew.space) monitors your site over HTTP from outside your server and alerts you on Discord, Slack, or Telegram when it fails, and gives you a public status page to share with users. Point a check at `https://example.com/up` (or a route that also queries the database) so a broken app counts as down, not just a dead server. Setup steps are in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Why does Nginx need to point at Laravel's public folder?

`public/` contains only `index.php` and your assets. Pointing `root` at the project root would make `.env`, `vendor/`, `storage/`, and your source code downloadable or executable over HTTP. With `root` on `public/`, those files are outside the web root.

### Should I use Laravel Forge or set up Nginx manually?

Forge and similar tools provision this exact stack (Nginx, PHP-FPM, queue workers, scheduler, Certbot) for you. Setting it up manually once is worth it so you know what's running and can debug it, and it costs nothing beyond the server.

### Do I need Supervisor for Laravel queues?

No. Laravel's docs use Supervisor, but systemd is already on every Ubuntu server and does the same job: it starts the worker at boot and restarts it when it exits. Use whichever you're comfortable maintaining.

### How do I fix "Permission denied" errors in storage after a deploy?

Files created by your deploy user (for example, when you ran `artisan` as yourself) can end up unwritable by `www-data`. Re-run `sudo chown -R $USER:www-data storage bootstrap/cache && sudo chmod -R 775 storage bootstrap/cache`, and run artisan commands that write cache files as the same user consistently.

## Conclusion

Deploying Laravel with Nginx on Ubuntu comes down to a few rules: point `root` at `public/`, route everything through `index.php` with `try_files`, keep `APP_DEBUG=false`, make only `storage/` and `bootstrap/cache/` writable, and cache your config on each deploy. Add a systemd queue worker, the scheduler cron entry, and HTTPS, and you have a production setup. Then add an external check with [GreenCrew](https://greencrew.space) on your `/up` route so you hear about a 500 error before your users do.
