# How to Deploy Django with PostgreSQL, Gunicorn, and Nginx on Ubuntu (systemd Socket and Static Files)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Django, Python, PostgreSQL, Gunicorn, DevOps

Description: To deploy Django in production, run it with Gunicorn under a systemd socket at /run/gunicorn.sock, point Nginx at it with proxy_pass http://unix:/run/gunicorn.sock;, serve collected static files with an alias location, and store data in PostgreSQL. This guide covers the Postgres database, virtualenv, production settings, collectstatic, systemd units, HTTPS, and fixing 502, 400, and CSRF errors.

Date: 2026-10-19

**Short answer:** Django's `runserver` is for development only. In production, create a PostgreSQL database and user, install your project in a virtualenv with `gunicorn` and `psycopg`, and set `DEBUG = False`, `ALLOWED_HOSTS`, and `STATIC_ROOT`. Run `python manage.py migrate` and `collectstatic`, then start Gunicorn through a systemd socket and service listening on `/run/gunicorn.sock`. Nginx serves `/static/` straight from disk with `alias` and forwards every other request with `proxy_pass http://unix:/run/gunicorn.sock;`. Add HTTPS with Certbot and set `SECURE_PROXY_SSL_HEADER` so Django knows the request was secure.

## How Django, Gunicorn, and Nginx Work Together

Django is a Python web framework, but it doesn't include a production web server. The standard setup splits the work into three layers:

| Layer | Software | Job |
| --- | --- | --- |
| Web server | Nginx | Ports 80/443, TLS, static and media files, buffering slow clients |
| Application server | Gunicorn | Runs Django through its WSGI interface with several worker processes |
| Database | PostgreSQL | Stores application data |
| Process manager | systemd | Starts Gunicorn at boot and restarts it on failure |

A request flows like this: browser → Nginx → Unix socket → Gunicorn worker → Django view → PostgreSQL. Files under `/static/` never reach Python at all.

### Why Not runserver in Production?

