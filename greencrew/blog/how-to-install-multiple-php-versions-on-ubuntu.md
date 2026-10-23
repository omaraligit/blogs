# How to Install PHP on Ubuntu and Run Multiple Versions (ondrej/php PPA, update-alternatives, and PHP-FPM per Site)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, PHP, PHP-FPM, Ubuntu, Linux, DevOps

Description: To install a specific PHP version on Ubuntu, add the ondrej/php PPA with sudo add-apt-repository ppa:ondrej/php, then install versioned packages like php8.4-fpm and php8.2-fpm side by side. Switch the CLI version with sudo update-alternatives --config php, and point each Nginx site at its own socket, such as fastcgi_pass unix:/run/php/php8.2-fpm.sock. This guide covers extensions, php.ini locations, Composer, upgrading a site, and common errors.

Date: 2026-10-23

**Short answer:** Ubuntu's own repositories ship one PHP version per release (PHP 8.3 on Ubuntu 24.04, PHP 8.1 on 22.04). To get a different version, or several at once, add Ondřej Surý's PPA with `sudo add-apt-repository ppa:ondrej/php`, run `sudo apt update`, and install versioned packages such as `php8.4-fpm php8.4-cli php8.4-mysql`. Every version installs into its own `/etc/php/<version>/` directory and runs its own PHP-FPM service and socket, so versions don't conflict. Pick the version the `php` command uses with `sudo update-alternatives --config php`. For each Nginx site, set `fastcgi_pass` to the socket of the version that site needs.

## Why Install a Specific PHP Version on Ubuntu?

Most people end up here for one of three reasons:

- **Your app needs a newer PHP than Ubuntu ships.** For example, a new Laravel or Symfony release requires a version that your LTS release doesn't have yet.
- **A legacy app needs an older PHP.** You need to keep it running while you migrate it, without holding back every other site on the server.
- **You're setting up a local development environment** and want to match production exactly.

Ubuntu's default `php` package always points to the release's single bundled version. It doesn't give you a choice. The `ondrej/php` PPA is the standard way around this. It's maintained by Ondřej Surý, who also maintains PHP packaging for Debian, and it provides many PHP versions built for current Ubuntu LTS releases. You can install them side by side.

One warning before you start: **don't install end-of-life PHP for new work.** Older tutorials still tell you to install PHP 7.4, but PHP 7.4 reached end of life in November 2022 and gets no security fixes. Check [php.net/supported-versions](https://www.php.net/supported-versions.php) for the current list and use a supported branch wherever you can.

## Prerequisites

- Ubuntu 24.04 or 22.04 with a sudo user
- Nginx installed if you're serving sites ([how to install Nginx on Ubuntu](/blog/how-to-install-nginx-on-ubuntu))
- The PHP version your app requires (check its `composer.json` under `require.php`)

This guide uses **PHP 8.4** as the "new" version and **PHP 8.2** as the "legacy" version. Swap in whichever versions you need. The commands are the same.

## Step 1: Check Which PHP Version Is Installed

```bash
php -v                     # CLI version (if any PHP is installed)
ls /etc/php/               # one directory per installed version
ls /run/php/               # FPM sockets for running versions
apt policy php             # which version Ubuntu's default "php" package points to
```

If `/etc/php/` already contains a version, it stays installed. Adding the PPA doesn't remove it. It only makes more versions available.

## Step 2: Add the ondrej/php PPA

```bash
sudo apt update
sudo apt install -y software-properties-common ca-certificates
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```

To see the versions you can now install:

```bash
apt-cache search --names-only '^php[0-9.]+-fpm$'
```

On **Debian**, use Ondřej's repository at [packages.sury.org](https://packages.sury.org/php/) instead of the PPA. The package names are the same.

## Step 3: Install PHP-FPM, the CLI, and Extensions

Install the FPM and CLI packages explicitly:

```bash
sudo apt install -y php8.4-fpm php8.4-cli php8.4-common \
  php8.4-mysql php8.4-mbstring php8.4-xml php8.4-curl \
  php8.4-zip php8.4-bcmath php8.4-intl php8.4-gd
```

