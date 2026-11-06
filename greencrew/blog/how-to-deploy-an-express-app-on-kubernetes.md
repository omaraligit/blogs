# How to Deploy an Express App on Kubernetes (Deployment, Service, Ingress, and Probes)

Author: [greencrew](https://greencrew.space/blog)

Tags: Kubernetes, Node.js, Express, Docker, DevOps

Description: To deploy an Express app on Kubernetes, build and push a Docker image, then run kubectl apply -f with a Deployment (replicas, resource limits, readiness and liveness probes on /health), a ClusterIP Service on port 80 → 3000, and an Ingress that routes your domain to that Service. This guide covers the Dockerfile, registry pull secrets, ConfigMaps and Secrets, rolling updates, kubectl rollout undo, graceful shutdown with preStop, and troubleshooting.

Date: 2026-11-06

**Short answer:** Package your Express app as a container image and push it to a registry such as Docker Hub or GitHub Container Registry. Then write three Kubernetes manifests. The first is a **Deployment** that runs 2–3 replicas of the image with CPU and memory requests and readiness and liveness probes on a `/health` route. The second is a **Service** of type `ClusterIP` that gives those pods one stable address. The third is an **Ingress** that sends `app.example.com` to the Service. Apply them with `kubectl apply -f k8s/`, check them with `kubectl get pods,svc,ingress`, and ship new versions with `kubectl set image` (roll back with `kubectl rollout undo`).

## What Does It Mean to Deploy Express on Kubernetes?

Kubernetes doesn't run your `server.js` directly. It runs **containers**, groups them into **pods**, and keeps the number of pods you asked for running. When a pod crashes or a node dies, Kubernetes starts a replacement. You describe the end state in YAML, and the cluster works to keep it that way.

An Express deployment needs these objects:

| Object | What it does for your Express app | Docs |
| --- | --- | --- |
| Deployment | Runs N identical pods and handles rolling updates | [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) |
| Service | Stable virtual IP and DNS name in front of the pods | [Service](https://kubernetes.io/docs/concepts/services-networking/service/) |
| Ingress | Routes HTTP(S) traffic for a hostname to a Service | [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) |
| ConfigMap | Non-secret settings (`NODE_ENV`, feature flags) | [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Secret | Passwords, API keys, registry credentials | [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) |

If you're still on one VPS, [PM2 and Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx) is simpler. Kubernetes is worth the extra effort when you need several replicas, zero-downtime rollouts, and self-healing across more than one machine.

## Prerequisites

- A running Kubernetes cluster and `kubectl` configured to talk to it (`kubectl get nodes` should list your nodes). For a small self-hosted cluster, see [how to set up a MicroK8s cluster on Ubuntu](/blog/how-to-set-up-a-microk8s-kubernetes-cluster-on-ubuntu).
- Docker (or another image builder) on your workstation or CI runner.
- An account on a container registry: Docker Hub, GitHub Container Registry (GHCR), or your cloud provider's registry.
- Node.js locally for testing. See [how to install Node.js on Ubuntu](/blog/how-to-install-nodejs-on-ubuntu).

## Step 1: Create a Kubernetes-Ready Express App

Kubernetes needs two things from your app. It needs a cheap **health endpoint** to probe, and it needs the app to **exit cleanly on `SIGTERM`**, which Kubernetes sends before it kills a pod.

```bash
mkdir express-k8s && cd express-k8s
npm init -y
npm install express
```

Create `server.js`:

```javascript
const express = require('express');

const app = express();
const port = process.env.PORT || 3000;
let shuttingDown = false;

app.get('/', (req, res) => {
  res.json({ message: process.env.GREETING || 'Hello from Express on Kubernetes', pod: process.env.HOSTNAME });
});

// Used by readiness and liveness probes. Keep it fast and dependency-free.
app.get('/health', (req, res) => {
  if (shuttingDown) return res.status(503).json({ status: 'shutting down' });
  res.json({ status: 'ok' });
});

const server = app.listen(port, '0.0.0.0', () => console.log(`Listening on ${port}`));

process.on('SIGTERM', () => {
  console.log('SIGTERM received, draining connections');
  shuttingDown = true;
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 25000).unref(); // hard stop before terminationGracePeriodSeconds
});
```

The `HOSTNAME` variable holds the pod name, so later you can watch requests spread across replicas. Express's own [health checks and graceful shutdown guide](https://expressjs.com/en/advanced/healthcheck-graceful-shutdown.html) explains the same pattern.

## Step 2: Write a Dockerfile

Keep the image small, run it as a non-root user, and start `node` directly (not `npm start`) so the process receives signals:

```dockerfile
FROM node:lts-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Add a `.dockerignore` with `node_modules`, `.git`, and `.env`. For multi-stage builds, layer caching, and pinning a specific Node major version, see [how to build a Node.js application with Docker](/blog/how-to-build-a-nodejs-application-with-docker).

## Step 3: Build and Push the Image to a Registry

Your cluster nodes pull images from a registry, not from your laptop. Tag every build with a unique version. Avoid shipping `latest` to production, because then you can't tell which code a pod is running.

| Registry | Image name format | Login |
| --- | --- | --- |
| Docker Hub | `docker.io/<user>/express-k8s:1.0.0` | `docker login` |
| GitHub Container Registry | `ghcr.io/<user-or-org>/express-k8s:1.0.0` | `echo $GHCR_TOKEN \| docker login ghcr.io -u <user> --password-stdin` |

```bash
docker build -t ghcr.io/your-user/express-k8s:1.0.0 .
docker push ghcr.io/your-user/express-k8s:1.0.0
```

If your cluster nodes use a different CPU architecture than your machine (for example, you build on Apple Silicon and the servers run amd64), build for the target platform with `docker buildx build --platform linux/amd64 ... --push`.

### Private registries: create an imagePullSecret

If the image is private, the kubelet needs credentials. Create a `docker-registry` Secret in the app's namespace:

```bash
kubectl create namespace express-app

kubectl create secret docker-registry regcred \
  --namespace express-app \
  --docker-server=ghcr.io \
  --docker-username=your-user \
  --docker-password="$GHCR_TOKEN"
```

You'll reference `regcred` in the Deployment. The Kubernetes docs cover this in [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/).

## Step 4: Add a ConfigMap and a Secret

Keep configuration out of the image so the same image can run in staging and production. Create `k8s/config.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: express-config
  namespace: express-app
data:
  NODE_ENV: "production"
  PORT: "3000"
  GREETING: "Hello from Express on Kubernetes"
---
apiVersion: v1
kind: Secret
metadata:
  name: express-secrets
  namespace: express-app
type: Opaque
stringData:
  SESSION_SECRET: "change-me"
  DATABASE_URL: "postgres://app:password@postgres:5432/app"
```

Secrets are only base64-encoded by default, not encrypted. Don't commit real values to Git. Create them with `kubectl create secret generic` from CI, or use a tool like Sealed Secrets or an external secrets manager. If your app needs a database in the cluster, see [how to deploy PostgreSQL on Kubernetes](/blog/how-to-deploy-postgresql-on-kubernetes).

## Step 5: Create the Deployment

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: express-app
  namespace: express-app
  labels:
    app: express-app
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: express-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: express-app
    spec:
      imagePullSecrets:
        - name: regcred          # remove if the image is public
      terminationGracePeriodSeconds: 30
      containers:
        - name: express
          image: ghcr.io/your-user/express-k8s:1.0.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: express-config
            - secretRef:
                name: express-secrets
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 3
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sleep", "5"]
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
```

What each important field does:

| Field | Purpose | Tip |
| --- | --- | --- |
| `replicas` | Number of pods to keep running | Use at least 2 so one pod can restart without downtime |
| `maxUnavailable: 0`, `maxSurge: 1` | Start a new pod before an old one stops during an update | Zero-downtime rollouts as long as the readiness probe is honest |
| `envFrom` | Loads every key of the ConfigMap/Secret as env vars | Pods don't see changes until they restart (`kubectl rollout restart`) |
| `resources.requests` | What the scheduler reserves on a node | Without requests, pods can get packed onto an overloaded node |
| `limits.memory` | Hard memory cap; exceeding it gets the pod `OOMKilled` | Many teams leave the CPU limit unset to avoid throttling |
| `readinessProbe` | Only "ready" pods receive Service traffic | Fails during startup and shutdown, so traffic goes elsewhere |
| `livenessProbe` | Restarts the container if it stops responding | Don't check databases here, or a DB outage restarts every pod |
| `preStop: sleep 5` | Delays SIGTERM so endpoints update before the app stops | Prevents a burst of 502s during rollouts |

The `preStop` sleep matters more than it looks. When a pod starts terminating, Kubernetes removes it from Service endpoints and sends `SIGTERM` at roughly the same time. The ingress controller and kube-proxy may still route a few requests to it for a moment. A short sleep before `SIGTERM` gives them time to catch up, and then your `server.close()` handler drains the requests still in flight. The `sleep` binary must exist in the image (it does in Alpine and Debian images). Newer Kubernetes releases also support a built-in `sleep` action for `preStop`. See [container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/) and [pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination). Keep `preStop` plus your shutdown time under `terminationGracePeriodSeconds`.

Probe options are documented in [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

## Step 6: Expose the Pods with a Service

Pods come and go, and each one gets a new IP. A Service gives them one stable name. Create `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: express-app
  namespace: express-app
spec:
  type: ClusterIP
  selector:
    app: express-app
  ports:
    - name: http
      port: 80
      targetPort: 3000
```

Inside the cluster, other pods can now reach the app at `http://express-app.express-app.svc.cluster.local` (or just `http://express-app` from the same namespace).

### ClusterIP vs NodePort vs LoadBalancer

| Type | Reachable from | When to use it |
| --- | --- | --- |
| `ClusterIP` (default) | Inside the cluster only | Behind an Ingress. This is the normal choice for web apps |
| `NodePort` | `<any-node-ip>:30000–32767` | Quick tests, bare-metal clusters without a load balancer |
| `LoadBalancer` | A public IP from your cloud (or MetalLB on bare metal) | One public IP per Service. Works, but gets expensive with many apps |

For HTTP apps, use `ClusterIP` plus one Ingress controller that shares a single load balancer across every app and handles TLS.

## Step 7: Route a Domain with an Ingress

An Ingress is a set of HTTP routing rules. It does nothing until an **Ingress controller** runs in the cluster. The common choices are NGINX-based controllers, Traefik, and the controllers built into managed platforms.

A note on the controller choice: the Kubernetes project [announced the retirement of the community ingress-nginx controller](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/), so check its current status before you pick it for a new cluster. The upstream direction for new setups is the [Gateway API](https://gateway-api.sigs.k8s.io/), which many controllers already implement. The `Ingress` resource itself is still a stable Kubernetes API, and the manifest below works with any controller once you set `ingressClassName`.

Create `k8s/ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: express-app
  namespace: express-app
spec:
  ingressClassName: nginx        # or "traefik", "public" (MicroK8s), etc.
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: express-app
                port:
                  number: 80
```

Run `kubectl get ingressclass` to see which class names your cluster has. Point a DNS `A` record for `app.example.com` at the controller's external IP. For automatic HTTPS certificates, see [how to secure Kubernetes with cert-manager, Traefik, and Let's Encrypt](/blog/how-to-secure-kubernetes-with-cert-manager-traefik-and-lets-encrypt).

Because Express now sits behind a proxy, add `app.set('trust proxy', 1)` so `req.ip` and `req.protocol` read the forwarded headers. The [Nginx reverse proxy guide](/blog/how-to-set-up-nginx-reverse-proxy) explains those headers.

## Step 8: Apply the Manifests and Verify

```bash
kubectl apply -f k8s/
kubectl -n express-app rollout status deployment/express-app
kubectl -n express-app get pods,svc,ingress -o wide
```

Before DNS is ready, test the Service through a port-forward:

```bash
kubectl -n express-app port-forward svc/express-app 8080:80
curl http://localhost:8080/
curl http://localhost:8080/health
```

Run the first `curl` a few times through the Ingress, and the `pod` field changes as requests spread across replicas.

## Step 9: Roll Out Updates and Roll Back

Build and push a new tag, then update the Deployment:

```bash
docker build -t ghcr.io/your-user/express-k8s:1.1.0 . && docker push ghcr.io/your-user/express-k8s:1.1.0

kubectl -n express-app set image deployment/express-app express=ghcr.io/your-user/express-k8s:1.1.0
kubectl -n express-app rollout status deployment/express-app
```

Kubernetes starts a new pod, waits for its readiness probe to pass, and only then stops an old one. If a new pod never becomes ready, the rollout stops and the old pods keep serving traffic. To go back:

```bash
kubectl -n express-app rollout history deployment/express-app
kubectl -n express-app rollout undo deployment/express-app            # previous revision
kubectl -n express-app rollout undo deployment/express-app --to-revision=2
```

In a GitOps or CI setup, update the image tag in `deployment.yaml` and `kubectl apply` it, so Git stays the source of truth. To scale, run `kubectl scale deployment/express-app --replicas=5` or add a [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/). Scaling with pods replaces the in-process [Node.js clustering](/blog/how-to-scale-nodejs-applications-with-clustering) approach, so run one Node process per container. For packaging all of this as a Helm chart, see [how to scale Node.js and MongoDB on Kubernetes with Helm](/blog/how-to-scale-nodejs-and-mongodb-on-kubernetes-with-helm).

## kubectl Cheat Sheet for Express Deployments

| Task | Command |
| --- | --- |
| List pods with node and IP | `kubectl -n express-app get pods -o wide` |
| Stream logs from all replicas | `kubectl -n express-app logs -l app=express-app -f --prefix` |
| Logs from the previous crashed container | `kubectl -n express-app logs <pod> --previous` |
| Describe a pod (events, probe failures) | `kubectl -n express-app describe pod <pod>` |
| Shell into a pod | `kubectl -n express-app exec -it <pod> -- sh` |
| Check Service endpoints | `kubectl -n express-app get endpointslices -l kubernetes.io/service-name=express-app` |
| Restart pods (reload ConfigMap) | `kubectl -n express-app rollout restart deployment/express-app` |
| Scale replicas | `kubectl -n express-app scale deployment/express-app --replicas=5` |
| Recent namespace events | `kubectl -n express-app get events --sort-by=.lastTimestamp` |
| Resource usage (needs metrics-server) | `kubectl -n express-app top pods` |

The full command list is in the official [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/).

## Troubleshooting Express on Kubernetes

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name or tag, or private image without credentials | Check the tag exists. Create `regcred` in the **same namespace** and reference it in `imagePullSecrets` |
| `exec format error` in logs | Image built for the wrong CPU architecture | Rebuild with `docker buildx build --platform linux/amd64` (or `arm64`) |
| `CrashLoopBackOff` | App throws on startup (missing env var, bad config) | `kubectl logs <pod> --previous`. Check the ConfigMap/Secret keys |
| Pod `Running` but `0/1 READY` | Readiness probe failing (wrong port/path, app listening on `127.0.0.1`) | Listen on `0.0.0.0`. Match `containerPort` and probe port to `PORT` |
| Pods restart every few minutes | Liveness probe too strict or checking a slow dependency | Raise `failureThreshold`/`initialDelaySeconds`, or add a `startupProbe`. Keep `/health` dependency-free |
| `OOMKilled` in `describe pod` | Memory limit lower than the app needs | Raise `limits.memory`, and look for leaks with `kubectl top pods` |
| Ingress returns 404 | Wrong `ingressClassName` or host doesn't match | `kubectl get ingressclass`. Check that the `host` matches the request exactly |
| Ingress returns 502/503 | Service selector doesn't match pod labels, or no ready pods | `kubectl get endpointslices`. Fix labels. See the [502 Bad Gateway guide](/blog/how-to-fix-502-bad-gateway-nginx) |
| Brief 502s during every rollout | Pods stop before endpoints update | Add the `preStop` sleep and `maxUnavailable: 0`. Handle `SIGTERM` |
| ConfigMap change has no effect | Env vars are read only at container start | `kubectl rollout restart deployment/express-app` |

## How to Monitor Your Express App After Deploying

Probes only tell Kubernetes whether a pod is healthy **from inside the cluster**. They can't see an expired certificate, a broken DNS record, a misconfigured Ingress, or a cloud load balancer that stopped forwarding traffic. Your pods can all show `Running` while users get errors.

Add an outside check. [GreenCrew](https://greencrew.space) monitors your public URL over HTTP from outside your infrastructure and alerts you on Discord, Slack, or Telegram when it stops responding correctly. It also gives you a public status page you can share with users during an incident. Point it at `https://app.example.com/health`, the same route your probes use, so a passing check means the whole path from DNS through the Ingress to a pod works. Setup steps are in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Do I need an Ingress, or can I use a LoadBalancer Service?

A `LoadBalancer` Service works fine for a single app. Each one usually provisions its own cloud load balancer and public IP, though, and it doesn't do host-based routing or TLS termination for you. With more than one HTTP app, use one Ingress controller (or a Gateway API implementation) and `ClusterIP` Services behind it.

### Should I use PM2 or Node clustering inside the container?

Usually not. Kubernetes already restarts failed containers and scales by adding pods, so one Node process per container is the standard pattern. It keeps logs, probes, and resource limits simple. Give each pod a CPU request and let replicas or a HorizontalPodAutoscaler handle load.

### What's the difference between readiness and liveness probes?

A failing **readiness** probe removes the pod from Service traffic but leaves it running, which is useful during startup and shutdown. A failing **liveness** probe makes the kubelet restart the container. Start with both on a cheap `/health` route, and never make liveness depend on a database or external API.

### How do I deploy without downtime?

Run at least two replicas, set `maxUnavailable: 0`, use an honest readiness probe, add a short `preStop` sleep, and handle `SIGTERM` by closing the HTTP server. With all five in place, `kubectl set image` replaces pods one at a time with no failed requests.

## Conclusion

Deploying Express on Kubernetes takes a handful of pieces. Build a small image, push it to a registry with a versioned tag, and describe the app with a Deployment, a ClusterIP Service, and an Ingress. Use probes on `/health`, set resource requests, load settings from ConfigMaps and Secrets, and combine a `preStop` sleep with a `SIGTERM` handler so rollouts don't drop requests. Then roll out with `kubectl set image` and roll back with `kubectl rollout undo` if something goes wrong.

Next, add [HTTPS with cert-manager and Let's Encrypt](/blog/how-to-secure-kubernetes-with-cert-manager-traefik-and-lets-encrypt) and set up an outside check with [GreenCrew](https://greencrew.space), so you get Discord, Slack, or Telegram alerts when your app goes down.
