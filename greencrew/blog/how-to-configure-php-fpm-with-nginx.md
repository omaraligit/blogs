# How to Configure PHP-FPM with Nginx (fastcgi_pass, Unix Socket, and Pool Tuning)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, PHP-FPM, PHP, Linux, DevOps

Description: To run PHP with Nginx, install php-fpm, then add a location ~ \.php$ block that includes snippets/fastcgi-php.conf and sets fastcgi_pass unix:/run/php/php-fpm.sock. This guide covers the full server block, pool settings like pm.max_children, the status page, and fixes for 502, 504, and "File not found" errors.

Date: 2026-10-13

**Short answer:** Nginx can't execute PHP by itself, so it hands `.php` requests to **PHP-FPM** over the FastCGI protocol. Install it with `sudo apt install php-fpm`, find the socket with `ls /run/php/`, and add a `location ~ \.php$ { include snippets/fastcgi-php.conf; fastcgi_pass unix:/run/php/php8.3-fpm.sock; }` block to your server block (use the socket name you found). Run `sudo nginx -t && sudo systemctl reload nginx`, then load a `phpinfo()` page to confirm it works. After that, size `pm.max_children` to your RAM so PHP-FPM doesn't run out of workers under load, which is one of the most common causes of 502 and 504 errors on PHP sites.

## What Is PHP-FPM?

PHP-FPM (FastCGI Process Manager) is the standard way to run PHP behind Nginx. It keeps a **pool** of PHP worker processes running and listens on a Unix socket or TCP port. Nginx serves static files directly and forwards only PHP requests to the pool using the [ngx_http_fastcgi_module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html).

This is different from Apache's `mod_php`, where PHP runs inside the web server process. With Nginx and PHP-FPM:

| Component | Job |
| --- | --- |
| Nginx | Accepts connections, terminates TLS, serves images, CSS, and JS, and passes `.php` requests on |
| FastCGI | The protocol Nginx uses to talk to PHP-FPM |
| PHP-FPM master | Reads pool config, starts and stops workers, writes the FPM log |
| PHP-FPM workers | Execute PHP scripts, one request at a time each |

Because each worker handles one request at a time, the number of workers caps how many PHP requests you can process at once. That's the setting you'll tune later.

## Step 1: Install PHP-FPM

On Ubuntu or Debian, the `php-fpm` package installs the PHP version your release ships with:

```bash
sudo apt update
sudo apt install php-fpm php-mysql php-curl php-mbstring php-xml
```

Find the installed version, service name, and socket:

```bash
php -v
ls /run/php/
# php8.3-fpm.pid  php8.3-fpm.sock  php-fpm.sock
systemctl status 'php*-fpm'
```

The version number in the socket and service name depends on your distribution. This guide uses `php8.3-fpm` (the default on Ubuntu 24.04). Replace it with whatever `ls /run/php/` shows. Debian and Ubuntu also create a version-less `php-fpm.sock` symlink through the alternatives system, which you can use if you want configs that survive PHP upgrades.

If you don't have Nginx yet, follow [how to install Nginx on Ubuntu](https://greencrew.space/blog/how-to-install-nginx-on-ubuntu) first.

## Step 2: Configure the Nginx Server Block

Create or edit `/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
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

    # Block access to hidden files like .env and .git
    location ~ /\.(?!well-known) {
        deny all;
    }
}
```

Enable it and reload:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

The `try_files ... /index.php?$query_string` line is the front-controller pattern used by Laravel, Symfony, WordPress, and most modern PHP frameworks. Any URL that isn't a real file is routed to `index.php`.

### What's Inside snippets/fastcgi-php.conf

Debian and Ubuntu's Nginx package ships this helper. It does the safe parts for you:

| Line | Purpose |
| --- | --- |
| `fastcgi_split_path_info ^(.+?\.php)(/.*)$;` | Splits `/index.php/foo` into script and `PATH_INFO` |
| `try_files $fastcgi_script_name =404;` | Returns 404 if the PHP file doesn't exist, instead of passing it to PHP |
| `fastcgi_param PATH_INFO $path_info;` | Passes the extra path to PHP |
| `fastcgi_index index.php;` | Default script for directory requests |
| `include fastcgi.conf;` | Standard FastCGI params, **including `SCRIPT_FILENAME`** |

The `try_files $fastcgi_script_name =404` check matters for security. Without it, a request like `/uploads/avatar.jpg/x.php` could trick PHP into executing an uploaded file. Older guides fix this by setting `cgi.fix_pathinfo=0` in `php.ini`. With the `try_files` check in place, that's a second layer rather than the only one.

### On Other Distros: fastcgi.conf vs fastcgi_params

If your Nginx doesn't have `snippets/fastcgi-php.conf` (for example, packages from nginx.org or RHEL-based systems), write the block yourself:

```nginx
location ~ \.php$ {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+?\.php)(/.*)$;
    fastcgi_pass unix:/run/php-fpm/www.sock;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
}
```

