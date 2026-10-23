# How to Set Up Laravel, Nginx, and MySQL with Docker Compose (Dockerfile, nginx.conf, and .env)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Laravel, Docker, MySQL, PHP, DevOps

Description: To run Laravel with Docker Compose, define three services: app (a php:8.3-fpm image with pdo_mysql), nginx (with fastcgi_pass app:9000 and root /var/www/html/public), and mysql (with a named volume), then set DB_HOST=mysql in Laravel's .env. This guide gives you the full docker-compose.yml, Dockerfile, and Nginx config, plus migrations, queue workers, the scheduler, production notes, and fixes for common errors.

Date: 2026-10-23

**Short answer:** Create three Compose services. The first is `app`, built from `php:8.3-fpm` with the `pdo_mysql`, `zip`, `intl`, and `bcmath` extensions plus Composer. The second is `nginx`, built from the official `nginx` image, with a server block that uses `root /var/www/html/public`, `try_files $uri $uri/ /index.php?$query_string;`, and `fastcgi_pass app:9000;`. The third is `mysql`, built from the official `mysql` image, with a named volume and a healthcheck. Mount your Laravel project into both `app` and `nginx`. In `.env`, set `DB_CONNECTION=mysql` and `DB_HOST=mysql` (the service name, not `127.0.0.1`). Then run `docker compose up -d --build`, `docker compose exec app composer install`, and `docker compose exec app php artisan migrate`.

## What Does a Laravel Docker Compose Stack Look Like?

Laravel in Docker uses the same LEMP layout as on a regular server. The difference is that each piece runs in its own container, and the containers talk to each other over a private Compose network using their service names as hostnames:

| Service | Image | Job | Talks to |
| --- | --- | --- | --- |
| `nginx` | `nginx:stable-alpine` | Serves `public/` and passes PHP requests to FastCGI | `app:9000` |
| `app` | Custom, `FROM php:8.3-fpm` | Runs Laravel through PHP-FPM, Artisan, and Composer | `mysql:3306` |
| `mysql` | `mysql:8.4` | Application database with persistent storage | — |
| `queue` (optional) | Same image as `app` | Runs `php artisan queue:work` | `mysql`, Redis |
| `scheduler` (optional) | Same image as `app` | Runs `php artisan schedule:work` | `mysql` |