This matters. If you install the bare `php8.4` metapackage, apt can pull in **Apache** and `libapache2-mod-php8.4` to satisfy its dependencies. On an Nginx server you want `php8.4-fpm`, not the Apache module.

To add the legacy version alongside it, run the same command with the other version number. Brace expansion keeps it short:

```bash
sudo apt install -y php8.2-{fpm,cli,common,mysql,mbstring,xml,curl,zip,bcmath,intl,gd}
```

### Common PHP Extensions and What Uses Them

| Package | Provides | Typically needed by |
| --- | --- | --- |
| `phpX.Y-fpm` | FastCGI Process Manager | Any site behind Nginx |
| `phpX.Y-cli` | The `phpX.Y` command | Composer, Artisan, cron jobs |
| `phpX.Y-mysql` | `pdo_mysql`, `mysqli` | MySQL/MariaDB apps, WordPress |
| `phpX.Y-pgsql` | `pdo_pgsql` | PostgreSQL apps |
| `phpX.Y-mbstring` | Multibyte strings | Laravel, Symfony, WordPress |
| `phpX.Y-xml` | DOM, SimpleXML, XMLWriter | Most frameworks and Composer packages |
| `phpX.Y-curl` | HTTP client | API calls, Guzzle |
| `phpX.Y-zip` | Zip archives | Composer, file exports |
| `phpX.Y-intl` | Locale and number formatting | Laravel, Symfony, e-commerce |
| `phpX.Y-gd` or `phpX.Y-imagick` | Image processing | WordPress media, thumbnails |
| `phpX.Y-redis` | Redis client | Caches, queues, sessions |

Extensions are installed **per version**. Installing `php8.4-redis` does nothing for PHP 8.2. Check what each version has loaded:

```bash
php8.4 -m
php8.2 -m | grep -i redis
```

## Step 4: Confirm Both PHP-FPM Services Are Running

Each version gets its own systemd service and socket:

```bash
systemctl status php8.4-fpm php8.2-fpm --no-pager
ls /run/php/
# php8.2-fpm.sock  php8.4-fpm.sock  php-fpm.sock
```

| Item | PHP 8.4 | PHP 8.2 |
| --- | --- | --- |
| CLI binary | `/usr/bin/php8.4` | `/usr/bin/php8.2` |
| FPM service | `php8.4-fpm` | `php8.2-fpm` |
| FPM socket | `/run/php/php8.4-fpm.sock` | `/run/php/php8.2-fpm.sock` |
| CLI php.ini | `/etc/php/8.4/cli/php.ini` | `/etc/php/8.2/cli/php.ini` |
| FPM php.ini | `/etc/php/8.4/fpm/php.ini` | `/etc/php/8.2/fpm/php.ini` |
| FPM pool config | `/etc/php/8.4/fpm/pool.d/www.conf` | `/etc/php/8.2/fpm/pool.d/www.conf` |
| Extra .ini files | `/etc/php/8.4/mods-available/` | `/etc/php/8.2/mods-available/` |

Note that the **CLI and FPM have separate php.ini files**. If you raise `memory_limit` in the CLI file, your website won't see the change, and changing the FPM file won't affect Artisan or cron. To find the files a binary actually loads, run `php8.4 --ini`.

The version-less `php-fpm.sock` is a symlink managed by the alternatives system. It points to one version's socket. It's handy, but it's safer to use the explicit versioned socket in Nginx so that installing a new version never silently moves a site.

## Step 5: Switch the Default PHP CLI Version with update-alternatives

The `php` command is a symlink chain managed by `update-alternatives`. Run the interactive picker:

```bash
sudo update-alternatives --config php
```

Or set the version directly, which is handy in scripts:

```bash
sudo update-alternatives --set php /usr/bin/php8.4
php -v
```

If you build PECL extensions, switch the related tools too so they match:

```bash
sudo update-alternatives --set phar /usr/bin/phar8.4
sudo update-alternatives --set phar.phar /usr/bin/phar.phar8.4
sudo update-alternatives --set phpize /usr/bin/phpize8.4
sudo update-alternatives --set php-config /usr/bin/php-config8.4
```