`fastcgi.conf` and `fastcgi_params` are nearly identical. The difference is that `fastcgi.conf` already sets `SCRIPT_FILENAME`. If you include `fastcgi_params` and forget that line, PHP-FPM doesn't know which file to run and answers "File not found."

## Step 3: Test PHP Processing

Create a test file, load it, then delete it (phpinfo leaks a lot about your server):

```bash
echo '<?php phpinfo();' | sudo tee /var/www/example.com/public/info.php
curl -s http://example.com/info.php | grep -o 'Server API.*FPM/FastCGI'
sudo rm /var/www/example.com/public/info.php
```

If the browser **downloads** the `.php` file instead of running it, Nginx didn't match your `location ~ \.php$` block. Check that the site is enabled and that you reloaded Nginx.

## Unix Socket vs TCP Port

PHP-FPM can listen on either. Set it with `listen` in the pool file:

| Option | Pool `listen` | Nginx `fastcgi_pass` | Use when |
| --- | --- | --- | --- |
| Unix socket | `/run/php/php8.3-fpm.sock` | `unix:/run/php/php8.3-fpm.sock` | Nginx and PHP are on the same server (default, slightly less overhead) |
| TCP | `127.0.0.1:9000` | `127.0.0.1:9000` | PHP-FPM runs in a separate container or host |

With a socket, the Nginx user (`www-data` on Ubuntu) needs permission to read and write it. The default pool already sets this:

```ini
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```

With TCP on anything other than localhost, restrict who can connect with `listen.allowed_clients` and a firewall, because PHP-FPM executes whatever script path it's asked to run.

## Step 4: Tune the PHP-FPM Pool

The default pool lives at `/etc/php/8.3/fpm/pool.d/www.conf`. These are the settings that matter most:

| Directive | Default in www.conf | What it does |
| --- | --- | --- |
| `pm` | `dynamic` | How workers are managed: `static`, `dynamic`, or `ondemand` |
| `pm.max_children` | `5` | Hard cap on workers, which is the max number of concurrent PHP requests |
| `pm.start_servers` | `2` | Workers started at boot (`dynamic` only) |
| `pm.min_spare_servers` | `1` | Minimum idle workers kept ready |
| `pm.max_spare_servers` | `3` | Idle workers above this are killed |
| `pm.max_requests` | `0` (unlimited) | Recycle a worker after N requests, which contains memory leaks |
| `request_terminate_timeout` | `0` (off) | Kill a request running longer than this |

A default of 5 workers is tiny. Five slow requests (a slow database query, an external API call) and every other visitor waits in the queue.

### Choose a pm Mode

| Mode | Behavior | Best for |
| --- | --- | --- |
| `static` | Always runs exactly `pm.max_children` workers | Dedicated servers with predictable traffic |
| `dynamic` | Scales between spare limits, up to `max_children` | Most servers |
| `ondemand` | Starts workers only when requests arrive | Low-traffic sites or many pools on one small VPS |

### Size pm.max_children from Memory

Measure the average memory per worker under real traffic:

```bash
ps --no-headers -o rss -C php-fpm8.3 | awk '{sum+=$1; n++} END {printf "%.1f MB avg over %d procs\n", sum/n/1024, n}'
```

Then calculate: **max_children = RAM you can give PHP / average worker size**. For example, if you can spare 2 GB for PHP and workers average 60 MB, that's about 33. Leave headroom for the database, Nginx, and the OS. Setting it too high is worse than too low, because the server swaps or the OOM killer starts terminating processes.

A reasonable starting point for a 4 GB VPS running only the web stack:

```ini
pm = dynamic
pm.max_children = 30
pm.start_servers = 6
pm.min_spare_servers = 4
pm.max_spare_servers = 10
pm.max_requests = 500
request_terminate_timeout = 60s
```

Apply it:

```bash
sudo php-fpm8.3 -t && sudo systemctl reload php8.3-fpm
```

When you hit the cap, PHP-FPM tells you in its log:

```bash
sudo grep max_children /var/log/php8.3-fpm.log
# WARNING: [pool www] server reached pm.max_children setting (5), consider raising it
```

### Log Slow Requests

To find out *why* workers are busy, turn on the slow log. It dumps a PHP stack trace for any request that runs too long:

```ini
request_slowlog_timeout = 5s
slowlog = /var/log/php8.3-fpm-slow.log
```

## Step 5: Enable the PHP-FPM Status Page

PHP-FPM can report active, idle, and queued workers. Turn it on in the pool:

```ini
pm.status_path = /fpm-status
ping.path = /fpm-ping
```

Expose it to localhost only in Nginx:

```nginx
location ~ ^/(fpm-status|fpm-ping)$ {
    allow 127.0.0.1;
    deny all;
    access_log off;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
}
```

```bash
curl -s http://127.0.0.1/fpm-status
# active processes:     3
# idle processes:       7
# listen queue:         0
# max children reached: 0
```

