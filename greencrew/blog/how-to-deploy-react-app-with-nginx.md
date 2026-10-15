# How to Deploy a React App with Nginx on Ubuntu (try_files for React Router, Caching, and API Proxy)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, React, Vite, Frontend, DevOps

Description: To deploy a React app with Nginx, run npm run build, copy the build folder (dist for Vite) to /var/www/your-app, and add a server block with root pointing at it and try_files $uri $uri/ /index.html so React Router routes don't return 404. This guide covers the full config, caching hashed assets, proxying /api to a backend, subfolder deploys, and HTTPS.

Date: 2026-10-15

**Short answer:** A React app is static files after you build it, so Nginx serves it directly with no Node.js process. Run `npm run build`, copy the output folder (`dist/` for Vite, `build/` for Create React App) to `/var/www/myapp`, and set `root /var/www/myapp;` in a server block. The line that matters is **`try_files $uri $uri/ /index.html;`**. It sends every URL that isn't a real file to `index.html`, so client-side routes like `/dashboard` work on refresh instead of returning **404 Not Found**. Then cache hashed assets for a year, don't cache `index.html`, and add HTTPS with Certbot.

## How Serving a React App with Nginx Works

When you run `npm run build`, your bundler turns JSX, CSS, and imports into plain HTML, JavaScript, and CSS files. The browser downloads them once, and from then on React Router changes pages in JavaScript without asking the server.

The catch: if a user refreshes on `/settings/profile` or opens it from a link, the browser asks Nginx for `/settings/profile`. That file doesn't exist on disk, so a default Nginx config returns 404. The fix is to fall back to `index.html` and let React Router render the right page.

| Build tool | Build command | Output folder | Hashed assets folder |
| --- | --- | --- | --- |
| Vite (`npm create vite@latest`) | `npm run build` | `dist/` | `dist/assets/` |
| Create React App (deprecated) | `npm run build` | `build/` | `build/static/` |
| Next.js static export (`output: 'export'`) | `npm run build` | `out/` | `out/_next/static/` |