The [Django docs](https://docs.djangoproject.com/en/stable/ref/django-admin/#runserver) are explicit: `runserver` hasn't gone through security audits or performance tests and shouldn't be used in production. Gunicorn is a mature WSGI server, and Nginx in front protects it from slow clients and serves files much faster than Python can.

## Prerequisites

- Ubuntu 24.04 or 22.04 with a sudo user
- Nginx installed ([how to install Nginx on Ubuntu](/blog/how-to-install-nginx-on-ubuntu))
- A Django project in Git, and a domain pointed at the server for HTTPS

This guide puts the project in `/srv/myproject` and calls the Django project package `myproject`. Replace both with your own names.

## Step 1: Install Python, PostgreSQL, and Build Tools

```bash
sudo apt update
sudo apt install python3-venv python3-dev libpq-dev postgresql postgresql-contrib nginx curl
```

`libpq-dev` and `python3-dev` are only needed if you compile the PostgreSQL driver from source. They're small and save you a confusing build error later.

## Step 2: Create the PostgreSQL Database and User

Open a Postgres shell as the `postgres` system user:

```bash
sudo -u postgres psql
```

```sql
CREATE USER myprojectuser WITH PASSWORD 'a-long-random-password';
CREATE DATABASE myproject OWNER myprojectuser;
ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
ALTER ROLE myprojectuser SET timezone TO 'UTC';
\q
```

Making the app user the **owner** of the database matters on PostgreSQL 15 and later, where ordinary users can no longer create tables in the `public` schema by default. Without ownership, `migrate` fails with `permission denied for schema public`. The three `ALTER ROLE` lines match the defaults Django expects, as described in Django's [PostgreSQL notes](https://docs.djangoproject.com/en/stable/ref/databases/#optimizing-postgresql-s-configuration), so it doesn't have to set them on every connection.

## Step 3: Set Up the Project and Virtualenv

```bash
sudo mkdir -p /srv/myproject
sudo chown $USER:www-data /srv/myproject
git clone https://github.com/you/myproject.git /srv/myproject
cd /srv/myproject
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn "psycopg[binary]"
```

`psycopg` (version 3) is supported by current Django releases; older projects may use `psycopg2-binary` instead. Either works. Add `gunicorn` and the driver to `requirements.txt` so the next deploy installs them.

Putting the project in `/srv` instead of your home directory avoids a common trap: Ubuntu home directories are often not readable by other users, so Nginx gets `Permission denied` when it tries to serve static files from them.

## Step 4: Configure Django Settings for Production

Keep secrets out of `settings.py` and read them from the environment:

```python
# myproject/settings.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = False
ALLOWED_HOSTS = ["example.com", "www.example.com"]
CSRF_TRUSTED_ORIGINS = ["https://example.com", "https://www.example.com"]

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "myproject",
        "USER": "myprojectuser",
        "PASSWORD": os.environ["DB_PASSWORD"],
        "HOST": "localhost",
        "PORT": "5432",
    }
}

STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"
MEDIA_URL = "media/"
MEDIA_ROOT = BASE_DIR / "media"

# Nginx terminates TLS and tells Django via this header
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

Create an environment file that only your deploy user and the `www-data` group can read:

```bash
cat > /srv/myproject/.env <<'EOF'
DJANGO_SECRET_KEY=replace-with-a-long-random-string
DB_PASSWORD=a-long-random-password
EOF
chmod 640 /srv/myproject/.env
```

| Setting | Why it matters |
| --- | --- |
| `DEBUG = False` | `True` shows full tracebacks and settings to anyone who triggers an error |
| `ALLOWED_HOSTS` | Requests with any other `Host` header get `400 Bad Request` |
| `CSRF_TRUSTED_ORIGINS` | Needed for HTTPS form posts on Django 4.0+; values must include the scheme |
| `STATIC_ROOT` | Where `collectstatic` copies files for Nginx to serve |
| `SECURE_PROXY_SSL_HEADER` | Makes `request.is_secure()` true behind Nginx. **Only safe if Nginx always sets the header** |

Generate a secret key with `python -c "import secrets; print(secrets.token_urlsafe(50))"`.

## Step 5: Migrate, Collect Static Files, and Test Gunicorn

```bash
cd /srv/myproject
source .venv/bin/activate
set -a; source .env; set +a
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
python manage.py check --deploy
```

`check --deploy` lists security settings you've missed, such as HSTS. Fix what applies to you.

Now run Gunicorn by hand to make sure the app loads:

```bash
gunicorn --bind 127.0.0.1:8000 myproject.wsgi:application
# in a second terminal:
curl -I -H "Host: example.com" http://127.0.0.1:8000/
```

A `200`, `301`, or `302` means Django is working. Stop Gunicorn with `Ctrl+C`.

## Step 6: Run Gunicorn with a systemd Socket and Service

A systemd **socket** unit creates `/run/gunicorn.sock` at boot and starts Gunicorn on the first request. If Gunicorn crashes, systemd keeps the socket open and restarts it.

Create `/etc/systemd/system/gunicorn.socket`:

```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock
SocketUser=www-data

[Install]
WantedBy=sockets.target
```

Create `/etc/systemd/system/gunicorn.service`:

```ini
[Unit]
Description=gunicorn daemon for myproject
Requires=gunicorn.socket
After=network.target postgresql.service

[Service]
User=deploy
Group=www-data
WorkingDirectory=/srv/myproject
EnvironmentFile=/srv/myproject/.env
ExecStart=/srv/myproject/.venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --timeout 60 \
          --bind unix:/run/gunicorn.sock \
          myproject.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Replace `deploy` with the user that owns `/srv/myproject`. Then start the socket and test it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn.socket
file /run/gunicorn.sock
curl --unix-socket /run/gunicorn.sock -H "Host: example.com" -I http://localhost/
sudo systemctl status gunicorn
```

The `curl` request activates `gunicorn.service`. Logs go to the journal: `sudo journalctl -u gunicorn -f`.

| Gunicorn option | Recommendation |
| --- | --- |
| `--workers` | The [Gunicorn docs](https://docs.gunicorn.org/en/stable/design.html#how-many-workers) suggest `(2 x CPU cores) + 1` as a starting point; lower it on small-RAM servers |
| `--timeout` | Kill a worker stuck longer than this (default 30s). Keep it below Nginx's `proxy_read_timeout` |
| `--access-logfile -` | Send access logs to stdout, which systemd captures |
| `--bind unix:...` | Unix socket: no open TCP port, slightly less overhead |
| `HUP` signal | `systemctl reload gunicorn` starts new workers with new code, then stops the old ones |

## Step 7: Configure Nginx for Django

Create `/etc/nginx/sites-available/myproject`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    client_max_body_size 20M;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /srv/myproject/staticfiles/;
        expires 30d;
        access_log off;
    }

    location /media/ {
        alias /srv/myproject/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
sudo ufw allow 'Nginx Full'
```

`proxy_params` ships with Ubuntu's Nginx package and sets `Host`, `X-Real-IP`, `X-Forwarded-For`, and `X-Forwarded-Proto`. Note the trailing slashes in `alias`: `location /static/` must pair with `alias .../staticfiles/`, or Nginx builds the wrong path. For other proxy headers and timeouts, see [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy).

## Step 8: Add HTTPS with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot adds the certificate and an HTTP to HTTPS redirect. Because `proxy_params` sends `X-Forwarded-Proto: https` and Django trusts it through `SECURE_PROXY_SSL_HEADER`, secure cookies and CSRF checks work without changes. Full details: [how to install Let's Encrypt SSL on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

## Deploying Django Updates

```bash
cd /srv/myproject
git pull
source .venv/bin/activate
set -a; source .env; set +a
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
sudo systemctl reload gunicorn
```

`reload` sends `HUP`, so Gunicorn swaps workers without dropping the socket. Use `restart` if you changed the systemd unit or Python version.

## Troubleshooting Django with Gunicorn and Nginx

| Symptom | Cause | Fix |
| --- | --- | --- |
| 502, error log: `connect() to unix:/run/gunicorn.sock failed (2: No such file or directory)` | Socket unit not running | `sudo systemctl enable --now gunicorn.socket` |
| 502, error log: `(13: Permission denied)` on the socket | Nginx (`www-data`) can't open the socket | Set `SocketUser=www-data` or matching group, reload the units |
| 502 right after the first request | Gunicorn crashes on import (bad setting, missing env var) | `sudo journalctl -u gunicorn -n 50` |
| `Bad Request (400)` | `Host` not in `ALLOWED_HOSTS` | Add the domain, reload Gunicorn |
| `CSRF verification failed` on HTTPS | Origin not trusted, or Django thinks the request is HTTP | Set `CSRF_TRUSTED_ORIGINS` with `https://`, check `SECURE_PROXY_SSL_HEADER` |
| Admin has no CSS | `collectstatic` not run, or `alias` path wrong | Run `collectstatic`, compare `STATIC_ROOT` to the `alias` path |
| Static files return 403 | Nginx can't traverse a parent directory | `namei -l /srv/myproject/staticfiles` and fix permissions |
| Redirect loop after HTTPS | `SECURE_SSL_REDIRECT = True` but the proto header isn't forwarded | Make sure `include proxy_params;` is in `location /` |
| `permission denied for schema public` | PostgreSQL 15+ schema permissions | Make the app user the database owner (Step 2) |
| 504 Gateway Timeout | Slow view exceeds Gunicorn `--timeout` or Nginx timeout | Move slow work to a task queue (Celery, RQ), or raise both timeouts |

Gateway errors are covered in more depth in [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx).

## Monitor Your Django Site

systemd restarts Gunicorn when it crashes, but it can't tell when every worker is up and returning 500 because PostgreSQL ran out of connections or a migration didn't run. Add a small health view that touches the database:

```python
# myproject/urls.py
from django.db import connection
from django.http import JsonResponse
from django.urls import path

def health(request):
    with connection.cursor() as cursor:
        cursor.execute("SELECT 1")
    return JsonResponse({"status": "ok"})

urlpatterns = [
    path("health/", health),
    # ...
]
```

Then monitor it from outside. [GreenCrew](https://greencrew.space) checks `https://example.com/health/` over HTTP, sends alerts to Discord, Slack, or Telegram when it fails, and gives you a public status page for your users. Because the check runs through Nginx, TLS, Gunicorn, Django, and PostgreSQL, a failure anywhere in the chain shows up. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Should Gunicorn use a Unix socket or a TCP port?

Use a Unix socket when Nginx and Gunicorn are on the same server. It doesn't open a network port and has slightly less overhead. Use TCP (`--bind 127.0.0.1:8000`) when Gunicorn runs in a container or on a different host.

### Can Django serve its own static files in production?

Django's `staticfiles` app only serves files when `DEBUG = True`. In production, let Nginx serve `STATIC_ROOT` as shown above, or use the WhiteNoise package if you can't change the web server config.

### How many Gunicorn workers do I need?

Start with `(2 x CPU cores) + 1` and watch memory. Each worker is a full copy of your Django app, so on a 1 GB server, 2–3 workers is usually the limit. For I/O-heavy apps, Gunicorn's `gthread` worker class with `--threads` can handle more concurrent requests per worker.

### Do I need uWSGI instead of Gunicorn?

No. Both are production WSGI servers. Gunicorn has fewer options and is simpler to configure, which is why most Django deployment guides use it. If you need ASGI (async views, WebSockets), run Uvicorn workers or Daphne instead.

## Conclusion

A production Django deployment on Ubuntu has a clear shape: PostgreSQL owned by an app user, Django in a virtualenv with `DEBUG = False` and secrets in an environment file, Gunicorn on a systemd socket, and Nginx serving static files and proxying the rest. Run `migrate` and `collectstatic` on each deploy, reload Gunicorn, and keep HTTPS on with Certbot. Then point [GreenCrew](https://greencrew.space) at your `/health/` view so you know right away when the site stops answering.
