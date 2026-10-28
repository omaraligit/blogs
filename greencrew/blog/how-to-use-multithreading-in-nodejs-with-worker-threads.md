# How to Use Multithreading in Node.js (worker_threads, Worker Pools, and When to Use Them)

Author: [greencrew](https://greencrew.space/blog)

Tags: Node.js, worker_threads, Performance, JavaScript, DevOps

Description: To use multithreading in Node.js, import Worker from node:worker_threads, move CPU-heavy code into a separate worker file, and exchange data with postMessage and the message event so the main event loop stays free. This guide covers a blocking-vs-worker example, a promise wrapper, worker pools with Piscina, sharing memory, the libuv thread pool, and when to use cluster instead.

Date: 2026-10-28

**Short answer:** Node.js runs your JavaScript on a single thread, so one CPU-heavy function (hashing, image processing, parsing a huge JSON file) blocks every other request. To run that work in parallel, create a worker with `new Worker('./worker.js', { workerData })` from the built-in `node:worker_threads` module. In the worker, do the work and send the result back with `parentPort.postMessage(result)`. Wrap it in a Promise so callers can `await` it, and use a worker pool (for example the Piscina library) so you don't spawn a new thread per request. Worker threads help with CPU-bound work. For I/O, Node's async APIs are already non-blocking.

## What Is Multithreading in Node.js?

Node.js has one main thread that runs the **event loop**. It's very good at I/O (network, disk, databases) because it hands that work to the operating system and moves on. It's bad at long CPU work, because while a function computes, the event loop can't serve any other request.

`worker_threads` (stable since Node.js 12) lets you run JavaScript on additional threads inside the same process. Each worker has its own V8 instance, event loop, and memory heap, and communicates with the main thread by passing messages.

| Workload | Blocks the event loop? | Use worker threads? |
| --- | --- | --- |
| HTTP calls, database queries, `fs.promises` | No | No, async I/O already handles it |
| `JSON.parse` of a multi-MB payload | Yes | Often |
| Password hashing in pure JS, crypto loops | Yes | Yes (or use async `crypto` APIs) |
| Image resizing, PDF generation, compression in JS | Yes | Yes |
| Machine learning inference in JS | Yes | Yes |
| A simple CRUD API | No | No, scale with [clustering](/blog/how-to-scale-nodejs-applications-with-clustering) instead |

### Worker Threads vs Cluster vs Child Processes

| | `worker_threads` | `cluster` | `child_process` |
| --- | --- | --- | --- |
| Unit | Thread inside one process | Separate Node processes | Any separate process |
| Memory | Separate heaps, can share `SharedArrayBuffer` | Fully separate | Fully separate |
| Startup cost | Lower | Higher | Highest |
| Best for | Offloading CPU tasks | Using all cores for HTTP traffic | Running other programs (ffmpeg, Python) |
| Shares a server port | No | Yes | No |

The [Node.js worker_threads docs](https://nodejs.org/api/worker_threads.html) are the full reference.

## Step 1: See the Problem with a Blocking Endpoint

```javascript
// server.js
const http = require('node:http');

function fibonacci(n) {
  return n < 2 ? n : fibonacci(n - 1) + fibonacci(n - 2);
}

http.createServer((req, res) => {
  if (req.url === '/blocking') {
    const result = fibonacci(42);          // seconds of pure CPU
    return res.end(`Result: ${result}\n`);
  }
  res.end('Fast response\n');
}).listen(3000, () => console.log('http://localhost:3000'));
```

Run `node server.js`. In one terminal call `curl localhost:3000/blocking`, and while it's computing, call `curl localhost:3000/` in another. The "fast" endpoint waits until the Fibonacci call finishes. Every user of your API pays for one user's CPU work.

## Step 2: Move the Work into a Worker Thread

Create `fib-worker.js`:

```javascript
// fib-worker.js
const { parentPort, workerData } = require('node:worker_threads');

function fibonacci(n) {
  return n < 2 ? n : fibonacci(n - 1) + fibonacci(n - 2);
}

parentPort.postMessage(fibonacci(workerData.n));
```

Update the server to start a worker for the heavy route:

```javascript
// server.js
const http = require('node:http');
const path = require('node:path');
const { Worker } = require('node:worker_threads');

function runFibonacci(n) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(path.join(__dirname, 'fib-worker.js'), { workerData: { n } });
    worker.once('message', resolve);
    worker.once('error', reject);
    worker.once('exit', (code) => {
      if (code !== 0) reject(new Error(`Worker stopped with exit code ${code}`));
    });
  });
}

http.createServer(async (req, res) => {
  if (req.url === '/non-blocking') {
    try {
      const result = await runFibonacci(42);
      return res.end(`Result: ${result}\n`);
    } catch (err) {
      res.statusCode = 500;
      return res.end('Worker failed\n');
    }
  }
  res.end('Fast response\n');
}).listen(3000);
```

Repeat the test. `/` now answers immediately while `/non-blocking` computes on another thread.

### worker_threads API Cheat Sheet

| API | Where | Purpose |
| --- | --- | --- |
| `new Worker(file, { workerData })` | Main | Start a thread and pass initial data (cloned) |
| `worker.postMessage(value, [transferList])` | Main | Send more messages to the worker |
| `worker.on('message' / 'error' / 'exit')` | Main | Receive results and failures |
| `worker.terminate()` | Main | Stop the thread (returns a Promise) |
| `parentPort.postMessage(value)` | Worker | Send a result back |
| `parentPort.on('message', fn)` | Worker | Receive messages from the main thread |
| `workerData` | Worker | The data passed at startup |
| `isMainThread` | Both | `true` on the main thread, for single-file scripts |
| `resourceLimits` | Option | Cap a worker's heap size (`maxOldGenerationSizeMb`) |

## Step 3: Use a Worker Pool Instead of One Thread per Request

Creating a worker per request costs time and memory, and under load you can create hundreds of threads. A **pool** keeps a fixed number of workers alive and queues tasks for them. [Piscina](https://github.com/piscinajs/piscina) is a widely used pool library maintained by Node.js contributors:

```bash
npm install piscina
```

```javascript
// fib-task.js: export the task function
function fibonacci(n) {
  return n < 2 ? n : fibonacci(n - 1) + fibonacci(n - 2);
}
module.exports = ({ n }) => fibonacci(n);
```

```javascript
// server.js
const http = require('node:http');
const path = require('node:path');
const os = require('node:os');
const Piscina = require('piscina');

const pool = new Piscina({
  filename: path.resolve(__dirname, 'fib-task.js'),
  maxThreads: Math.max(1, os.availableParallelism() - 1), // leave a core for the event loop
});

http.createServer(async (req, res) => {
  if (req.url === '/pooled') {
    const result = await pool.run({ n: 40 });
    return res.end(`Result: ${result}\n`);
  }
  res.end('Fast response\n');
}).listen(3000);
```

Size the pool around the number of CPU cores. More threads than cores doesn't make CPU-bound work faster. It only adds context switching. `os.availableParallelism()` is available in current Node.js LTS releases. On older versions, use `os.cpus().length`.

## Step 4: Pass Data Efficiently

Messages are copied with the structured clone algorithm. That's fine for small objects but expensive for large buffers. You have two faster options:

**Transfer ownership** of an `ArrayBuffer` (zero-copy, and the sender can no longer use it):

```javascript
const buffer = new ArrayBuffer(50 * 1024 * 1024);
worker.postMessage({ buffer }, [buffer]);
```

**Share memory** with a `SharedArrayBuffer` and coordinate with `Atomics`:

```javascript
// main.js
const { Worker } = require('node:worker_threads');
const shared = new SharedArrayBuffer(4);
const counter = new Int32Array(shared);

const workers = Array.from({ length: 4 }, () =>
  new Worker('./count-worker.js', { workerData: { shared } })
);
Promise.all(workers.map((w) => new Promise((r) => w.on('exit', r))))
  .then(() => console.log('Total:', Atomics.load(counter, 0)));
```

```javascript
// count-worker.js
const { workerData } = require('node:worker_threads');
const counter = new Int32Array(workerData.shared);
for (let i = 0; i < 1_000_000; i++) Atomics.add(counter, 0, 1);
```

Without `Atomics`, concurrent writes to shared memory race and lose updates. Share memory only when copying is too slow.

## Step 5: Know What the libuv Thread Pool Already Does

Node.js already uses background threads for some built-in work, through libuv's thread pool: file system calls, `dns.lookup`, `zlib`, and async `crypto` functions such as `crypto.pbkdf2` and `crypto.scrypt`. Its default size is 4 threads.

```bash
UV_THREADPOOL_SIZE=8 node server.js
```

| If your bottleneck is... | Do this |
| --- | --- |
| Many concurrent `fs` or `crypto.scrypt` calls queueing | Raise `UV_THREADPOOL_SIZE` (set it before Node starts) |
| Your own JavaScript loops | Worker threads |
| Synchronous APIs (`fs.readFileSync`, `crypto.pbkdf2Sync`) in request handlers | Switch to the async versions first |

Always try the async version of a built-in API before reaching for a worker.

## Step 6: Measure Before and After

Don't guess. Measure with a load generator such as [autocannon](https://github.com/mcollina/autocannon):

```bash
npx autocannon -c 50 -d 20 http://localhost:3000/
```

Run it against the fast route while another terminal hammers the CPU-heavy route. Compare latency percentiles with the blocking version and with the pooled version. For a lower-level view, `node --cpu-prof server.js` writes a CPU profile you can open in Chrome DevTools.

## Troubleshooting Worker Threads

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Cannot find module './worker.js'` | Relative path resolved from the current working directory | Use `path.join(__dirname, 'worker.js')` |
| `DataCloneError: ... could not be cloned` | Sending functions, class instances with methods, or sockets | Send plain data. Recreate objects in the worker. |
| Server slower after adding workers | One new worker per request, or more threads than cores | Use a fixed-size pool |
| Memory grows with each request | Workers never exit | `worker.terminate()`, or use a pool |
| Worker crash takes down nothing, request hangs | No `error`/`exit` handler | Reject the Promise on `error` and non-zero `exit` |
| `ERR_WORKER_OUT_OF_MEMORY` | Worker heap limit hit | Set `resourceLimits.maxOldGenerationSizeMb`, or process data in chunks |
| Shared counter values wrong | Race condition | Use `Atomics` operations |
| ES module worker fails to load | Mixing CommonJS and ESM | Use a `.mjs` worker file, or `new Worker(new URL('./worker.mjs', import.meta.url))` |

## Monitor Response Times in Production

The bug that worker threads fix, a blocked event loop, rarely crashes your app. Instead every request gets slow at the same time, or a load balancer starts timing out. Process managers won't restart a process that's busy but alive.

An outside check catches this. [GreenCrew](https://greencrew.space) runs HTTP uptime checks against your endpoints and alerts you on Discord, Slack, or Telegram when they stop responding correctly, and gives you a public status page for your users. Point it at a lightweight `/health` route that runs on the main thread. If the event loop is blocked, the health check fails too. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Is Node.js single-threaded?

Your JavaScript runs on one main thread per process. Node.js itself uses extra threads internally (the libuv pool, garbage collection), and `worker_threads` lets you run your own JavaScript on more threads.

### Should I use worker threads or cluster?

Use `worker_threads` to offload specific CPU-heavy tasks while keeping the server responsive. Use `cluster` (or PM2 cluster mode) to run several copies of a whole HTTP server across CPU cores. Many production apps use both. Clustering is covered in [how to scale Node.js applications with clustering](/blog/how-to-scale-nodejs-applications-with-clustering).

### Do worker threads make database queries faster?

No. Database queries are I/O, and Node already handles them asynchronously without blocking. Workers help only when your own code is busy computing.

### How many worker threads should I create?

For CPU-bound work, roughly one per available core, minus one for the main thread. Use a pool so the number stays fixed under load, then measure with a load test.

## Conclusion

Multithreading in Node.js means moving CPU-heavy JavaScript off the event loop with `node:worker_threads`. Wrap workers in Promises, reuse them through a pool like Piscina, transfer or share large buffers instead of copying them, and remember that async built-ins already use libuv's thread pool. Measure before and after with a load test. In production, run a [clustered](/blog/how-to-scale-nodejs-applications-with-clustering) app behind [PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx), and let [GreenCrew](https://greencrew.space) alert you on Discord, Slack, or Telegram if your endpoints ever stop responding.