(The `phpize` and `php-config` alternatives only exist if you've installed `php8.4-dev`.)

You don't have to change the default at all. You can call a version directly by name, which is often clearer in cron jobs and deploy scripts:

```bash
php8.2 /var/www/legacy/artisan schedule:run
php8.4 /usr/local/bin/composer install --no-dev
```

`update-alternatives` only changes the command line. **It doesn't affect which PHP your websites use.** Nginx decides that in the next step.

## Step 6: Point Each Nginx Site at Its Own PHP Version

The version a site runs depends on the socket in its `fastcgi_pass` line. Here are two sites on the same server running different versions:

```nginx
# /etc/nginx/sites-available/new-app.conf
server {
    listen 80;
    server_name new.example.com;
    root /var/www/new-app/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.4-fpm.sock;
    }
}
```

```nginx
# /etc/nginx/sites-available/legacy-app.conf
server {
    listen 80;
    server_name legacy.example.com;
    root /var/www/legacy-app/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    }
}
```

Enable the sites, test, and reload:

```bash
sudo ln -s /etc/nginx/sites-available/new-app.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/legacy-app.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

To confirm which version a site really runs, temporarily drop a file into its web root, load it in a browser, and then delete it:

```bash
echo '<?php echo PHP_VERSION;' | sudo tee /var/www/legacy-app/public/version.php
curl -s http://legacy.example.com/version.php   # 8.2.x
sudo rm /var/www/legacy-app/public/version.php
```

The full FastCGI setup, socket permissions, and `pm.max_children` sizing are covered in [how to configure PHP-FPM with Nginx](/blog/how-to-configure-php-fpm-with-nginx).

### Optional: A Separate PHP-FPM Pool per Site

By default, every site using PHP 8.4 shares the `www` pool and runs as `www-data`. If you want one site's PHP code isolated from another's, create a dedicated pool with its own user and socket:

```ini
; /etc/php/8.4/fpm/pool.d/new-app.conf
[new-app]
user = newapp
group = newapp
listen = /run/php/php8.4-fpm-new-app.sock
listen.owner = www-data
listen.group = www-data
pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin newapp
sudo php-fpm8.4 -t && sudo systemctl reload php8.4-fpm
```

Then use `fastcgi_pass unix:/run/php/php8.4-fpm-new-app.sock;` in that site's server block, and give `newapp` ownership of the directories the app writes to.

## Step 7: Install Composer for a Local Development Environment

Composer runs on whichever PHP you call it with. Install it once, using the official installer from [getcomposer.org/download](https://getcomposer.org/download/), which also shows the current hash to verify against:

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
# compare the file's hash with the one published on getcomposer.org/download
php composer-setup.php --install-dir=/usr/local/bin --filename=composer
php -r "unlink('composer-setup.php');"
composer --version
```

To install a project's dependencies with a specific PHP version, call Composer through that binary: `php8.2 /usr/local/bin/composer install`. To stop Composer from resolving packages that only work on your laptop's newer PHP, pin the production version in the project's `composer.json`:

```bash
composer config platform.php 8.2.0
```

For quick local testing without Nginx, PHP's built-in server is enough:

```bash
cd ~/projects/my-app/public
php8.4 -S 127.0.0.1:8000
```

The built-in server is for development only. In production, put PHP-FPM behind Nginx.

## Step 8: Upgrade a Site to a New PHP Version (and Remove the Old One)

Moving a site from PHP 8.2 to 8.4 is just a switch of sockets. You can roll it back by changing one line:

1. Install the same extensions for the new version (compare `php8.2 -m` with `php8.4 -m`).
2. Copy over **only the settings you changed** in `/etc/php/8.2/fpm/php.ini` and the pool file. Don't copy the whole files, because defaults differ between versions.
3. Run your test suite and `composer install` with `php8.4`.
4. Change `fastcgi_pass` to `php8.4-fpm.sock`, then run `sudo nginx -t && sudo systemctl reload nginx`.
5. Update cron jobs, queue workers, and deploy scripts that call `php8.2` by name.
6. Watch the error log, `/var/log/nginx/error.log`, and your app logs for deprecation warnings and fatal errors.

When nothing uses the old version anymore, stop it and remove it:

```bash
sudo systemctl disable --now php8.2-fpm
sudo apt purge 'php8.2*'
sudo apt autoremove
```

Each idle PHP-FPM service keeps its own worker processes in memory, so don't leave versions running that no site uses.

## Troubleshooting Multiple PHP Versions on Ubuntu

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Unable to locate package php8.4-fpm` | PPA not added, or `apt update` not run after adding it | `sudo add-apt-repository ppa:ondrej/php && sudo apt update` |
| Apache installed itself and took port 80 | Installed the `php8.4` metapackage instead of `php8.4-fpm` | `sudo apt purge apache2 libapache2-mod-php*`, then install `php8.4-fpm` |
| `php -v` shows the wrong version | CLI alternative still points to the old binary | `sudo update-alternatives --config php` |
| Site still runs the old version after switching the CLI | Nginx uses `fastcgi_pass`, not the CLI | Change the socket in the server block, then reload Nginx |
| 502 Bad Gateway, `connect() to unix:/run/php/php8.4-fpm.sock failed (2: No such file or directory)` | That FPM service isn't installed or running | `sudo systemctl start php8.4-fpm`; check `ls /run/php/` |
| `Call to undefined function mysqli_connect()` or `could not find driver` | Extension installed for a different version | Install `phpX.Y-mysql` for the version the site uses, then restart its FPM |
| php.ini change has no effect | Edited the CLI file instead of the FPM file (or the reverse) | Check the file against `php8.4 --ini`, edit the right one, `sudo systemctl restart php8.4-fpm` |
| Composer: `Your requirements could not be resolved` (PHP version) | Composer ran with a different PHP than the project targets | Run `php8.2 /usr/local/bin/composer install` or set `config.platform.php` |

For more on gateway errors, see [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Every Site After a PHP Upgrade

A PHP version switch is one of the easiest ways to break a site without noticing. The homepage loads fine, but a checkout page or an admin route hits a removed function and returns a 500. Testing each site by hand right after the switch helps. It doesn't help when something breaks at 3 a.m. after an unattended `apt upgrade`.

[GreenCrew](https://greencrew.space) checks your sites over HTTP from outside your server and alerts you on Discord, Slack, or Telegram the moment one goes down. It also gives you a public status page to share with users. Add a check for each site, not just one per server, and point the checks at a page that actually runs PHP and touches the database, so a broken FPM pool counts as downtime. The setup is in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Can I run two PHP versions on the same server?

Yes. Packages from the `ondrej/php` PPA install into separate `/etc/php/<version>/` directories, with separate binaries (`php8.2`, `php8.4`), FPM services, and sockets. Nginx picks the version for each site through `fastcgi_pass`, so one server can host sites on different versions at the same time.

### Is the ondrej/php PPA safe to use in production?

It's the most widely used source of alternative PHP builds for Ubuntu, and its maintainer also packages PHP for Debian. It's still a third-party repository, though, so updates come from it rather than from Canonical. Pin the versioned package names you rely on, and test upgrades on staging first.

### How do I install PHP 7.4 on Ubuntu?

The PPA has historically carried older branches like `php7.4-fpm`, and the install commands are the same as above. However, PHP 7.4 has been end of life since November 2022 and gets no security patches. Use it only to keep a legacy app running while you migrate it to a supported version, and plan that migration now.

### Does update-alternatives change the PHP version Nginx uses?

No. `update-alternatives --config php` only changes which binary the `php` command runs: your shell, Composer, and cron jobs that call `php`. Nginx sends requests to whichever PHP-FPM socket is in each server block's `fastcgi_pass` line.

## Conclusion

To install the PHP version you need on Ubuntu, add `ppa:ondrej/php`, install the versioned `-fpm` and `-cli` packages plus extensions for each version, and keep them in their own `/etc/php/<version>/` directories. Use `update-alternatives` to choose the default CLI and `fastcgi_pass` to choose each site's version, so upgrades become a one-line, reversible change. If you're setting up the full stack, continue with [how to install a LEMP stack on Ubuntu](/blog/how-to-install-lemp-stack-on-ubuntu) or [how to deploy Laravel with Nginx](/blog/how-to-deploy-laravel-with-nginx-on-ubuntu). Then add [GreenCrew](https://greencrew.space) checks so a PHP upgrade that breaks a site alerts you right away.
