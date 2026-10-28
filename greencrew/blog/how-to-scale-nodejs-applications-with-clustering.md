# How to Scale Node.js Applications with Clustering (cluster Module, Load Testing, and Zero-Downtime Restarts)

Author: [greencrew](https://greencrew.space/blog)

Tags: Node.js, Cluster, Performance, Nginx, Scaling, DevOps

Description: To scale a Node.js application across CPU cores, use the built-in node:cluster module: in the primary process call cluster.fork() once per os.availableParallelism(), and let each worker run your HTTP server on the same port. This guide covers a complete cluster script, restarting crashed workers, load testing with autocannon, zero-downtime rolling restarts, shared state with Redis, sticky sessions, and PM2 or Nginx alternatives.

Date: 2026-10-28

**Short answer:** A Node.js process runs your JavaScript on one CPU core. To use every core, import `node:cluster`. In the primary process (`cluster.isPrimary`), call `cluster.fork()` once per core from `os.availableParallelism()`. Each worker then runs your normal `app.listen(3000)`, and the primary distributes incoming connections between them. Restart workers when they exit, move in-memory state (sessions, caches, rate limits) to Redis, and measure the gain with `npx autocannon`. If you don't want to maintain cluster code yourself, `pm2 start app.js -i max` does the same thing.

## What Is Clustering in Node.js?

Clustering runs several copies of your Node.js server, called **workers**, that share one listening port. A **primary** process starts and supervises them. Each worker is a full Node process with its own event loop, memory, and V8 heap, so a CPU-heavy request on one worker doesn't stall the others.

| Without clustering | With clustering |
| --- | --- |
| One core busy, others idle | One worker per core |
| A crash takes the whole app down | Primary restarts the crashed worker. Others keep serving. |
| Restart means downtime | Rolling restart, one worker at a time |
| In-memory state just works | State must be shared (Redis, database) |

By default the primary accepts connections and hands them to workers round-robin (on every platform except Windows). See the [Node.js cluster docs](https://nodejs.org/api/cluster.html) for details.

### Clustering vs Worker Threads vs Multiple Servers

| Approach | Scales | Use when |
| --- | --- | --- |
| `node:cluster` | One machine, all cores, same port | You want to run your whole HTTP app on every core |
| [PM2 cluster mode](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx) | Same, managed for you | You already use PM2 on a VM |
| [`worker_threads`](/blog/how-to-use-multithreading-in-nodejs-with-worker-threads) | Specific CPU tasks inside a process | One route does heavy computation |
| Several processes + [Nginx load balancing](/blog/how-to-set-up-nginx-load-balancing-with-failover) | Many machines | You outgrow one server or need failover |
| Container replicas | Many machines | You run Docker Swarm, Kubernetes, or ECS |

## Prerequisites

- Node.js LTS ([how to install Node.js on Ubuntu](/blog/how-to-install-nodejs-on-ubuntu))
- An HTTP app. The examples use Express (`npm install express`).
- A machine with more than one CPU core (`nproc` tells you)

## Step 1: Start with a Single-Process App

```javascript
// app.js
const express = require('express');
const app = express();

app.get('/', (req, res) => res.send(`Handled by process ${process.pid}\n`));

app.get('/heavy', (req, res) => {
  let total = 0;
  for (let i = 0; i < 5e7; i++) total += i;   // simulate CPU work
  res.send(`Total ${total} from ${process.pid}\n`);
});

app.get('/health', (req, res) => res.json({ status: 'ok', pid: process.pid }));

module.exports = app;
```

```javascript
// server.js
const app = require('./app');
const port = process.env.PORT || 3000;
app.listen(port, () => console.log(`Worker ${process.pid} listening on ${port}`));
```

## Step 2: Measure the Baseline

Install nothing globally. Just run [autocannon](https://github.com/mcollina/autocannon) through `npx`:

```bash
node server.js
# in another terminal
npx autocannon -c 100 -d 20 http://localhost:3000/heavy
```

Write down requests per second and the p99 latency. Those are the numbers clustering should improve. Results depend entirely on your hardware and workload, so always measure your own app.

## Step 3: Add the cluster Module

Create `cluster.js`:

```javascript
// cluster.js
const cluster = require('node:cluster');
const os = require('node:os');

const numWorkers = Number(process.env.WEB_CONCURRENCY) || os.availableParallelism();

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} starting ${numWorkers} workers`);

  for (let i = 0; i < numWorkers; i++) cluster.fork();

  cluster.on('exit', (worker, code, signal) => {
    if (worker.exitedAfterDisconnect) return;   // intentional shutdown
    console.warn(`Worker ${worker.process.pid} died (${signal || code}). Starting a new one.`);
    cluster.fork();
  });
} else {
  require('./server');
}
```

```bash
node cluster.js
curl localhost:3000/    # run it a few times. The PID changes.
```

Run the same autocannon test against `/heavy`. CPU-bound routes should scale roughly with the number of cores. I/O-bound routes gain less, because one Node process already handles I/O concurrency well.

| API | Purpose |
| --- | --- |
| `cluster.isPrimary` | `true` in the supervising process (replaces the deprecated `isMaster`) |
| `cluster.fork(env)` | Start a worker, optionally with extra environment variables |
| `cluster.workers` | Object of live workers, keyed by ID |
| `worker.disconnect()` | Stop accepting new connections, exit after in-flight ones finish |
| `worker.exitedAfterDisconnect` | `true` if the exit was intentional |
| `cluster.on('exit' / 'online' / 'listening')` | Lifecycle events |
| `process.send()` / `worker.on('message')` | Messages between workers and the primary |

### How Many Workers?

Start with one per available core. On a server that also runs MySQL, Redis, or Nginx, leave a core or two for them. Use the `WEB_CONCURRENCY` variable above to tune the count without editing code. Each worker has its own memory, so check `free -m` under load: four workers use roughly four times the memory of one.

## Step 4: Shut Down Gracefully and Restart Without Downtime

Make each worker finish in-flight requests on `SIGTERM`:

```javascript
// server.js
const app = require('./app');
const server = app.listen(process.env.PORT || 3000);

process.on('SIGTERM', () => {
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 10000).unref();   // force-exit if a request hangs
});
```

Add a rolling restart to the primary. On `SIGUSR2`, it replaces workers one at a time, so there's always capacity:

```javascript
// inside the `if (cluster.isPrimary)` block
process.on('SIGUSR2', async () => {
  const workers = Object.values(cluster.workers);
  for (const old of workers) {
    const fresh = cluster.fork();
    await new Promise((resolve) => fresh.once('listening', resolve));
    old.disconnect();
    await new Promise((resolve) => old.once('exit', resolve));
  }
  console.log('Rolling restart complete');
});