Laravel also ships [Laravel Sail](https://laravel.com/docs/sail), its official Docker development environment. Sail is a fine choice if you only need a local setup. Building the stack yourself, as this guide does, gives you an Nginx + PHP-FPM setup that matches a typical production server, and you control every file.

If you'd rather deploy Laravel directly on an Ubuntu server without containers, follow [how to deploy Laravel with Nginx on Ubuntu](/blog/how-to-deploy-laravel-with-nginx-on-ubuntu) instead.

## Prerequisites

- Docker Engine with the Compose plugin (`docker compose version` should work). See the [Docker install docs](https://docs.docker.com/engine/install/).
- A Laravel project, or an empty folder where you'll create one
- Basic familiarity with Laravel's `.env` file and Artisan

This guide uses PHP 8.3 and MySQL 8.4. Use whichever versions your Laravel release supports. Laravel lists its PHP requirements in the [installation docs](https://laravel.com/docs/installation).

## Step 1: Create the Laravel Project and Folder Structure

If you don't have a project yet, create one with the official Composer image so you don't need PHP installed on the host:

```bash
docker run --rm --user "$(id -u):$(id -g)" -v "$(pwd)":/app -w /app composer:2 \
  create-project laravel/laravel example-app
cd example-app
```

Then add a `docker/` directory for the container config:

```text
example-app/
├── app/  bootstrap/  config/  public/  routes/  storage/ ...
├── .env
├── docker-compose.yml
└── docker/
    ├── nginx/
    │   └── default.conf
    └── php/
        └── Dockerfile
```

## Step 2: Write the PHP-FPM Dockerfile

```dockerfile
# docker/php/Dockerfile
FROM php:8.3-fpm

ARG HOST_UID=1000
ARG HOST_GID=1000

RUN apt-get update && apt-get install -y --no-install-recommends \
        git unzip libzip-dev libicu-dev \
    && docker-php-ext-install pdo_mysql bcmath zip intl opcache \
    && rm -rf /var/lib/apt/lists/*

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Match www-data to your host user so bind-mounted files stay editable
RUN groupmod -o -g ${HOST_GID} www-data \
    && usermod -o -u ${HOST_UID} -g ${HOST_GID} www-data

ENV COMPOSER_HOME=/tmp/composer
WORKDIR /var/www/html
USER www-data
```

What each part does:

| Line | Why it's there |
| --- | --- |
| `FROM php:8.3-fpm` | Official PHP image with PHP-FPM listening on port 9000 |
| `docker-php-ext-install pdo_mysql ...` | Compiles extensions the base image doesn't include (`mbstring`, `tokenizer`, `ctype`, and `xml` are already built in) |
| `libzip-dev libicu-dev` | Headers needed to build the `zip` and `intl` extensions |
| `COPY --from=composer:2` | Copies the Composer binary from the official Composer image |
| `groupmod` / `usermod` | Makes files that Laravel writes to `storage/` belong to your host user, not a random UID |
| `USER www-data` | Runs PHP-FPM, Artisan, and Composer as a non-root user |

The extension helpers (`docker-php-ext-install`, `docker-php-ext-enable`) are documented on the [official PHP image page](https://hub.docker.com/_/php).

## Step 3: Write the Nginx Config for Laravel

```nginx
# docker/nginx/default.conf
server {
    listen 80;
    server_name _;
    root /var/www/html/public;
    index index.php;

    client_max_body_size 20m;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

There are three Docker-specific details here:

- **`fastcgi_pass app:9000`** uses the Compose service name. Docker's internal DNS resolves `app` to the PHP container, so you don't use a Unix socket or `127.0.0.1`.
- **`SCRIPT_FILENAME` is a path inside the `app` container.** Nginx doesn't read the PHP file itself. It tells PHP-FPM which file to run, so the project must be mounted at the **same path** (`/var/www/html`) in both containers.
- **`include fastcgi_params`** is used instead of Ubuntu's `snippets/fastcgi-php.conf`. That snippet is a Debian/Ubuntu packaging file that the official `nginx` image doesn't include.

The directives come from Nginx's [ngx_http_fastcgi_module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html). Socket and pool tuning are covered in [how to configure PHP-FPM with Nginx](/blog/how-to-configure-php-fpm-with-nginx).

## Step 4: Write docker-compose.yml

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
      args:
        HOST_UID: ${HOST_UID:-1000}
        HOST_GID: ${HOST_GID:-1000}
    volumes:
      - ./:/var/www/html
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped

  nginx:
    image: nginx:stable-alpine
    ports:
      - "8080:80"
    volumes:
      - ./:/var/www/html:ro
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app
    restart: unless-stopped

  mysql:
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 20
    restart: unless-stopped

volumes:
  mysql-data:
```

Compose automatically reads the `.env` file in the project folder for `${...}` substitution. Laravel's `.env` lives in that same folder, so the database name and credentials are defined **once** and shared by Laravel and the MySQL container.

| Setting | Purpose |
| --- | --- |
| `./:/var/www/html` bind mount | Code changes on your machine show up instantly (development) |
| `:ro` on the Nginx mount | Nginx only needs to read files |
| `mysql-data` named volume | The database survives `docker compose down` and container rebuilds |
| `healthcheck` + `condition: service_healthy` | `app` waits until MySQL accepts connections |
| `"8080:80"` | Open the site at `http://localhost:8080` |
| No `ports` on `mysql` | The database is reachable only from inside the Compose network |

To connect a desktop database client during development, add `ports: ["127.0.0.1:3306:3306"]` to `mysql`. This binds the port to localhost only, so it isn't exposed on every interface.

## Step 5: Configure Laravel's .env for Docker

Edit the database section of `.env`. Recent Laravel versions default to SQLite in new projects, so change the connection too:

```dotenv
APP_URL=http://localhost:8080

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=change-me-strong-password
DB_ROOT_PASSWORD=change-me-root-password

# Optional: only if your host user isn't UID/GID 1000 (check with `id -u` and `id -g`)
HOST_UID=1000
HOST_GID=1000
```

`DB_HOST=mysql` is the most important line. Inside the `app` container, `127.0.0.1` means the PHP container itself, where no database is running. Use the service name.

Don't set `DB_USERNAME=root`. The MySQL image creates `MYSQL_USER` as an extra account and refuses to start if you name it `root`. The root password comes from `MYSQL_ROOT_PASSWORD` instead. All the image's variables are listed on the [official MySQL image page](https://hub.docker.com/_/mysql).

## Step 6: Build, Start, and Run Migrations

```bash
docker compose up -d --build
docker compose ps
```

Then install dependencies and prepare the app inside the `app` container:

```bash
docker compose exec app composer install
docker compose exec app php artisan key:generate
docker compose exec app php artisan migrate
docker compose exec app php artisan storage:link
```

Open `http://localhost:8080`. You should see the Laravel welcome page. New Laravel versions also include a `/up` health route:

```bash
curl -i http://localhost:8080/up
```

To keep typing short, add a shell alias:

```bash
alias art='docker compose exec app php artisan'
art migrate:status
```

### Useful Docker Compose Commands for Laravel

| Task | Command |
| --- | --- |
| Follow all logs | `docker compose logs -f` |
| Laravel log | `docker compose exec app tail -f storage/logs/laravel.log` |
| Shell in the PHP container | `docker compose exec app bash` |
| MySQL shell | `docker compose exec mysql mysql -u laravel -p laravel` |
| Test the Nginx config | `docker compose exec nginx nginx -t` |
| Reload Nginx after config edits | `docker compose exec nginx nginx -s reload` |
| Rebuild after Dockerfile changes | `docker compose up -d --build app` |
| Stop (keep data) | `docker compose down` |
| Stop and **delete the database** | `docker compose down -v` |

## Step 7: Add a Queue Worker and the Scheduler

Queue workers and the scheduler use the same image as `app`, just with a different command. Add them under `services:`:

```yaml
  queue:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
      args:
        HOST_UID: ${HOST_UID:-1000}
        HOST_GID: ${HOST_GID:-1000}
    command: php artisan queue:work --sleep=3 --tries=3 --max-time=3600
    volumes:
      - ./:/var/www/html
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped

  scheduler:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
      args:
        HOST_UID: ${HOST_UID:-1000}
        HOST_GID: ${HOST_GID:-1000}
    command: php artisan schedule:work
    volumes:
      - ./:/var/www/html
    depends_on:
      mysql:
        condition: service_healthy
    restart: unless-stopped
```

`schedule:work` runs the scheduler in the foreground every minute, so you don't need cron inside the container. Queue workers keep your code in memory, so after you deploy new code run `docker compose exec app php artisan queue:restart` (or restart the `queue` service) so the workers pick up the changes. Details are in Laravel's [queue docs](https://laravel.com/docs/queues) and [scheduling docs](https://laravel.com/docs/scheduling).

## Step 8: Prepare the Stack for Production

The setup above is tuned for development. For a production server, change these:

| Development | Production |
| --- | --- |
| Bind mount `./:/var/www/html` | `COPY` the code into the image, so each release is one immutable image |
| `composer install` | `composer install --no-dev --optimize-autoloader` during the image build |
| `APP_ENV=local`, `APP_DEBUG=true` | `APP_ENV=production`, `APP_DEBUG=false` |
| No caching | `php artisan config:cache`, `route:cache`, and `view:cache` on container start |
| Port `8080` open to the world | Bind to `127.0.0.1:8080` and put a TLS-terminating proxy in front |
| Secrets in `.env` on disk | Restrict file permissions, or use Docker secrets or your platform's secret store |

Once the code isn't bind-mounted, Nginx still needs the files in `public/` (CSS, JS, images). Either build an Nginx image that copies `public/` from the same build, or share the directory through a named volume. Either way, both containers must keep using the same `/var/www/html` path.

For HTTPS on a single server, a simple pattern is to keep this stack bound to `127.0.0.1:8080` and run Nginx on the host as a reverse proxy with a Let's Encrypt certificate. See [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy) and [how to install Let's Encrypt SSL with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot). When Laravel runs behind a proxy, configure its trusted proxies so it generates `https://` URLs.

## Troubleshooting Laravel, Nginx, and MySQL in Docker

| Symptom | Cause | Fix |
| --- | --- | --- |
| `SQLSTATE[HY000] [2002] Connection refused` | `DB_HOST=127.0.0.1` or `localhost` | Set `DB_HOST=mysql`, then `php artisan config:clear` |
| `SQLSTATE[HY000] [1045] Access denied for user` | MySQL volume was initialized with old credentials. `MYSQL_*` variables only apply on **first** start | Change the password with SQL, or wipe the volume with `docker compose down -v` (deletes data) |
| MySQL container exits: `MYSQL_USER="root"` error | Used `root` as `DB_USERNAME` | Use a non-root username |
| 502 Bad Gateway | `app` container is down or `fastcgi_pass` host/port is wrong | `docker compose ps`, `docker compose logs app`, and check `fastcgi_pass app:9000` |
| `File not found.` or `Primary script unknown` | Code mounted at different paths in `nginx` and `app` | Mount the project at `/var/www/html` in both |
| 404 on every route except `/` | Missing `try_files ... /index.php?$query_string` | Add it to `location /` and reload Nginx |
| `Permission denied` on `storage/logs/laravel.log` | Container UID doesn't match the host files | Set `HOST_UID`/`HOST_GID` in `.env` to your `id -u`/`id -g` and rebuild |
| `.env` changes ignored | Config is cached | `docker compose exec app php artisan config:clear` |
| `NOTICE: [pool www] 'user' directive is ignored` | PHP-FPM runs as `www-data` already (`USER www-data`) | Harmless. You can ignore it |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` | Another service uses the port | Change `"8080:80"` to a free port |

For more Nginx gateway debugging, see [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Your Dockerized Laravel App

`restart: unless-stopped` brings a crashed container back up, but it can't fix every failure. A container that's running but stuck, a full disk, a database that won't come up, or a bad deploy that returns 500 on every page all look "up" to Docker.

[GreenCrew](https://greencrew.space) checks your site over HTTP from outside the server and alerts you on Discord, Slack, or Telegram when it stops responding. It also gives you a public status page to share with users. Point a check at your public URL plus Laravel's `/up` route, or at a route that runs a database query, so a failing MySQL container counts as downtime too. The setup takes a few minutes: [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Why does Nginx need its own container instead of running inside the PHP image?

The convention is one process per container. The official `php:*-fpm` image runs only PHP-FPM and has no web server. Running Nginx separately lets you update, restart, and scale each piece on its own, and the `app:9000` FastCGI link between them is just a network connection.

### Should I use Laravel Sail or a custom Docker Compose setup?

Use Sail if you want a ready-made local environment with no setup. Build your own Compose file when you want local development to match a production Nginx + PHP-FPM stack, need custom extensions, or plan to deploy the same images to a server.

### Will I lose my MySQL data when I run docker compose down?

No. The data lives in the named volume `mysql-data`, which `docker compose down` keeps. Only `docker compose down -v` or `docker volume rm` deletes it. Back up the database with `mysqldump` before running either.

### How do I run Composer and Artisan commands in Docker?

Run them inside the `app` container with `docker compose exec app composer require vendor/package` or `docker compose exec app php artisan <command>`. They run with the container's PHP version and extensions, so they match what serves your site.

## Conclusion

A Laravel Docker Compose setup comes down to three services and a few rules. Build `app` from `php:8.3-fpm` with the extensions Laravel needs. Point Nginx's `root` at `/var/www/html/public` and `fastcgi_pass` at `app:9000`. Mount the code at the same path in both containers. Give MySQL a named volume and a healthcheck, and connect to it with `DB_HOST=mysql`. Add `queue` and `scheduler` services from the same image, bake the code into the image for production, and put TLS in front. Then add an external [GreenCrew](https://greencrew.space) check on your `/up` route so you hear about a broken container before your users do.