Create React App is deprecated, and the [React docs](https://react.dev/learn/creating-a-react-app) recommend a framework or a build tool like Vite for new projects. This guide uses Vite's `dist/`, but the Nginx config is the same for any of them. Only the folder names change. (A Next.js app with server-side rendering or API routes is a Node.js server, not static files. Deploy it like any [Node.js app with PM2 and Nginx](https://greencrew.space/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx).)

## Prerequisites

- Ubuntu server with Nginx installed ([how to install Nginx on Ubuntu](https://greencrew.space/blog/how-to-install-nginx-on-ubuntu))
- Your React project building locally or in CI
- A domain pointed at the server for HTTPS

## Step 1: Build the React App

Build on your machine or in CI, not on a small production server. Builds use a lot of memory, and the server only needs the output.

```bash
npm ci
npm run build
ls dist
# assets  favicon.svg  index.html
```

### Set Environment Variables at Build Time

Variables are baked into the JavaScript bundle when you build. Changing them on the server does nothing until you rebuild.

| Tool | Prefix | Example | Read in code |
| --- | --- | --- | --- |
| Vite | `VITE_` | `VITE_API_URL=/api` | `import.meta.env.VITE_API_URL` |
| Create React App | `REACT_APP_` | `REACT_APP_API_URL=/api` | `process.env.REACT_APP_API_URL` |

Anything in the bundle is public. **Never put API secrets in a React environment variable.** Keep them in your backend.

## Step 2: Copy the Build to the Server

```bash
sudo mkdir -p /var/www/myapp
sudo chown $USER:$USER /var/www/myapp
rsync -avz --delete dist/ user@your-server:/var/www/myapp/
```

The trailing slash on `dist/` copies the folder's *contents*, and `--delete` removes files from old builds. Nginx's worker user (`www-data` on Ubuntu) needs read access, which the default permissions give it.

## Step 3: Create the Nginx Server Block

Create `/etc/nginx/sites-available/myapp`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/myapp;
    index index.html;

    # Single-page app fallback for React Router
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Hashed build assets never change, so cache them for a year
    location /assets/ {
        try_files $uri =404;
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Always revalidate the HTML entry point
    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;
}
```

Enable it and reload:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Test a deep link, which is the request that breaks without the fallback:

```bash
curl -I http://example.com/settings/profile
# HTTP/1.1 200 OK
# Content-Type: text/html
```

### What Each Directive Does

| Directive | Purpose |
| --- | --- |
| `root /var/www/myapp` | Folder Nginx serves files from |
| `try_files $uri $uri/ /index.html` | Serve the file if it exists, otherwise hand the URL to React Router |
| `location /assets/` + `try_files $uri =404` | A missing JS/CSS file returns a real 404 instead of HTML (which would show up as a confusing "Unexpected token '<'" error) |
| `expires 1y` + `immutable` | Browsers keep hashed files for a year without revalidating |
| `Cache-Control "no-cache"` on `index.html` | Browsers check for a new `index.html` on each visit, so users get the new build right after a deploy |
| `gzip on` | Compresses JS and CSS. See [Gzip and Brotli in Nginx](https://greencrew.space/blog/how-to-enable-gzip-and-brotli-compression-in-nginx) for Brotli |

The caching split works because the bundler puts a content hash in each asset filename (for example `index-B4x9kQ2a.js`). A new build gets new filenames, and the fresh `index.html` points at them. The `try_files` directive is documented in the [ngx_http_core_module reference](https://nginx.org/en/docs/http/ngx_http_core_module.html#try_files).

**Note on `add_header`:** Nginx only inherits `add_header` lines from the parent level when a block defines none of its own. If you set [security headers](https://greencrew.space/blog/how-to-add-security-headers-in-nginx) at the `server` level, repeat them inside any `location` that has its own `add_header`, or move them into a shared include file.

## Step 4: Proxy API Requests to Your Backend

Serving the frontend and API from the same domain avoids CORS entirely. Have React call relative paths like `/api/users`, and let Nginx forward them:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

With `proxy_pass http://127.0.0.1:3000;` (no path after the port), the backend receives the full `/api/users` path. If you write `proxy_pass http://127.0.0.1:3000/;` with a trailing slash, Nginx strips the `/api/` prefix and the backend receives `/users`. Pick the one your backend expects. More options are in [how to set up Nginx as a reverse proxy](https://greencrew.space/blog/how-to-set-up-nginx-reverse-proxy).

## Step 5: Deploy React to a Subfolder (Optional)

To serve the app at `example.com/app/` instead of the root, three things have to agree:

1. **Vite:** set `base: '/app/'` in `vite.config.js` (CRA uses `"homepage": "/app"` in `package.json`).
2. **React Router:** pass the base path, for example `<BrowserRouter basename="/app">` or `createBrowserRouter(routes, { basename: '/app' })`.
3. **Nginx:**

```nginx
location /app/ {
    alias /var/www/myapp/;
    try_files $uri $uri/ /app/index.html;
}
```

If any one of the three is missing, you'll get a blank page with 404s for `/assets/...` in the browser console.

## Step 6: Add HTTPS

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot edits the server block to listen on 443 and redirect HTTP to HTTPS. See [how to install Let's Encrypt SSL on Nginx with Certbot](https://greencrew.space/blog/how-to-install-lets-encrypt-ssl-nginx-certbot) for renewal checks.

## Zero-Downtime Deploys with Release Folders

`rsync --delete` into the live folder can briefly serve a new `index.html` before its assets have arrived. To swap builds atomically, upload each build to its own folder and switch a symlink:

```bash
RELEASE=/var/www/myapp-releases/$(date +%Y%m%d%H%M%S)
rsync -az dist/ user@your-server:$RELEASE/
ssh user@your-server "ln -sfn $RELEASE /var/www/myapp-new && mv -Tf /var/www/myapp-new /var/www/myapp"
```

`mv -T` replaces the symlink in one step. For this to work, `/var/www/myapp` has to be a symlink rather than a real directory, so set that up once before the first release. Keep the last few releases so rolling back is just pointing the symlink back. Old asset files from previous builds stay available to users who still have the old `index.html` open.

## Troubleshooting React on Nginx

| Symptom | Cause | Fix |
| --- | --- | --- |
| 404 on refresh or deep links like `/dashboard` | No SPA fallback | Add `try_files $uri $uri/ /index.html;` to `location /` |
| Blank white page, 404s for `/assets/*.js` in the console | Wrong `base`/`homepage` or wrong `root` | Match the bundler base path, Router `basename`, and Nginx location |
| `Uncaught SyntaxError: Unexpected token '<'` | A JS request got `index.html` back | Add `location /assets/ { try_files $uri =404; }` and check the file was uploaded |
| Users still see the old version after a deploy | `index.html` cached by the browser or a CDN | Send `Cache-Control: no-cache` for `index.html`, purge the CDN |
| 403 Forbidden | Nginx can't read the folder, or `index.html` is missing | `ls -la /var/www/myapp`, fix permissions, check you copied the build output |
| API calls return 404 or HTML | No `/api/` location, or the prefix is stripped incorrectly | Add the proxy block and check the trailing slash on `proxy_pass` |
| CORS errors | Frontend calls the API on a different origin | Proxy `/api/` through the same domain, or set CORS headers on the API |
| `.env` change has no effect | Variables are baked in at build time | Rebuild and redeploy |
| Default "Welcome to nginx!" page shows | Default site still enabled or `server_name` mismatch | Remove `sites-enabled/default`, check `server_name` |

## Monitor Your React Deployment

A static site rarely "crashes," but it still breaks: a deploy uploads an `index.html` pointing at missing assets, a certificate fails to renew, or the `/api/` backend goes down while the HTML shell keeps loading fine.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks against your site from outside your network and sends alerts to Discord, Slack, or Telegram when something fails. It also gives you a public status page to share with users. Monitor the homepage *and* an API endpoint like `https://example.com/api/health`, since the frontend can look healthy while the backend it depends on is down. Setup steps are in [how to get website downtime alerts on Discord, Slack, and Telegram](https://greencrew.space/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Why does my React app show 404 when I refresh the page?

Nginx looks for a file matching the URL path, and client-side routes like `/about` don't exist on disk. Add `try_files $uri $uri/ /index.html;` so Nginx returns `index.html` and React Router renders the route.

### Do I need Node.js on the server to run a React app?

No, not for a client-side React app. After `npm run build` it's plain static files, and Nginx serves them on its own. You only need Node.js on the server for server-side rendering (for example Next.js without static export) or for your API.

### Should I use HashRouter to avoid the 404 problem?

You can, but URLs like `/#/dashboard` look worse and are weaker for SEO. The `try_files` fallback fixes the problem on the server side and lets you keep clean URLs with `BrowserRouter`.

### How do I serve multiple React apps on one server?

Give each app its own server block with its own `server_name` and `root`, or serve them from subfolders using the `base` + `basename` + `alias` combination from Step 5.

## Conclusion

Deploying a React app with Nginx takes three steps: build it, copy the output folder to the server, and serve it with a `try_files $uri $uri/ /index.html` fallback so client-side routes survive a refresh. Add long-lived caching for hashed assets, `no-cache` for `index.html`, an `/api/` proxy to skip CORS, and HTTPS. Then add an external uptime check with [GreenCrew](https://greencrew.space) so a broken deploy or a dead backend reaches you as an alert before users notice.