process.on('SIGTERM', () => {
  for (const w of Object.values(cluster.workers)) w.disconnect();
});
```

Deploy new code, then trigger the reload with `kill -USR2 <primary-pid>`. New workers load the updated files from disk.

## Step 5: Move Shared State Out of Memory

Workers don't share memory. Anything stored in a JavaScript variable exists only in the worker that set it.

| State | Problem in a cluster | Fix |
| --- | --- | --- |
| `express-session` default MemoryStore | User logs in on worker 1, next request hits worker 2 and is logged out | `connect-redis` session store |
| In-memory cache (`Map`, `node-cache`) | Each worker caches separately, invalidation misses others | Redis cache, or accept per-worker caches |
| In-memory rate limiting | Limits multiply by worker count | Redis-backed limiter, or [rate limit in Nginx](/blog/how-to-set-up-rate-limiting-in-nginx) |
| Scheduled jobs (`node-cron`) | Job runs once per worker | Run jobs only in the primary, or in a separate process |
| WebSocket / Socket.IO rooms | Clients on different workers can't see each other | Sticky sessions + a shared adapter |

For scheduled jobs, the simplest pattern is to start one dedicated worker with `cluster.fork({ RUN_JOBS: '1' })` and check `process.env.RUN_JOBS` before scheduling anything.

### Sticky Sessions for Socket.IO

Socket.IO's HTTP long-polling fallback needs every request from a client to reach the same worker. The Socket.IO project provides [`@socket.io/sticky`](https://socket.io/docs/v4/using-multiple-nodes/) for the routing and `@socket.io/cluster-adapter` (or the Redis adapter) to broadcast between workers. If you run separate processes behind Nginx instead, `ip_hash` in the upstream block gives you stickiness.

## Step 6: Run the Cluster in Production

A cluster primary still needs something to start it on boot and restart it if it dies. With systemd:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Node.js cluster
After=network.target

[Service]
User=deploy
WorkingDirectory=/var/www/myapp
Environment=NODE_ENV=production PORT=3000 WEB_CONCURRENCY=4
ExecStart=/usr/bin/node cluster.js
ExecReload=/bin/kill -USR2 $MAINPID
KillSignal=SIGTERM
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl reload myapp      # rolling restart via SIGUSR2
```

