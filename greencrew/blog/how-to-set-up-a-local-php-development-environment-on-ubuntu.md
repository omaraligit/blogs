# How to Set Up a Local PHP Development Environment on Ubuntu (php -S Built-In Server, Xdebug, and Composer)

Author: [greencrew](https://greencrew.space/blog)

Tags: PHP, Ubuntu, Xdebug, Composer, Nginx, DevOps

Description: To set up a local PHP development environment on Ubuntu, run sudo apt install php-cli php-mbstring php-xml php-curl php-zip php-sqlite3 unzip, enable development error settings, and start a project with php -S localhost:8000 -t public. This guide covers the built-in web server, Xdebug step debugging in VS Code, Composer, a local database, and when to switch to Nginx with PHP-FPM.

Date: 2026-10-28

**Short answer:** Install PHP and the common extensions with `sudo apt install php-cli php-mbstring php-xml php-curl php-zip php-sqlite3 unzip`. Then add a small `99-dev.ini` override that turns on `display_errors` and `error_reporting = E_ALL`. Start any project with PHP's built-in server, `php -S localhost:8000 -t public`, and open `http://localhost:8000`. Add `php-xdebug` with `xdebug.mode=debug` for breakpoints in VS Code, and Composer for dependencies. You don't need Apache, Nginx, or Docker until you want your laptop to behave like production.

## What Is a Local PHP Development Environment?

A local PHP development environment is everything you need to write, run, and debug PHP code on your own machine before it reaches a server. On Ubuntu it usually has five parts:

| Component | What it does | Package / tool |
| --- | --- | --- |
| PHP CLI | Runs scripts, Composer, and the built-in server | `php-cli` |
| Extensions | String, XML, HTTP, ZIP, and database support most frameworks require | `php-mbstring`, `php-xml`, `php-curl`, `php-zip`, `php-sqlite3`, `php-mysql` |
| Web server | Serves your app at `http://localhost` | `php -S` (built in), or Nginx + PHP-FPM |
| Debugger | Breakpoints, step-through, stack traces | `php-xdebug` + an IDE |
| Dependency manager | Installs libraries and frameworks | Composer |

This is the opposite of a production setup. Locally you *want* loud errors, a debugger, and fast restarts. On a server you want the reverse. Keeping the two configs separate is most of the job.

### Which PHP Version Will You Get?

Ubuntu's own repositories ship one PHP branch per release:

| Ubuntu release | Default PHP from `apt install php` |
| --- | --- |
| 24.04 LTS (Noble) | PHP 8.3 |
| 22.04 LTS (Jammy) | PHP 8.1 |

Older tutorials (including many written for Ubuntu 20.04) install **PHP 7.4**. PHP 7.4 reached end of life in November 2022 and gets no security fixes (see the [PHP supported versions page](https://www.php.net/supported-versions.php)). Only use it locally if you have to maintain a legacy app. In that case, install it next to a current version from the ondrej/php PPA, as shown in [how to install multiple PHP versions on Ubuntu](/blog/how-to-install-multiple-php-versions-on-ubuntu).

## Prerequisites

- Ubuntu 24.04 or 22.04 (desktop, server, or WSL2 all work)
- A user with `sudo`
- An editor. The debugging steps use VS Code, but PhpStorm works the same way.

## Step 1: Install PHP and Common Extensions

Update the package index and install PHP CLI with the extensions that Laravel, Symfony, WordPress, and most Composer packages expect:

```bash
sudo apt update
sudo apt install -y php-cli php-common php-mbstring php-xml php-curl \
  php-zip php-intl php-bcmath php-sqlite3 php-mysql unzip git
```

`unzip` and `git` aren't PHP packages, but Composer uses them to download dependencies quickly.

Check the result:

```bash
php -v
php -m | sort
```

`php -v` prints the version. `php -m` lists every loaded module, so you can confirm that `mbstring`, `xml`, `curl`, and `pdo_sqlite` are there.

### Where PHP's Config Files Live

On Ubuntu each SAPI (the way PHP is run) has its own `php.ini`:

| Path | Used by |
| --- | --- |
| `/etc/php/8.3/cli/php.ini` | `php` on the command line, `php -S`, Composer, Artisan |
| `/etc/php/8.3/fpm/php.ini` | PHP-FPM (only if you install `php-fpm`) |
| `/etc/php/8.3/cli/conf.d/` | Extra `.ini` files loaded after `php.ini`, in alphabetical order |
| `/etc/php/8.3/mods-available/` | One `.ini` per extension, enabled with `phpenmod` |

Replace `8.3` with your version. `php --ini` prints exactly which files are loaded.

## Step 2: Turn On Development Error Reporting

Ubuntu's packaged `php.ini` is based on PHP's **production** template, so errors can be hidden or only logged. Rather than editing `php.ini`, drop an override file into `conf.d`. It's easy to find later and easy to delete:

```bash
sudo tee /etc/php/8.3/cli/conf.d/99-dev.ini > /dev/null <<'EOF'
; Local development settings. Never copy this file to a server.
display_errors = On
display_startup_errors = On
error_reporting = E_ALL
log_errors = On
memory_limit = 512M
upload_max_filesize = 64M
post_max_size = 64M
EOF
```

Confirm the values took effect:

```bash
php -i | grep -E 'display_errors|error_reporting|memory_limit'
```

| Directive | Dev value | Production value | Why |
| --- | --- | --- | --- |
| `display_errors` | `On` | `Off` | See notices and warnings in the browser right away |
| `error_reporting` | `E_ALL` | `E_ALL` (logged, not shown) | Catch deprecations before an upgrade breaks you |
| `display_startup_errors` | `On` | `Off` | Show extension load failures |
| `memory_limit` | `512M` | Sized per app | Composer and test suites can be memory hungry |

The full list of directives is in the [php.ini directives reference](https://www.php.net/manual/en/ini.list.php).

## Step 3: Run Your App with the PHP Built-In Web Server

PHP has shipped a development web server since 5.4. It needs no config files and no root:

```bash
mkdir -p ~/code/hello/public
cd ~/code/hello
cat > public/index.php <<'EOF'
<?php
echo "Hello from PHP " . PHP_VERSION;
phpinfo(INFO_GENERAL);
EOF

php -S localhost:8000 -t public
```

Open `http://localhost:8000`. Each request is logged in the terminal, and `Ctrl+C` stops the server.

### Useful `php -S` Options

| Command | What it does |
| --- | --- |
| `php -S localhost:8000` | Serve the current directory |
| `php -S localhost:8000 -t public` | Use `public/` as the document root (Laravel, Symfony, Slim) |
| `php -S 0.0.0.0:8000 -t public` | Reachable from other devices on your LAN, such as a phone |
| `php -S localhost:8000 router.php` | Send every request through a router script |
| `PHP_CLI_SERVER_WORKERS=4 php -S ...` | Handle parallel requests (PHP 7.4+, not on Windows) |

### Add a Router Script for Pretty URLs

Frameworks need every non-file URL routed to `index.php`. With Nginx you'd use `try_files`. With the built-in server you use a router script, and returning `false` tells PHP to serve the requested file as-is:

```php
<?php
// router.php
$path = __DIR__ . '/public' . parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);

if ($_SERVER['REQUEST_URI'] !== '/' && is_file($path)) {
    return false; // serve CSS, JS, images directly
}

require __DIR__ . '/public/index.php';
```

```bash
php -S localhost:8000 -t public router.php
```

Framework CLIs wrap the same server for you: `php artisan serve` in Laravel and `symfony server:start` in Symfony (the Symfony CLI adds TLS and a local proxy).

The [PHP manual](https://www.php.net/manual/en/features.commandline.webserver.php) is clear that the built-in server is for development only. It's single-threaded by default and not hardened, so never expose it to the internet.

## Step 4: Install Composer

Composer installs your framework and libraries and generates the autoloader. The quick option is `sudo apt install composer`, but Ubuntu's package often lags behind the current release. The verified global install from [getcomposer.org](https://getcomposer.org/download/) keeps you on the latest version and lets you run `composer self-update`.

We cover that install, including signature verification, `composer.json`, and `--no-dev` deploys, in [how to install and use Composer on Ubuntu](/blog/how-to-install-and-use-composer-on-ubuntu). Once it's installed:

```bash
composer --version
composer create-project laravel/laravel ~/code/myapp
cd ~/code/myapp && php artisan serve
```

## Step 5: Set Up Xdebug Step Debugging

`var_dump()` only gets you so far. Xdebug adds breakpoints, variable inspection, and readable stack traces.

```bash
sudo apt install -y php-xdebug
php -v   # should now show "with Xdebug v3.x"
```

Add the debugger settings to the Xdebug module file:

```bash
sudo tee -a /etc/php/8.3/mods-available/xdebug.ini > /dev/null <<'EOF'
xdebug.mode = debug,develop
xdebug.start_with_request = trigger
xdebug.client_host = 127.0.0.1
xdebug.client_port = 9003
EOF
```

| Setting | Value | Meaning |
| --- | --- | --- |
| `xdebug.mode` | `debug,develop` | Step debugging plus improved `var_dump()` and error output |
| `xdebug.start_with_request` | `trigger` | Only debug when a trigger is present, which keeps every other request fast |
| `xdebug.start_with_request` | `yes` | Debug every request (simpler, slower) |
| `xdebug.client_host` | `127.0.0.1` | Where your IDE is listening |
| `xdebug.client_port` | `9003` | Xdebug 3's default port (Xdebug 2 used 9000) |

Restart `php -S` after changing any `.ini` file. It reads config only at startup.

### Connect VS Code

1. Install the **PHP Debug** extension (publisher: Xdebug).
2. Create `.vscode/launch.json` in your project:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Listen for Xdebug",
      "type": "php",
      "request": "launch",
      "port": 9003
    }
  ]
}
```

3. Press **F5** to start listening, then set a breakpoint in `public/index.php`.
4. Trigger a debug session:

```bash
# Browser: append the trigger to the URL
http://localhost:8000/?XDEBUG_TRIGGER=1

# CLI scripts and tests
XDEBUG_TRIGGER=1 php artisan migrate
```

If the breakpoint isn't hit, `php -i | grep xdebug.mode` should show `debug`, and `xdebug_info()` in a PHP page shows the connection log. The [Xdebug step debugging docs](https://xdebug.org/docs/step_debug) cover every setting.

Xdebug slows PHP down noticeably even when idle. Leave it out of production entirely, and on a laptop disable it with `sudo phpdismod xdebug` when you're benchmarking.

## Step 6: Add a Local Database

Pick whatever your production app uses. If you don't care yet, start with SQLite.

| Option | Install | Best for |
| --- | --- | --- |
| SQLite | Already installed (`php-sqlite3`) | Prototypes, tests, single-user apps. The database is just a file. |
| MySQL | `sudo apt install mysql-server` | Matching a LEMP production server |
| MariaDB | `sudo apt install mariadb-server` | Drop-in MySQL alternative, default on many hosts |
| PostgreSQL | `sudo apt install postgresql php-pgsql` | Django/Postgres shops, advanced SQL |

A quick SQLite connection test:

```php
<?php
$pdo = new PDO('sqlite:' . __DIR__ . '/../database.sqlite');
$pdo->exec('CREATE TABLE IF NOT EXISTS visits (at TEXT)');
$pdo->exec("INSERT INTO visits VALUES (datetime('now'))");
echo $pdo->query('SELECT COUNT(*) FROM visits')->fetchColumn() . " visits";
```

For MySQL users, privileges, and `mysql_secure_installation`, follow Step 2 of [how to install a LEMP stack on Ubuntu](/blog/how-to-install-lemp-stack-on-ubuntu).

## Step 7 (Optional): Mirror Production with Nginx and PHP-FPM

The built-in server is fine for most work. It doesn't reproduce Nginx rewrites, upload limits, FastCGI timeouts, or multiple sites at once. When a bug only shows up on the server, run the same stack locally:

```bash
sudo apt install -y nginx php-fpm
echo "127.0.0.1 myapp.test" | sudo tee -a /etc/hosts
```

`.test` is a reserved top-level domain ([RFC 2606](https://www.rfc-editor.org/rfc/rfc2606)), so it will never collide with a real site. Create `/etc/nginx/sites-available/myapp.test`:

```nginx
server {
    listen 80;
    server_name myapp.test;
    root /home/youruser/code/myapp/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myapp.test /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

The `www-data` user must be able to traverse your home directory (`chmod o+x /home/youruser /home/youruser/code`). Remember that PHP-FPM reads `/etc/php/8.3/fpm/`, not the `cli` folder, so copy `99-dev.ini` into `/etc/php/8.3/fpm/conf.d/` too and run `sudo systemctl restart php8.3-fpm`. Socket paths and pool settings are explained in [how to configure PHP-FPM with Nginx](/blog/how-to-configure-php-fpm-with-nginx).

### Built-In Server vs Nginx + PHP-FPM vs Docker

| | `php -S` | Nginx + PHP-FPM | Docker Compose |
| --- | --- | --- | --- |
| Setup time | Seconds | ~10 minutes | ~20 minutes |
| Matches production | Loosely | Closely (if prod is LEMP) | Exactly, if you use the same images |
| Multiple PHP versions | One per terminal | One FPM pool per version | One image per project |
| Needs root | No | Yes | Docker group |
| Good for | Daily coding, quick tests | Debugging rewrites and limits | Teams, onboarding, CI parity |

If your team wants identical environments, see [how to set up Laravel, Nginx, and MySQL with Docker Compose](/blog/how-to-set-up-laravel-nginx-mysql-with-docker-compose).

## Troubleshooting Your Local PHP Setup

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Command 'php' not found` | PHP CLI not installed | `sudo apt install php-cli` |
| `Class "DOMDocument" not found` / `ext-xml` missing | XML extension missing | `sudo apt install php-xml` |
| Composer: `requires ext-mbstring` | Extension missing for the PHP version Composer uses | `sudo apt install php8.3-mbstring`, then `php -m` |
| Blank white page, no error | `display_errors` off | Add `99-dev.ini` (Step 2) and restart `php -S` |
| `Failed to listen on localhost:8000 (Address already in use)` | Another server on that port | Use another port, or `sudo ss -ltnp \| grep 8000` to find it |
| Pretty URLs return 404 | No router script | Use `-t public router.php` or `php artisan serve` |
| Xdebug breakpoints never hit | Wrong port, no trigger, or IDE not listening | Port `9003`, add `XDEBUG_TRIGGER=1`, press F5 first |
| Everything got slow | Xdebug loaded | `sudo phpdismod xdebug` when not debugging |
| `.ini` change ignored | Edited the wrong SAPI's file | Check `php --ini`. CLI and FPM have separate folders. |
| Nginx `403 Forbidden` on `~/code` | `www-data` can't traverse home | `chmod o+x` each parent directory |

## From Local to Production: Monitor What You Ship

A local environment tells you the code works on your laptop. It tells you nothing about whether the deployed site is up at 3 a.m. Once the app is live (for example on a [LEMP server](/blog/how-to-install-lemp-stack-on-ubuntu)), add a lightweight `/health` route that runs PHP and touches the database. Then point an external check at it.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks against your URLs and alerts you on Discord, Slack, or Telegram when they stop responding correctly. It also gives you a public status page you can share with users. Setup takes a few minutes. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should I still install PHP 7.4 on Ubuntu?

Only to maintain a legacy app that can't run on PHP 8 yet. PHP 7.4 is end of life and unpatched. Install it from the ondrej/php PPA next to a supported version, and plan the upgrade. Our [multiple PHP versions guide](/blog/how-to-install-multiple-php-versions-on-ubuntu) shows how to switch between them with `update-alternatives`.

### Is the PHP built-in server good enough for development?

Yes, for most day-to-day work. It handles routing through a router script, serves static files, and restarts instantly. Switch to Nginx with PHP-FPM when you need to reproduce server-specific behavior like rewrites, upload limits, or timeouts.

### Do I need Apache or Nginx to learn PHP?

No. `php -S localhost:8000` is a complete web server for learning and prototyping. You'll need a real web server only when you deploy, and then Nginx with PHP-FPM is the common choice.

### Why does `php -v` show Xdebug but my breakpoints don't work?

Usually it's one of three things: `xdebug.mode` doesn't include `debug`, the request has no trigger while `start_with_request = trigger`, or your IDE is listening on 9000 instead of Xdebug 3's default port 9003. Call `xdebug_info()` in a page to see what Xdebug tried to do.

## Conclusion

A good local PHP environment on Ubuntu needs only a few pieces: `php-cli` with the usual extensions, a `99-dev.ini` that shows every error, `php -S` for instant serving, Xdebug for real debugging, and Composer for dependencies. Add Nginx with PHP-FPM or Docker only when you need to match production more closely. When the code ships, put the same care into the live site: monitor it with [GreenCrew](https://greencrew.space) so outages reach your Discord, Slack, or Telegram before they reach your users.
