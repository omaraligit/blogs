# How to Install Node.js on Ubuntu (apt, NodeSource, and nvm Compared)

Author: [greencrew](https://greencrew.space/blog)

Tags: Node.js, Ubuntu, npm, nvm, Linux, DevOps

Description: To install a current LTS version of Node.js on Ubuntu, add the NodeSource repository with curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash - and run sudo apt install -y nodejs, or install nvm and run nvm install --lts for per-user version switching. This guide compares apt, NodeSource, and nvm, then covers npm global packages, build tools, switching versions, and uninstalling Node.js.

Date: 2026-10-28

**Short answer:** There are three common ways to install Node.js on Ubuntu. `sudo apt install nodejs npm` is the quickest, but it gives you Ubuntu's packaged version, which is often several major releases behind. For servers, add the **NodeSource** repository (`curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -`, then `sudo apt install -y nodejs`) to get a current LTS release that `apt upgrade` keeps patched. For development machines, or for running several versions side by side, install **nvm** and run `nvm install --lts`. Check the result with `node -v` and `npm -v`.

## Which Way Should You Install Node.js on Ubuntu?

| Method | Version you get | Who it's for | Updates |
| --- | --- | --- | --- |
| Ubuntu repo (`apt install nodejs`) | Whatever the Ubuntu release shipped with | Quick scripts, tools that only need *some* Node | `apt upgrade` (security patches only, same major) |
| NodeSource repo | Any current major (LTS or Current) | Production servers, CI runners | `apt upgrade` within the major you chose |
| nvm (Node Version Manager) | Any version, several at once | Developers, multiple projects, testing upgrades | `nvm install <version>` per user |
| Official binary tarball | Any version | Air-gapped or custom installs | Manual |

Ubuntu's repositories freeze Node.js at release time. Ubuntu 24.04 ships Node.js 18 and Ubuntu 22.04 ships Node.js 12, and both of those majors are past end of life upstream. Modern frameworks (Next.js, NestJS, Strapi, Vite) won't run on them. Unless you have a reason to stay on the distro package, use NodeSource or nvm.

### Pick an LTS Version

Node.js publishes a new major release every six months. Even-numbered majors become **Long-Term Support (LTS)** releases and get security fixes for about 30 months. Odd-numbered majors are short-lived "Current" releases. In production, always run an Active or Maintenance LTS version. The live schedule is on the [Node.js releases page](https://nodejs.org/en/about/previous-releases). This guide uses `24` in its examples. Substitute whichever LTS major is current when you read this.

## Prerequisites

- Ubuntu 24.04 or 22.04 (server, desktop, or WSL2)
- A user with `sudo` access
- `curl` installed (`sudo apt install -y curl`)

## Option 1: Install Node.js from Ubuntu's Repositories

```bash
sudo apt update
sudo apt install -y nodejs npm
node -v
npm -v
```

This is fine for running a small tool, but you're stuck with the distro's major version for the life of that Ubuntu release. If `node -v` prints a version older than your project's `engines` field, remove it (see [How to Uninstall Node.js](#how-to-uninstall-nodejs-from-ubuntu)) and use Option 2 or 3.

## Option 2: Install Node.js with NodeSource (Recommended for Servers)

[NodeSource](https://github.com/nodesource/distributions) maintains Debian/Ubuntu packages for every supported Node.js major. Their setup script adds the APT repository and signing key:

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x -o nodesource_setup.sh
less nodesource_setup.sh          # read it before running anything as root
sudo -E bash nodesource_setup.sh
sudo apt install -y nodejs
```

The NodeSource `nodejs` package includes `npm` (and `npx`), so don't install Ubuntu's separate `npm` package on top of it. Verify:

```bash
node -v
npm -v
which node        # /usr/bin/node
```

From now on, `sudo apt update && sudo apt upgrade` installs patch and minor releases within the major you chose. To move to the next major (24 → 26), run the setup script for the new major and `sudo apt install nodejs` again.

### Install Build Tools for Native Modules

Packages like `bcrypt`, `sharp` (when no prebuilt binary matches), and `sqlite3` compile C/C++ code during `npm install`. Install a compiler toolchain once:

```bash
sudo apt install -y build-essential python3
```

## Option 3: Install Node.js with nvm (Recommended for Development)

[nvm](https://github.com/nvm-sh/nvm) installs Node.js into your home directory, per user, with no `sudo`. It lets you keep several versions installed and switch per project.

Install nvm. Copy the current command from the nvm README, since the version in the URL changes with each release:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc          # or open a new terminal
nvm --version
```

Install and use Node.js:

```bash
nvm install --lts         # latest LTS
nvm install 22            # an older line, side by side
nvm ls                    # list installed versions
nvm use 24                # switch in this shell
nvm alias default 24      # version for new shells
```

### Pin a Version per Project with .nvmrc

```bash
cd ~/code/myapp
echo "24" > .nvmrc
nvm use                   # reads .nvmrc
nvm install               # installs it if missing
```

Commit `.nvmrc` so everyone on the team (and your CI) runs the same major. Pair it with an `engines` field in `package.json`:

```json
{
  "engines": {
    "node": ">=24"
  }
}
```

### nvm Command Reference

| Command | What it does |
| --- | --- |
| `nvm install --lts` | Install the newest LTS release |
| `nvm install 24` | Install the newest 24.x |
| `nvm use 22` | Switch the current shell to 22.x |
| `nvm alias default 24` | Default version for new shells |
| `nvm ls-remote --lts` | List all LTS versions available |
| `nvm install 24 --reinstall-packages-from=22` | Upgrade and carry over global packages |
| `nvm uninstall 20` | Remove a version |
| `nvm which 24` | Print the path to that `node` binary |

### nvm on Servers: One Caveat

nvm is a shell function loaded from `~/.bashrc`. systemd units, cron jobs, and `sudo` don't load your shell profile, so plain `node` won't be found there. Either reference the full path (`nvm which 24`), or use NodeSource on servers. Process managers like PM2 record the absolute path when you run `pm2 startup`, so they work with nvm as long as you start them from a shell where nvm is loaded.

## Configure npm Global Packages Without sudo

With **nvm**, `npm install -g` writes into your home directory. No `sudo` is needed, and you shouldn't use it.

With **NodeSource** or the Ubuntu package, global installs go to `/usr/lib/node_modules` and need `sudo`. To avoid running npm as root, point the global prefix at your home directory:

```bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
npm install -g pm2
```

### pnpm and Yarn

Corepack (which provides `pnpm` and `yarn` shims) is bundled with Node.js 24 and earlier. Newer majors no longer ship it, so install it from npm there:

```bash
corepack enable                 # Node 24 and earlier
npm install -g corepack && corepack enable   # newer Node majors
pnpm -v
```

## Test Your Node.js Install

Create a tiny HTTP server to confirm everything works:

```bash
mkdir -p ~/hello-node && cd ~/hello-node
cat > server.js <<'EOF'
const http = require('node:http');
http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end(`Hello from Node.js ${process.version}\n`);
}).listen(3000, '127.0.0.1', () => console.log('Listening on http://127.0.0.1:3000'));
EOF
node server.js
```

In another terminal, run `curl http://127.0.0.1:3000`. Once that works, you're ready to [run it in production with PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx) or [package it with Docker](/blog/how-to-build-a-nodejs-application-with-docker).

## How to Upgrade Node.js on Ubuntu

| Installed with | Upgrade within major | Upgrade to new major |
| --- | --- | --- |
| Ubuntu repo | `sudo apt upgrade` | Not possible. Switch to NodeSource or nvm. |
| NodeSource | `sudo apt upgrade` | Run the new `setup_XX.x` script, then `sudo apt install nodejs` |
| nvm | `nvm install 24` (gets latest 24.x) | `nvm install 26 --reinstall-packages-from=24` |

After a major upgrade, delete `node_modules` and run `npm ci`. Native modules are compiled against a specific Node ABI and must be rebuilt. Restart your app (for example `pm2 reload all`, then `pm2 save`).

## How to Uninstall Node.js from Ubuntu

**Ubuntu or NodeSource package:**

```bash
sudo apt remove --purge -y nodejs npm
sudo apt autoremove -y
ls /etc/apt/sources.list.d/ | grep -i nodesource   # remove that file to drop the repo
sudo apt update
```

**nvm:**

```bash
nvm deactivate
nvm uninstall 24
rm -rf ~/.nvm      # removes nvm itself; also delete the nvm lines from ~/.bashrc
```

## Troubleshooting Node.js on Ubuntu

| Symptom | Cause | Fix |
| --- | --- | --- |
| `node -v` shows an old version after installing NodeSource | Ubuntu's package still installed or pinned | `sudo apt remove nodejs npm`, rerun the NodeSource setup, `sudo apt install nodejs` |
| `nodejs : Conflicts: npm` | Ubuntu's `npm` package clashes with NodeSource's bundled npm | `sudo apt remove npm`. NodeSource already includes it. |
| `nvm: command not found` | Shell profile not reloaded | `source ~/.bashrc`, or open a new terminal |
| `node: not found` in cron/systemd | nvm only loads in interactive shells | Use the absolute path from `nvm which`, or NodeSource on servers |
| `EACCES: permission denied` on `npm install -g` | Global prefix owned by root | Set `npm config set prefix ~/.npm-global` or use nvm. Don't use `sudo npm`. |
| `gyp ERR! find Python` / `make: not found` | No build toolchain | `sudo apt install build-essential python3` |
| `Unsupported engine` warnings | Project needs a newer Node | Install the version in `.nvmrc` / `engines` |
| `Error: The module was compiled against a different Node.js version` | Native module built for old ABI | `rm -rf node_modules && npm ci` |

## Keep an Eye on the Apps You Run

Installing Node.js is step one. The apps you build on it still crash, run out of memory, or hang on a stuck database connection. Process managers restart a crashed process, but none of them tell you when your site is down from a user's point of view.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks against your app's public URL and alerts you on Discord, Slack, or Telegram when it stops responding correctly. It also gives you a public status page you can share with users. Point it at a `/health` route and see [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram) for setup.

## Frequently Asked Questions

### What's the best way to install Node.js on Ubuntu?

Use NodeSource on servers, because it's a normal APT package that gets security updates through `apt upgrade` and is available to every user and systemd service. Use nvm on development machines, where you need several versions and don't want `sudo` for global packages.

### Does NodeSource install npm too?

Yes. The NodeSource `nodejs` package bundles `npm` and `npx`. Installing Ubuntu's `npm` package alongside it causes a dependency conflict.

### Can I have nvm and NodeSource installed at the same time?

You can, but it's confusing. Whichever comes first in your `PATH` wins, and nvm puts its version first in interactive shells. If you need both (NodeSource for services, nvm for testing), check `which node` before debugging anything.

### Which Node.js version should I use in production?

An Active LTS release, or a Maintenance LTS release if your dependencies aren't ready for the newer one. Never run an odd-numbered Current release or an end-of-life major in production. Check the [Node.js release schedule](https://nodejs.org/en/about/previous-releases).

## Conclusion

Ubuntu's own `nodejs` package is too old for most real projects. Install Node.js from NodeSource on servers so `apt` keeps it patched, and use nvm on your workstation to switch versions per project with `.nvmrc`. Install `build-essential` for native modules, keep global npm packages out of root, and rebuild `node_modules` after every major upgrade. Then deploy your app behind [PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx) and let [GreenCrew](https://greencrew.space) alert you on Discord, Slack, or Telegram if it ever stops responding.