Or skip the custom script: `pm2 start server.js -i max` runs the same pattern, and `pm2 reload` does the rolling restart. Either way, put Nginx in front for TLS, compression, and static files. See [how to set up Nginx as a reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy).

## Troubleshooting Node.js Clustering

| Symptom | Cause | Fix |
| --- | --- | --- |
| No speedup after clustering | Workload is I/O-bound, or the bottleneck is the database | Profile first. Scale the database or add caching. |
| Throughput drops with more workers | More workers than cores, or memory swapping | Lower `WEB_CONCURRENCY`, check `free -m` |
| Users logged out randomly | Sessions in memory | Redis session store |
| Cron job runs N times | Scheduled in every worker | Run jobs in one designated worker |
| Workers restart in a tight loop | Crash on startup (bad config, port in use) | Read the first error. Add a backoff before `fork()`. |
| `Too many connections` from MySQL/Postgres | Each worker has its own pool | Pool size × workers must fit the database limit |
| Socket.IO `400 Session ID unknown` | No sticky sessions | `@socket.io/sticky`, or WebSocket-only transport |
| `EADDRINUSE` in workers | Primary or another app holds the port | Only workers call `listen()`. Check `ss -ltnp`. |

## Monitor the Clustered App

A cluster primary hides worker crashes by restarting them. That's good for uptime, but it can also hide a crash loop or a memory leak that keeps killing workers. Log every `exit` event, and watch the app from outside too.

[GreenCrew](https://greencrew.space) runs HTTP uptime checks against your public URL and alerts you on Discord, Slack, or Telegram when it stops responding correctly, with a public status page for your users. Point it at `/health` so a full outage, or a worker pool that's restarting too often to serve traffic, reaches you right away. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Is clustering the same as load balancing?

It's load balancing inside one machine. The primary spreads connections across worker processes on the same server. To spread traffic across several servers, put a load balancer such as [Nginx with upstream failover](/blog/how-to-set-up-nginx-load-balancing-with-failover) in front of them.

### Should I use the cluster module or PM2?

PM2 cluster mode uses the cluster module under the hood and adds restarts, logs, and `pm2 reload` with no extra code. Write your own cluster script if you want zero dependencies, custom worker roles (like a jobs worker), or full control over shutdown behavior.

### Do I need clustering inside Docker or Kubernetes?

Usually not. The common pattern is one Node process per container, with replicas scaled by the orchestrator. Clustering inside a container makes sense when you're limited to one large container per host.

### Does clustering help with a slow database?

No. Clustering adds CPU capacity for your JavaScript. If requests are waiting on the database, more workers mean more connections waiting. Fix the queries, add indexes, or cache results first.

## Conclusion

Scaling a Node.js application with clustering means running one worker per CPU core under a primary process that forks, supervises, and rolls them. Add graceful `SIGTERM` handling, a `SIGUSR2` rolling restart, and a systemd unit or PM2 to keep it running. Move sessions, caches, and rate limits to Redis, and measure the result with autocannon. Put Nginx in front for HTTPS, and let [GreenCrew](https://greencrew.space) watch the public endpoint so outages reach you on Discord, Slack, or Telegram.
