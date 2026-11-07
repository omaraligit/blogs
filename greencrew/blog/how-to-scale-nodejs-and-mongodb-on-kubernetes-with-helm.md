# How to Scale a Node.js App with MongoDB on Kubernetes Using Helm (Replica Set, Custom Chart, and HPA)

Author: [greencrew](https://greencrew.space/blog)

Tags: Kubernetes, Helm, Node.js, MongoDB, DevOps

Description: To scale a Node.js app with MongoDB on Kubernetes, install MongoDB as a replica set with Helm (architecture=replicaset, replicaCount=3), point your app at the headless-service connection string stored in a Secret, package the app as its own chart with helm create, deploy it with helm upgrade --install, and scale with replicaCount or a HorizontalPodAutoscaler. This guide covers values.yaml, the connection string, templating, autoscaling, rollbacks, and troubleshooting.

Date: 2026-11-06

**Short answer:** Install Helm, then deploy MongoDB from a chart with `architecture: replicaset` and `replicaCount: 3`, so you get a StatefulSet with one persistent volume per member. Your Node.js app connects with a replica set URI that lists each pod through the chart's headless Service (`mongo-mongodb-0.mongo-mongodb-headless...:27017`) and includes `replicaSet=rs0`. Keep that URI in a Kubernetes Secret. Scaffold a chart for the app with `helm create`, inject the Secret as an environment variable, and ship it with `helm upgrade --install`. You scale the stateless app with `--set replicaCount=5` or a HorizontalPodAutoscaler, and you scale MongoDB by raising its replica count. If a release breaks, `helm rollback` returns it to the previous revision.

## What Is Helm, and Why Use It for Node.js and MongoDB?

[Helm](https://helm.sh/docs/) is the package manager for Kubernetes. A **chart** is a folder of templated Kubernetes manifests plus a `values.yaml` file of defaults. A **release** is one installed instance of a chart. Each `helm upgrade` creates a new numbered revision, and you can roll back to any of them.

A Node.js + MongoDB stack has two very different halves:

| Component | Kubernetes object | State | How it scales |
| --- | --- | --- | --- |
| Node.js API | Deployment | Stateless | Add pods freely (`replicaCount`, HPA) |
| MongoDB | StatefulSet + headless Service | Stateful (PersistentVolumes) | Add replica set members; scale reads, not writes |

Helm lets you manage both with the same workflow. For MongoDB you use a maintained community chart. For your app you write a small chart of your own.

If you haven't containerized the app yet, start with [how to build a Node.js application with Docker](/blog/how-to-build-a-nodejs-application-with-docker). If you want a plain-manifest version first, without Helm, read [how to deploy an Express app on Kubernetes](/blog/how-to-deploy-an-express-app-on-kubernetes).

## Prerequisites

- A Kubernetes cluster with a default StorageClass. A local or single-node cluster works fine. See [how to set up a MicroK8s Kubernetes cluster on Ubuntu](/blog/how-to-set-up-a-microk8s-kubernetes-cluster-on-ubuntu).
- `kubectl` configured for that cluster (`kubectl get nodes` returns `Ready`)
- A Node.js app image pushed to a registry your cluster can pull from
- [metrics-server](https://github.com/kubernetes-sigs/metrics-server) if you want autoscaling (Step 8)

## Step 1: Install Helm

Helm is a single binary. On Ubuntu, the snap is the quickest option:

```bash
sudo snap install helm --classic
helm version
```

For other platforms or the official install script, use the [Helm installation guide](https://helm.sh/docs/intro/install/). Helm uses the same kubeconfig as `kubectl`, so no further setup is needed.

Create a namespace for the stack:

```bash
kubectl create namespace shop
```

## Step 2: Choose a MongoDB Chart or Operator

The Bitnami MongoDB chart has long been the default way to run MongoDB with Helm, and the rest of this guide uses its value names. Note that in 2025 Bitnami changed how it distributes its free container images and catalog. Before you install, check the [Bitnami charts repository](https://github.com/bitnami/charts/tree/main/bitnami/mongodb) for the current image source and any values you need to set, and don't copy old image tags from blog posts.

The main alternative is the [MongoDB Community Kubernetes Operator](https://github.com/mongodb/mongodb-kubernetes-operator) (MongoDB has since been folding it into its unified [MongoDB Controllers for Kubernetes](https://www.mongodb.com/docs/kubernetes/current/) project). With an operator, you install a controller once and then declare a replica set as a custom resource.

| Option | You manage | Best for |
| --- | --- | --- |
| Bitnami Helm chart | One `values.yaml`, `helm upgrade` | Quick, familiar Helm workflow |
| MongoDB operator | Operator + `MongoDBCommunity` resource | Long-lived clusters, operator-driven upgrades |
| Managed MongoDB (Atlas, etc.) | Just the connection string | Teams that don't want to run databases |

Whichever you pick, the app side (Steps 4–8) is the same: it only needs a replica set connection string.

## Step 3: Install MongoDB as a Replica Set with Helm

Create `mongo-values.yaml`:

```yaml
architecture: replicaset
replicaCount: 3
replicaSetName: rs0

auth:
  enabled: true
  rootPassword: "change-me-root"
  usernames:
    - shopapp
  passwords:
    - "change-me-app"
  databases:
    - shop

# Not needed with an odd number of data members
arbiter:
  enabled: false

persistence:
  enabled: true
  size: 10Gi

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    memory: 1Gi
```

Install it as a release named `mongo` (the chart is published as an OCI artifact):

```bash
helm install mongo oci://registry-1.docker.io/bitnamicharts/mongodb \
  --namespace shop \
  -f mongo-values.yaml
```

Watch the members come up one at a time:

```bash
kubectl -n shop get pods -l app.kubernetes.io/name=mongodb -w
kubectl -n shop get pvc
```

The chart creates a StatefulSet (`mongo-mongodb`), one PersistentVolumeClaim per member, and a **headless Service** (`mongo-mongodb-headless`). A headless Service gives every pod a stable DNS name, which is exactly what a replica set needs.

Keep real passwords out of Git. In production, create a Secret yourself and reference it with the chart's `auth.existingSecret` value instead of putting passwords in the values file.

### MongoDB chart values reference

| Value | Purpose | Example |
| --- | --- | --- |
| `architecture` | `standalone` or `replicaset` | `replicaset` |
| `replicaCount` | Number of data-bearing members | `3` |
| `replicaSetName` | Name used in `replicaSet=` in the URI | `rs0` |
| `auth.rootPassword` | Admin password | from Secret |
| `auth.usernames` / `passwords` / `databases` | App user created at init | `shopapp` / `shop` |
| `auth.existingSecret` | Use your own Secret for passwords | `mongo-auth` |
| `arbiter.enabled` | Voting member with no data | `false` with 3 members |
| `persistence.size` | PVC size per member | `10Gi` |

## Step 4: Build the Replica Set Connection String

Pod DNS names behind a headless Service follow this pattern:

```
<pod>.<headless-service>.<namespace>.svc.cluster.local
```

So the connection string for this release is:

```
mongodb://shopapp:change-me-app@mongo-mongodb-0.mongo-mongodb-headless.shop.svc.cluster.local:27017,mongo-mongodb-1.mongo-mongodb-headless.shop.svc.cluster.local:27017,mongo-mongodb-2.mongo-mongodb-headless.shop.svc.cluster.local:27017/shop?replicaSet=rs0&authSource=shop
```

Listing every member and adding `replicaSet=rs0` lets the MongoDB driver find the current primary and fail over automatically. See MongoDB's [connection string reference](https://www.mongodb.com/docs/manual/reference/connection-string/). Use `authSource=shop` because the chart creates the app user in the `shop` database.

Store it in a Secret:

```bash
kubectl -n shop create secret generic shop-api-mongo \
  --from-literal=MONGODB_URI='mongodb://shopapp:change-me-app@mongo-mongodb-0.mongo-mongodb-headless.shop.svc.cluster.local:27017,mongo-mongodb-1.mongo-mongodb-headless.shop.svc.cluster.local:27017,mongo-mongodb-2.mongo-mongodb-headless.shop.svc.cluster.local:27017/shop?replicaSet=rs0&authSource=shop'
```

Test the app user from inside the first MongoDB pod before you deploy the app:

```bash
kubectl -n shop exec -it mongo-mongodb-0 -- \
  mongosh "mongodb://shopapp:change-me-app@localhost:27017/shop?authSource=shop" --eval 'rs.status().members.map(m => m.stateStr)'
```

You should see one `PRIMARY` and two `SECONDARY` members.

## Step 5: Make the Node.js App Kubernetes-Friendly

Read the URI from the environment, expose a health route, and shut down cleanly on `SIGTERM`. A minimal Express + Mongoose `server.js`:

```javascript
const express = require('express');
const mongoose = require('mongoose');

const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => res.send('shop-api ok'));

// Readiness: only receive traffic when connected to MongoDB
app.get('/health', (req, res) => {
  const ok = mongoose.connection.readyState === 1;
  res.status(ok ? 200 : 503).json({ mongo: ok ? 'up' : 'down' });
});

mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected to MongoDB'))
  .catch((err) => { console.error(err); process.exit(1); });

const server = app.listen(port, '0.0.0.0', () => console.log(`Listening on ${port}`));

process.on('SIGTERM', () => {
  server.close(async () => {
    await mongoose.disconnect();
    process.exit(0);
  });
});
```

Build and push the image, for example `registry.example.com/shop-api:1.0.0`. Because the app is stateless, any number of copies can run against the same replica set.

## Step 6: Create a Helm Chart for the Node.js App

Scaffold a chart:

```bash
helm create shop-api
```

That generates:

| File | What it does |
| --- | --- |
| `Chart.yaml` | Chart name, chart `version`, and `appVersion` |
| `values.yaml` | Defaults: image, replicas, service, probes, resources |
| `templates/deployment.yaml` | The Deployment, templated from values |
| `templates/service.yaml` | ClusterIP Service in front of the pods |
| `templates/hpa.yaml` | HorizontalPodAutoscaler (off by default) |
| `templates/ingress.yaml` | Optional Ingress |
| `templates/_helpers.tpl` | Name and label helpers |

Edit `shop-api/values.yaml`. These are the parts you change:

```yaml
replicaCount: 2

image:
  repository: registry.example.com/shop-api
  pullPolicy: IfNotPresent
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 3000

# Name of the Secret holding MONGODB_URI
mongoSecretName: shop-api-mongo

livenessProbe:
  httpGet:
    path: /
    port: http
readinessProbe:
  httpGet:
    path: /health
    port: http

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

In `templates/deployment.yaml`, find the container spec and add the environment block below `imagePullPolicy`. The scaffold already wires `containerPort` to `.Values.service.port` and names it `http`:

```yaml
          env:
            - name: PORT
              value: {{ .Values.service.port | quote }}
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.mongoSecretName }}
                  key: MONGODB_URI
```

The scaffold's Deployment only sets `replicas: {{ .Values.replicaCount }}` when `autoscaling.enabled` is false, so the HPA and Helm don't fight over the replica count. Check the rendered output before you install:

```bash
helm lint ./shop-api
helm template shop-api ./shop-api -n shop | less
```

See the [Helm chart template guide](https://helm.sh/docs/chart_template_guide/) for the template syntax.

## Step 7: Deploy with helm upgrade --install

Use one command for both the first install and every later update:

```bash
helm upgrade --install shop-api ./shop-api \
  --namespace shop \
  --set image.tag=1.0.0 \
  --wait
```

`--install` creates the release if it doesn't exist, and `--wait` blocks until the pods pass their readiness probes. Verify:

```bash
helm -n shop list
kubectl -n shop get deploy,pods,svc -l app.kubernetes.io/instance=shop-api
kubectl -n shop port-forward svc/shop-api 3000:3000
curl localhost:3000/health
```

To expose the app publicly, enable the chart's Ingress and add TLS. See [how to secure Kubernetes with cert-manager, Traefik, and Let's Encrypt](/blog/how-to-secure-kubernetes-with-cert-manager-traefik-and-lets-encrypt).

## Step 8: Scale the Node.js App (Manually and with an HPA)

**Manual scaling** goes through Helm so the value is recorded in the release:

```bash
helm upgrade shop-api ./shop-api -n shop --reuse-values --set replicaCount=5
```

Avoid `kubectl scale` on Helm-managed Deployments. The next `helm upgrade` will reset the count to whatever is in your values.

**Autoscaling** uses a [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), which needs metrics-server and CPU **requests** on the pods (we set `100m`). On MicroK8s, run `microk8s enable metrics-server`. Elsewhere, follow the metrics-server README. Then:

```bash
kubectl top pods -n shop      # confirms metrics work
helm upgrade shop-api ./shop-api -n shop --reuse-values \
  --set autoscaling.enabled=true \
  --set autoscaling.minReplicas=2 \
  --set autoscaling.maxReplicas=10 \
  --set autoscaling.targetCPUUtilizationPercentage=70
kubectl -n shop get hpa -w
```

The HPA adds pods when average CPU passes 70% of the request and removes them when load drops. Each pod is a single Node.js process. Kubernetes scales by adding pods, so you usually don't need the in-process `cluster` module from [how to scale Node.js applications with clustering](/blog/how-to-scale-nodejs-applications-with-clustering).

## Step 9: Scale MongoDB

Scaling MongoDB is different from scaling the app. A replica set has **one primary** that accepts all writes, and the extra members provide redundancy and optional read capacity.

```bash
helm upgrade mongo oci://registry-1.docker.io/bitnamicharts/mongodb \
  -n shop -f mongo-values.yaml --set replicaCount=5
```

| Goal | What to do |
| --- | --- |
| Survive one node failure | 3 data members (majority = 2) |
| Survive two node failures | 5 data members |
| Offload reads | Add `readPreference=secondaryPreferred` to the URI for read-tolerant queries |
| Scale writes | Bigger primary (resources), better indexes, or [sharding](https://www.mongodb.com/docs/manual/sharding/) |

After adding members, update the `MONGODB_URI` Secret to list them (drivers discover new members on their own, but a complete seed list helps on cold starts). Then restart the app with `kubectl -n shop rollout restart deploy/shop-api`. Keep an odd number of voting members, and spread them across nodes with pod anti-affinity if your cluster has more than one node.

## Step 10: Roll Back a Bad Release

Every upgrade is a revision:

```bash
helm -n shop history shop-api
helm -n shop rollback shop-api 3
```

The rollback re-applies revision 3's manifests and values. It doesn't touch MongoDB data, so if a release ran a destructive migration, restore from backup rather than relying on a rollback. Back up with `mongodump` or volume snapshots before major upgrades.

## Helm Commands Cheat Sheet

| Command | What it does |
| --- | --- |
| `helm create <name>` | Scaffold a new chart |
| `helm lint ./chart` | Check a chart for errors |
| `helm template <rel> ./chart` | Render manifests without installing |
| `helm install <rel> <chart> -f values.yaml` | Install a release |
| `helm upgrade --install <rel> <chart>` | Install or upgrade (idempotent) |
| `helm upgrade <rel> <chart> --reuse-values --set k=v` | Change one value |
| `helm get values <rel>` | Show the values a release is using |
| `helm list -n <ns>` | List releases |
| `helm history <rel>` | List revisions |
| `helm rollback <rel> <rev>` | Return to a previous revision |
| `helm uninstall <rel>` | Remove a release (PVCs from StatefulSets are kept) |

The full list is in the [Helm commands docs](https://helm.sh/docs/helm/).

## Troubleshooting Node.js and MongoDB on Kubernetes

| Symptom | Cause | Fix |
| --- | --- | --- |
| MongoDB pods stuck `Pending` | No default StorageClass, PVC can't bind | `kubectl get sc`; enable a storage addon or set `persistence.storageClass` |
| App logs `MongoServerSelectionError` | Wrong host, missing `replicaSet=`, or members not ready | Test DNS with `nslookup mongo-mongodb-0.mongo-mongodb-headless.shop`; check `rs.status()` |
| `Authentication failed` | Wrong `authSource` or password changed after first install | Use `authSource=shop`; passwords are set at init, so change them inside MongoDB, not just in values |
| App pods `0/1 Ready` | `/health` returns 503 until Mongo connects | Check the Secret name/key and the app logs |
| HPA shows `<unknown>/70%` | metrics-server missing or no CPU requests | Install metrics-server; set `resources.requests.cpu` |
| Replica count snaps back after upgrade | Used `kubectl scale` on a Helm release | Scale with `helm upgrade --set replicaCount=` or enable the HPA |
| `ImagePullBackOff` | Image tag or registry auth wrong | `kubectl describe pod`; add `imagePullSecrets` in values |
| 502 from the ingress/proxy | No ready endpoints behind the Service | `kubectl get endpoints shop-api`; see [how to fix 502 Bad Gateway](/blog/how-to-fix-502-bad-gateway-nginx) |

## Monitor the Public Endpoint

Kubernetes probes restart unhealthy pods, but they run inside the cluster. They won't tell you when the ingress, DNS, certificate, or the whole cluster is unreachable. Add an outside check on the same `/health` route. Because that route returns 503 when MongoDB is down, one HTTP check covers both the API and the database.

[GreenCrew](https://greencrew.space) monitors your URL over HTTP and alerts you on Discord, Slack, or Telegram when it stops responding correctly. It also gives you a public status page to share with users during an incident. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Can I scale MongoDB writes by adding replica set members?

No. All writes go to the single primary, so extra members add redundancy and can serve reads, but they don't add write capacity. To scale writes, give the primary more resources, tune indexes, or move to a [sharded cluster](https://www.mongodb.com/docs/manual/sharding/).

### Why does the connection string list every pod instead of one Service?

The driver needs to reach each member directly to discover which one is primary and to fail over. A normal ClusterIP Service would load-balance across members and send writes to secondaries. The headless Service gives each pod a stable DNS name for the seed list.

### Should I put MongoDB in the same Helm chart as my app?

Usually not. Keep them as separate releases. The database has a different lifecycle (rare, careful upgrades, persistent data), while the app is redeployed often. Separate releases also mean `helm rollback shop-api` can never touch the database. If you need a relational database instead, see [how to deploy PostgreSQL on Kubernetes](/blog/how-to-deploy-postgresql-on-kubernetes).

### Do I still need Nginx or PM2?

Not inside the cluster. Kubernetes restarts crashed pods (the job PM2 does on a VM) and the Service load-balances across them. You still need an ingress controller such as Traefik or Nginx Ingress for public traffic. For a single-server setup without Kubernetes, [PM2 with Nginx](/blog/how-to-deploy-nodejs-app-with-pm2-and-nginx) is simpler.

## Conclusion

Running Node.js and MongoDB on Kubernetes with Helm comes down to treating the two halves differently. Install MongoDB as a three-member replica set from a chart or operator, and connect to it with a headless-service URI that includes `replicaSet=rs0` and lives in a Secret. Package the stateless API as its own chart with `helm create`, deploy it with `helm upgrade --install`, and scale it with `replicaCount` or an HPA. Use `helm rollback` when a release goes wrong. Then add an outside HTTP check on `/health` with [GreenCrew](https://greencrew.space), so you get a Discord, Slack, or Telegram alert the moment your API or database stops answering.