A growing `listen queue` or a non-zero `max children reached` means requests are waiting for a free worker. See the [PHP-FPM configuration docs](https://www.php.net/manual/en/install.fpm.configuration.php) for every field.

## Recommended FastCGI Settings in Nginx

| Directive | Default | When to change it |
| --- | --- | --- |
| `fastcgi_read_timeout` | `60s` | Raise for long reports or imports; otherwise Nginx returns 504 |
| `fastcgi_buffer_size` | 4k or 8k | Raise to `32k` if you see "upstream sent too big header" |
| `fastcgi_buffers` | `8 4k` or `8 8k` | Raise to `16 16k` for large responses |
| `fastcgi_intercept_errors` | `off` | Set `on` to show Nginx's error pages for PHP 4xx/5xx responses |
| `client_max_body_size` | `1m` | Raise for file uploads, and match `upload_max_filesize` in `php.ini` |

Keep `fastcgi_read_timeout` and PHP's `max_execution_time` / `request_terminate_timeout` in step. If Nginx gives up at 60 seconds while PHP keeps running for 300, the user sees a 504 and the work keeps consuming a worker.

## Troubleshooting Nginx and PHP-FPM

| Symptom | Cause | Fix |
| --- | --- | --- |
| 502 Bad Gateway, `connect() to unix:... failed (2: No such file or directory)` | Wrong socket path or PHP-FPM is stopped | `ls /run/php/`, fix `fastcgi_pass`, `sudo systemctl start php8.3-fpm` |
| 502, `(13: Permission denied)` on the socket | Nginx user can't access the socket | Set `listen.owner`/`listen.group` to the Nginx user and reload PHP-FPM |
| 502, `(11: Resource temporarily unavailable)` | Socket backlog full; all workers busy | Raise `pm.max_children`, fix slow requests |
| 504 Gateway Timeout | Script ran past `fastcgi_read_timeout` | Fix the slow code or raise the timeout |
| "File not found." / `Primary script unknown` in the log | `SCRIPT_FILENAME` missing or `root` wrong | Use `fastcgi.conf` or set `SCRIPT_FILENAME $document_root$fastcgi_script_name` |
| Browser downloads `.php` files | No matching `location ~ \.php$` | Add the block and reload Nginx |
| `upstream sent too big header` | Large cookies or headers from the app | Raise `fastcgi_buffer_size` and `fastcgi_buffers` |
| Changes to `php.ini` have no effect | Edited the CLI `php.ini` | Edit `/etc/php/8.3/fpm/php.ini` and reload PHP-FPM |

For a deeper walkthrough of gateway errors, see [how to fix 502 Bad Gateway in Nginx](https://greencrew.space/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Your PHP Site from the Outside

Pool exhaustion rarely shows up as a clean crash. It looks like a site that's fine at 3 a.m. and returns 502s during your busiest hour. You'll only see it in the logs after the fact unless something is checking for you.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks against your site from outside your network. When PHP-FPM stops answering and Nginx starts returning 502 or 504, you get an alert on Discord, Slack, or Telegram, and your public status page shows customers that you're on it. Point a check at a page that actually runs PHP (not a static file), so the check fails when PHP does. Setup steps are in [how to get website downtime alerts on Discord, Slack, and Telegram](https://greencrew.space/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should I use a Unix socket or TCP for PHP-FPM?

Use a Unix socket when Nginx and PHP-FPM run on the same machine. It skips the TCP stack and avoids exposing a port. Use TCP when PHP-FPM runs in another container or host, and restrict access to it.

### What should pm.max_children be?

Divide the memory you can dedicate to PHP by the average memory of one worker, measured under real traffic. There's no universal number. A small VPS may handle 10–20, while a large dedicated server might run hundreds.

### Do I need cgi.fix_pathinfo=0?

Not if your PHP location checks that the script exists (`try_files $fastcgi_script_name =404` or `try_files $uri =404`). That check stops the path-info exploit. Setting `cgi.fix_pathinfo=0` as well is harmless for most apps but can break ones that rely on `PATH_INFO`.

### Can I run multiple PHP versions with one Nginx?

Yes. Install each version's FPM package (for example from a PPA), and each gets its own socket under `/run/php/`. Point each server block's `fastcgi_pass` at the socket for the version that site needs.

## Conclusion

Connecting Nginx to PHP-FPM takes one `location ~ \.php$` block with `fastcgi_pass` pointed at the right socket. Keeping it fast takes a bit more: size `pm.max_children` from real memory usage, turn on the slow log and status page, and keep Nginx and PHP timeouts in step. Put an [Nginx reverse proxy](https://greencrew.space/blog/how-to-set-up-nginx-reverse-proxy) or [proxy cache](https://greencrew.space/blog/how-to-set-up-nginx-proxy-cache) in front if you need more headroom, and let [GreenCrew](https://greencrew.space) alert you the moment PHP stops responding.
