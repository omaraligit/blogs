# How to Install and Use Composer on Ubuntu (Verified Global Install, composer.json, and Production Deploys)

Author: [greencrew](https://greencrew.space/blog)

Tags: PHP, Composer, Ubuntu, Linux, Nginx, DevOps

Description: To install Composer on Ubuntu, install php-cli and unzip, download the installer from getcomposer.org, verify its SHA-384 hash against composer.github.io/installer.sig, and run php composer-setup.php --install-dir=/usr/local/bin --filename=composer so the composer command works globally. This guide covers the verified install script, require vs install vs update, composer.lock, autoloading, version constraints, production deploys with --no-dev, and common Composer errors.

Date: 2026-10-23

**Short answer:** Run `sudo apt install php-cli unzip curl`. Then download the official installer, check its hash against the signature published at `https://composer.github.io/installer.sig`, and install it globally with `sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer`. Check it with `composer --version`. In a project, `composer require vendor/package` adds a dependency, `composer install` installs the exact versions from `composer.lock`, and `composer update` upgrades them. On production servers, deploy with `composer install --no-dev --optimize-autoloader --no-interaction`.

## What Is Composer?

[Composer](https://getcomposer.org/) is PHP's dependency manager. You list the libraries a project needs in `composer.json`. Composer resolves compatible versions, downloads them from [Packagist](https://packagist.org/) into `vendor/`, records the exact versions in `composer.lock`, and generates an autoloader so your code can use the classes without `require` statements.

Laravel, Symfony, WordPress plugin tooling, and almost every modern PHP library depend on it. If you deploy PHP apps behind Nginx, you'll run Composer on every deploy.

| File or folder | What it is | Commit to Git? |
| --- | --- | --- |
| `composer.json` | Your declared dependencies and version constraints | Yes |
| `composer.lock` | Exact resolved versions of every package | Yes, for applications |
| `vendor/` | Downloaded packages and the autoloader | No (add to `.gitignore`) |
| `vendor/autoload.php` | The file your code includes to load everything | Generated |

## Prerequisites

- Ubuntu 24.04 or 22.04 with a sudo user
- PHP CLI installed (PHP-FPM alone isn't enough, because Composer runs on the command line)

If you're setting up a full web server, install [Nginx, MySQL, and PHP-FPM as a LEMP stack](/blog/how-to-install-lemp-stack-on-ubuntu) first.

## Step 1: Install PHP CLI and the Tools Composer Needs

```bash
sudo apt update
sudo apt install php-cli php-zip php-curl php-mbstring php-xml unzip curl git
php -v
```

| Package | Why Composer needs it |
| --- | --- |
| `php-cli` | Composer is a PHP program |
| `unzip` and `php-zip` | Extract downloaded package archives (without them Composer falls back to slower `git` clones, or fails) |
| `php-curl` | Faster, more reliable downloads |
| `git` | Installing packages from source and from VCS repositories |
| `php-mbstring`, `php-xml` | Required by most frameworks and many packages, so install them now |

## Step 2: Download and Verify the Composer Installer

Composer's maintainers publish the installer's SHA-384 hash so you can confirm the download hasn't been tampered with. This script comes from the official guide to [installing Composer programmatically](https://getcomposer.org/doc/faqs/how-to-install-composer-programmatically.md). It fetches the current signature, so it never goes stale:

```bash
cd ~
EXPECTED_CHECKSUM="$(php -r 'copy("https://composer.github.io/installer.sig", "php://stdout");')"
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
ACTUAL_CHECKSUM="$(php -r "echo hash_file('sha384', 'composer-setup.php');")"

if [ "$EXPECTED_CHECKSUM" != "$ACTUAL_CHECKSUM" ]; then
    echo 'ERROR: Invalid installer checksum' >&2
    rm composer-setup.php
else
    echo 'Installer verified'
fi
```

If you see `Installer verified`, continue. If the checksum fails, download it again. Don't run an installer that fails verification.

## Step 3: Install Composer Globally

Install the `composer` binary into `/usr/local/bin` so every user can run it from any directory:

```bash
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
rm composer-setup.php
composer --version
```

To install only for your user, without sudo, use `--install-dir=$HOME/.local/bin` instead. On Ubuntu that directory is already on your `PATH` once it exists and you open a new shell.

### apt install composer vs the Official Installer

Ubuntu also packages Composer. Both methods work, with different trade-offs:

| | `sudo apt install composer` | Official installer (this guide) |
| --- | --- | --- |
| Version | Fixed to what the Ubuntu release ships, often older | Current stable release |
| Updates | Through `apt upgrade` | `composer self-update` |
| Dependencies | Pulls in Ubuntu's default PHP packages | Uses whichever PHP you already have |
| Best for | Quick setups that match the distro | Production servers and current frameworks |

Newer framework releases sometimes need a newer Composer than an older Ubuntu release ships, so the official installer is the safer default.

## Step 4: Create a Project and Add Dependencies

Start a `composer.json` interactively, or just require a package and let Composer create the file:

```bash
mkdir ~/composer-demo && cd ~/composer-demo
composer require monolog/monolog
```

Composer writes `composer.json`, resolves a compatible version, creates `composer.lock`, and downloads the package into `vendor/`:

```json
{
    "require": {
        "monolog/monolog": "^3.0"
    }
}
```

(The exact constraint depends on the current version when you run the command.)

Use the autoloader in your code:

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use Monolog\Logger;
use Monolog\Handler\StreamHandler;

$log = new Logger('demo');
$log->pushHandler(new StreamHandler(__DIR__ . '/app.log'));
$log->info('Composer autoloading works');
```

```bash
php index.php && cat app.log
```

### Autoload Your Own Classes with PSR-4

Map a namespace to a folder in `composer.json`:

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

```bash
composer dump-autoload
```

Now `App\Services\Mailer` loads from `src/Services/Mailer.php` automatically. Run `composer dump-autoload` again whenever you change the `autoload` section.

## Step 5: Understand install vs update vs require

Mixing these up is the most common Composer mistake. It's also the one most likely to break production.

| Command | What it does | When to use it |
| --- | --- | --- |
| `composer require vendor/pkg` | Adds a package to `composer.json`, resolves it, and updates the lock file | Adding a new dependency |
| `composer require --dev vendor/pkg` | Same, but under `require-dev` (tests, linters) | Dev-only tools like PHPUnit |
| `composer install` | Installs **exactly** what `composer.lock` lists | Fresh clones, CI, and **every production deploy** |
| `composer update` | Re-resolves constraints, upgrades packages, and rewrites the lock file | Deliberately upgrading, on your machine, then commit the lock file |
| `composer update vendor/pkg` | Upgrades just one package | Targeted upgrades and security fixes |
| `composer remove vendor/pkg` | Removes a package and updates both files | Dropping a dependency |

The rule: **run `update` in development and `install` everywhere else.** If you run `composer update` on a server, you can install versions nobody has tested.

### Version Constraints Cheat Sheet

| Constraint | Allows | Example matches for `1.2.3` base |
| --- | --- | --- |
| `^1.2.3` | Same major, `>=1.2.3 <2.0.0` (the default) | `1.2.3`, `1.9.0` |
| `~1.2.3` | Same minor, `>=1.2.3 <1.3.0` | `1.2.3`, `1.2.9` |
| `~1.2` | `>=1.2 <2.0` | `1.2.0`, `1.8.0` |
| `1.2.*` | Any `1.2` patch release | `1.2.0`, `1.2.7` |
| `1.2.3` | Exactly that version | `1.2.3` only |
| `>=1.2 <1.5` | An explicit range | `1.2.0`, `1.4.9` |

The full syntax is in Composer's [versions and constraints docs](https://getcomposer.org/doc/articles/versions.md).

## Step 6: Run Composer on a Production Server

On a server that runs your app behind Nginx and PHP-FPM, deploy with:

```bash
cd /var/www/myapp
composer install --no-dev --optimize-autoloader --no-interaction --prefer-dist
```

| Flag | Effect |
| --- | --- |
| `--no-dev` | Skips `require-dev` packages (test tools never reach production) |
| `--optimize-autoloader` (`-o`) | Builds a class map so classes load faster |
| `--no-interaction` | Never stops to ask a question, which is safe for scripts and CI |
| `--prefer-dist` | Downloads zip archives instead of cloning Git repos |

**Don't run Composer as root.** Packages can run scripts during install, and running them as root gives them full control of the server. Composer warns about this itself. Run it as the user that owns the code:

```bash
sudo -u deploy composer install --no-dev -o --no-interaction
```

Then make sure the web server user can read `vendor/` and write only to the directories your app needs. The Laravel version of this is in [how to deploy Laravel with Nginx on Ubuntu](/blog/how-to-deploy-laravel-with-nginx-on-ubuntu).

Before you deploy, also check for known vulnerabilities in your dependencies:

```bash
composer audit
```

### Pin the Production PHP Version

If your laptop runs a newer PHP than the server, `composer update` can lock packages that won't run in production. Tell Composer which PHP the server has:

```bash
composer config platform.php 8.3.0
```

This writes `config.platform.php` into `composer.json`, so dependency resolution targets 8.3 no matter which PHP runs the command. To see what the current machine is missing for the lock file, run:

```bash
composer check-platform-reqs
```

## Step 7: Keep Composer Itself Updated

```bash
sudo composer self-update            # update to the latest stable release
sudo composer self-update --rollback # go back if the new version causes trouble
composer diagnose                    # check config, network, and PHP settings
```

Use `sudo` only for `self-update` when Composer lives in `/usr/local/bin`, because the binary is root-owned. Project commands should still run as your normal user.

### Everyday Composer Commands

| Task | Command |
| --- | --- |
| List installed packages | `composer show` |
| Show outdated packages | `composer outdated --direct` |
| Explain why a package is installed | `composer why vendor/pkg` |
| Explain why a version can't be installed | `composer why-not vendor/pkg 2.0` |
| Create a project from a template | `composer create-project laravel/laravel myapp` |
| Validate composer.json | `composer validate` |
| Clear the download cache | `composer clear-cache` |
| Run a script from composer.json | `composer run-script <name>` |

## Troubleshooting Common Composer Errors on Ubuntu

| Symptom | Cause | Fix |
| --- | --- | --- |
| `composer: command not found` | Installed to a directory that isn't on `PATH`, or left as `composer.phar` | Re-run the installer with `--install-dir=/usr/local/bin --filename=composer` |
| `ERROR: Invalid installer checksum` | Corrupted or tampered download, or an old saved hash | Re-run Step 2, which fetches the current signature |
| `The zip extension and unzip/7z commands are both missing` | No way to extract archives | `sudo apt install unzip php-zip` |
| `requires ext-intl * -> it is missing from your system` | A package needs a PHP extension | Install `php-intl` for the PHP version the CLI uses (`php -v`) |
| `Your requirements could not be resolved to an installable set of packages` | Version conflict, or your PHP is too old or new for a package | Read the conflict lines, run `composer why-not`, and adjust constraints or `platform.php` |
| `Allowed memory size of ... bytes exhausted` | PHP CLI `memory_limit` too low for dependency resolution | `COMPOSER_MEMORY_LIMIT=-1 composer update` |
| `Do not run Composer as root/super user!` | Running with sudo or as root | Run as the user that owns the project |
| `Warning: The lock file is not up to date with the latest changes in composer.json` | `composer.json` was edited without updating the lock | Run `composer update --lock` (or `composer update vendor/pkg`) on dev and commit the lock file |
| GitHub API rate limit or `Could not authenticate against github.com` | Too many anonymous API calls, or a private repo | `composer config --global github-oauth.github.com <token>` |
| Site returns 500 right after a deploy | Autoloader out of date, or a missing extension on the FPM side | Re-run `composer install -o`, check `php-fpm` has the same extensions as the CLI, and read the app log |

The CLI and PHP-FPM load separate `php.ini` files and can even run different PHP versions. A package can install fine and still fail on the website. If that happens, compare `php -m` with the extensions shown in a `phpinfo()` page served through Nginx. More on that in [how to configure PHP-FPM with Nginx](/blog/how-to-configure-php-fpm-with-nginx).

## Catch Broken Deploys After composer install

Most Composer problems show up at the worst possible moment: right after a deploy. A missing extension, an autoloader that wasn't rebuilt, or an accidental `composer update` on the server can turn every page into a 500 while the server itself looks perfectly healthy.

[GreenCrew](https://greencrew.space) checks your site over HTTP from outside the server and alerts you on Discord, Slack, or Telegram as soon as it starts failing. It also gives you a public status page to share with users during an incident. Point a check at a route that boots your full application, not a static file, so a broken `vendor/` directory counts as downtime. The setup takes a few minutes: [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should I commit the vendor folder to Git?

No. Commit `composer.json` and `composer.lock`, add `vendor/` to `.gitignore`, and run `composer install` on each machine. The lock file guarantees every environment gets the same versions without storing thousands of third-party files in your repository.

### What is the difference between composer install and composer update?

`composer install` reads `composer.lock` and installs exactly those versions. If there's no lock file, it resolves and creates one. `composer update` ignores the lock file, finds the newest versions your constraints allow, installs them, and rewrites the lock file. Use `update` on purpose in development, and use `install` in CI and production.

### Is it safe to install Composer with sudo?

Using `sudo` to place the binary in `/usr/local/bin` is fine, because it only writes one file. Running project commands like `composer install` with `sudo` isn't, because package scripts would run as root. After the global install, run Composer as your regular user or as the deploy user.

### How do I use Composer with a specific PHP version?

Composer runs on whichever PHP binary starts it. If you have several versions installed, call it through the one you want, for example `php8.3 /usr/local/bin/composer install`. Set `config.platform.php` in `composer.json` so dependency resolution matches your production PHP version.

## Conclusion

To install Composer on Ubuntu, add `php-cli` and `unzip`, verify the official installer against `installer.sig`, and install the binary to `/usr/local/bin/composer`. Commit `composer.json` and `composer.lock`, run `composer update` only on purpose in development, and deploy with `composer install --no-dev --optimize-autoloader --no-interaction` as a non-root user. Then serve the app through [Nginx and PHP-FPM](/blog/how-to-configure-php-fpm-with-nginx) and add a [GreenCrew](https://greencrew.space) check so you hear about a broken deploy before your users report it.
