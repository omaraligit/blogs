# How to Deploy PostgreSQL on Kubernetes (StatefulSet, Persistent Volume, Secret, and Backups)

Author: [greencrew](https://greencrew.space/blog)

Tags: Kubernetes, PostgreSQL, StatefulSet, Databases, DevOps

Description: To deploy PostgreSQL on Kubernetes, store the password in a Secret, run the official postgres image in a StatefulSet with a volumeClaimTemplate so data lives on a PersistentVolume, set PGDATA to a subdirectory of the mount, and expose it with a headless Service, then apply everything with kubectl apply -f. This guide covers probes with pg_isready, resource limits, connecting with kubectl exec and port-forward, pg_dump backup CronJobs, restores, upgrades, and when to use Helm, an operator, or a managed database instead.

Date: 2026-11-06

**Short answer:** Create a namespace and a `Secret` that holds `POSTGRES_PASSWORD`. Then deploy the official `postgres` image as a **StatefulSet**, not a Deployment, with a `volumeClaimTemplates` entry so each pod gets its own PersistentVolumeClaim. Set `PGDATA=/var/lib/postgresql/data/pgdata` so `initdb` doesn't fail on a non-empty mount. Add `pg_isready` readiness and liveness probes, and expose the database inside the cluster with a headless Service (`clusterIP: None`). Apps connect to `postgres.database.svc.cluster.local:5432`. Back it up with a `pg_dump` CronJob, and use an operator such as CloudNativePG when you need replication and failover.

## What Is the Right Way to Run PostgreSQL on Kubernetes?

PostgreSQL is a stateful service: its data has to survive pod restarts, rescheduling, and node failures. Kubernetes handles this with three building blocks:

| Building block | What it does for PostgreSQL |
| --- | --- |
| **StatefulSet** | Gives each pod a stable name (`postgres-0`) and its own volume, and starts or stops pods in order |
| **PersistentVolumeClaim (PVC)** | Requests durable storage from a StorageClass. The data outlives the pod |
| **Headless Service** | Gives the pod a stable DNS name (`postgres-0.postgres.database.svc.cluster.local`) |
| **Secret** | Stores the superuser password outside your manifests and images |
| **ConfigMap** | Holds non-secret settings such as the database name or a custom `postgresql.conf` |

### Why a StatefulSet and Not a Deployment?

A Deployment treats pods as interchangeable. During a rolling update it can start a new pod before the old one stops, and two PostgreSQL processes writing to the same volume will corrupt your data. With a `ReadWriteOnce` volume, the new pod can also get stuck in `ContainerCreating` while the old pod still holds the volume. A StatefulSet stops `postgres-0` before it starts its replacement, keeps the same PVC attached to the same pod identity, and supports `volumeClaimTemplates`. Read the [StatefulSet documentation](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) for the full set of guarantees.

## Prerequisites

- A working Kubernetes cluster and `kubectl` configured for it. For a single Ubuntu server, see [how to set up a MicroK8s Kubernetes cluster on Ubuntu](/blog/how-to-set-up-a-microk8s-kubernetes-cluster-on-ubuntu).
- A default **StorageClass** that can provision volumes. Check with `kubectl get storageclass`. On MicroK8s, enable `hostpath-storage`. On a cloud cluster, the provider's block storage class works.
- Basic familiarity with `kubectl apply`.

## Step 1: Create a Namespace

Keep the database in its own namespace so you can apply RBAC, quotas, and network policies to it separately:

```bash
kubectl create namespace database
```

## Step 2: Store Credentials in a Secret

Never put the password in the StatefulSet YAML. Create a Secret from the command line so it doesn't end up in Git:

```bash
kubectl -n database create secret generic postgres-secret \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -base64 24)"
```

By default, Secrets are only base64-encoded, not encrypted. On production clusters, turn on [encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) or use an external secret manager.

## Step 3: Add a ConfigMap for Non-Secret Settings

```yaml
# postgres-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: database
data:
  POSTGRES_DB: appdb
  PGDATA: /var/lib/postgresql/data/pgdata
```

### The PGDATA Subdirectory Gotcha

Many storage provisioners create a `lost+found` directory at the root of a new volume. If PostgreSQL's data directory is the mount point itself, `initdb` sees a non-empty directory and the pod crash-loops with `initdb: error: directory "/var/lib/postgresql/data" exists but is not empty`. Pointing `PGDATA` at a **subdirectory** (`.../data/pgdata`) avoids this. The official image's [Docker Hub page](https://hub.docker.com/_/postgres) documents the `PGDATA` variable. Newer major versions of the image have changed the default data path, so check the page for the tag you use and match your `mountPath` to it.

## Step 4: Create the Services

You need two Services. The headless Service gives the StatefulSet pods stable DNS names. The regular ClusterIP Service gives apps a single, simple hostname.

```yaml
# postgres-services.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: database
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: database
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

Don't use `NodePort` or `LoadBalancer` for the database. PostgreSQL should only be reachable from inside the cluster.

## Step 5: Deploy the PostgreSQL StatefulSet

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: postgres
          image: postgres:17   # pin a major version; never use :latest for a database
          ports:
            - containerPort: 5432
              name: postgres
          envFrom:
            - secretRef:
                name: postgres-secret
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              memory: 1Gi
          readinessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\" -h 127.0.0.1"]
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["sh", "-c", "pg_isready -U \"$POSTGRES_USER\" -d \"$POSTGRES_DB\" -h 127.0.0.1"]
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 6
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
        # storageClassName: your-class   # omit to use the cluster default
```

Apply everything and watch the pod start:

```bash
kubectl apply -f postgres-config.yaml -f postgres-services.yaml -f postgres-statefulset.yaml
kubectl -n database get pods -w
kubectl -n database get pvc
```

The PVC is named `data-postgres-0`. If you delete the StatefulSet, the PVC stays. That protects your data, but you have to delete the PVC yourself if you want it gone.

### Key StatefulSet Settings

| Setting | Why it matters |
| --- | --- |
| `serviceName` | Must match the headless Service, otherwise the pods don't get stable DNS names |
| `volumeClaimTemplates` | Creates one PVC per pod (`data-postgres-0`) |
| `terminationGracePeriodSeconds: 60` | Gives PostgreSQL time to shut down cleanly on SIGTERM |
| `resources.requests` | Lets the scheduler reserve CPU and memory. Without them, Postgres gets evicted first under pressure |
| `limits.memory` only | A CPU limit causes throttling. A memory limit stops a runaway query from taking down the node |
| `pg_isready` probes | Readiness gates traffic until Postgres accepts connections. Liveness restarts a hung server |

Keep the liveness probe lenient (`failureThreshold: 6`). Crash recovery after an unclean shutdown can take a while, and an aggressive liveness probe will kill Postgres in the middle of recovery. You can read more about probes in [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

## Step 6: Connect to PostgreSQL

**From inside the pod**, with `kubectl exec`:

```bash
kubectl -n database exec -it postgres-0 -- psql -U app -d appdb
```

**From your laptop**, with a port-forward. This needs no public exposure:

```bash
kubectl -n database port-forward svc/postgres 5432:5432
psql "postgresql://app:<password>@127.0.0.1:5432/appdb"
```

**From an app in the cluster**, use the Service DNS name. Read the password from the same Secret if the app lives in the `database` namespace. Otherwise, copy it into a Secret in the app's namespace, because pods can't reference Secrets across namespaces:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secret
        key: POSTGRES_PASSWORD
  - name: DATABASE_URL
    value: postgresql://app:$(DB_PASSWORD)@postgres.database.svc.cluster.local:5432/appdb
```

Keep `DB_PASSWORD` above `DATABASE_URL`, because a `$(VAR)` reference only expands to variables defined earlier in the list. If the password contains URL-special characters, pass separate host, user, and password variables instead of a URL. For a complete app deployment, see [how to deploy an Express app on Kubernetes](/blog/how-to-deploy-an-express-app-on-kubernetes). If you use an ORM, the connection settings work the same as in [using Sequelize with Node.js](/blog/how-to-use-sequelize-with-nodejs-and-mysql), with `dialect: 'postgres'`.

## Step 7: Back Up PostgreSQL with a pg_dump CronJob

A PersistentVolume is not a backup. A deleted namespace, a bad migration, or a storage failure can take the volume with it. Run a nightly [`pg_dump`](https://www.postgresql.org/docs/current/app-pgdump.html) to a separate volume, then copy the dumps off-cluster:

```yaml
# postgres-backup.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backups
  namespace: database
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 20Gi
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: database
spec:
  schedule: "0 3 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:17   # same major version as the server
              envFrom:
                - secretRef:
                    name: postgres-secret
                - configMapRef:
                    name: postgres-config
              command:
                - sh
                - -c
                - |
                  set -e
                  export PGPASSWORD="$POSTGRES_PASSWORD"
                  FILE=/backups/appdb-$(date +%Y%m%d-%H%M%S).dump
                  pg_dump -h postgres -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Fc -f "$FILE"
                  find /backups -name '*.dump' -mtime +7 -delete
                  echo "Backup written to $FILE"
              volumeMounts:
                - name: backups
                  mountPath: /backups
          volumes:
            - name: backups
              persistentVolumeClaim:
                claimName: postgres-backups
```

Test it right away instead of waiting until 3 a.m.:

```bash
kubectl apply -f postgres-backup.yaml
kubectl -n database create job --from=cronjob/postgres-backup backup-test
kubectl -n database logs job/backup-test
```

`-Fc` writes PostgreSQL's custom format, which is compressed and can be restored selectively. For point-in-time recovery, you'd need WAL archiving. That's where operators shine (see below).

## Step 8: Restore a Backup

Copy the dump into the database pod and run [`pg_restore`](https://www.postgresql.org/docs/current/app-pgrestore.html):

```bash
kubectl -n database cp ./appdb-20261106-030000.dump postgres-0:/tmp/restore.dump
kubectl -n database exec -it postgres-0 -- \
  pg_restore -U app -d appdb --clean --if-exists /tmp/restore.dump
```

Practice a restore into a scratch database every so often. A backup you've never restored is only a hope.

## Upgrading PostgreSQL on Kubernetes

Minor upgrades within the same major version are safe. Bump the tag (or re-pull it) and let the StatefulSet restart the pod. **Major upgrades are different.** The on-disk format changes between major versions, so pointing a newer image at an older data directory fails at startup. You can upgrade either way:

1. Take a `pg_dump`, deploy a new StatefulSet with the new major version and a fresh PVC, restore into it, then switch the Service selector.
2. Run [`pg_upgrade`](https://www.postgresql.org/docs/current/pgupgrade.html), which needs the old and new binaries side by side. That's awkward with single-version images, and it's one reason teams adopt an operator.

## Raw Manifests vs Helm Chart vs Operator vs Managed Database

| Approach | Best for | Replication and failover | Backups / PITR | Effort |
| --- | --- | --- | --- | --- |
| **Raw manifests** (this guide) | Learning, dev, small single-instance apps | No | Manual `pg_dump` | Low to start, high to grow |
| **Helm chart** | Quick, repeatable installs with sane defaults | Some charts offer read replicas | Chart-dependent | Low |
| **Operator** ([CloudNativePG](https://cloudnative-pg.io/documentation/current/), Zalando, Crunchy PGO) | Production on Kubernetes | Yes, automated | Built-in WAL archiving to object storage | Medium |
| **Managed database** (your cloud's PostgreSQL service) | Teams that don't want to operate databases | Yes | Yes, provider-managed | Lowest operational load |

A reasonable rule: use raw manifests to learn and for low-stakes workloads, and an operator once real users depend on the data. If you're installing other stateful services with Helm, see [how to scale Node.js and MongoDB on Kubernetes with Helm](/blog/how-to-scale-nodejs-and-mongodb-on-kubernetes-with-helm) for the same workflow.

## Troubleshooting PostgreSQL on Kubernetes

| Symptom | Cause | Fix |
| --- | --- | --- |
| Pod stuck in `Pending` | PVC can't bind. No default StorageClass | `kubectl get pvc,storageclass`. Set a default class or add `storageClassName` |
| `initdb: directory ... exists but is not empty` | `lost+found` at the volume root | Set `PGDATA` to a subdirectory of the mount |
| `CrashLoopBackOff` after an image bump | Major version changed against an old data directory | Roll back the tag, then dump and restore into the new version |
| Password change has no effect | `POSTGRES_PASSWORD` only applies on first init | `ALTER USER app PASSWORD '...'` in psql, then update the Secret |
| App can't resolve `postgres` | Different namespace | Use `postgres.database.svc.cluster.local` |
| Pod restarts during startup | Liveness probe too aggressive during recovery | Raise `initialDelaySeconds` and `failureThreshold`, or add a `startupProbe` |
| `OOMKilled` | Memory limit below `shared_buffers` + work memory | Raise `limits.memory` or tune `postgresql.conf` |
| Data gone after `kubectl delete` | PVC or namespace deleted | Restore from backup. Use a StorageClass with `reclaimPolicy: Retain` for databases |

Use `kubectl -n database describe pod postgres-0` and `kubectl -n database logs postgres-0 --previous` to diagnose most of these problems.

## Monitor the Apps That Depend on PostgreSQL

Kubernetes probes only tell Kubernetes whether the pod is healthy. They won't tell *you* that your site is returning errors because the database is down, out of disk, or refusing connections. The most useful check is from your users' side. Expose a `/health` endpoint in your app that runs a cheap query such as `SELECT 1`, so a broken database surfaces as a failing HTTP response.

Then point [GreenCrew](https://greencrew.space) at that public URL. It checks the endpoint over HTTP from outside your cluster, sends you a Discord, Slack, or Telegram alert when it fails, and gives your users a public status page. See [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram) for the setup. If the ingress in front of your app returns 502s when the backend can't reach the database, [how to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx) explains how to trace them.

## Frequently Asked Questions

### Is it safe to run PostgreSQL on Kubernetes in production?

Yes, provided you use durable storage, tested backups, and ideally an operator that handles replication and failover. A single-replica StatefulSet works for small apps, but there's downtime whenever the node fails or the pod is rescheduled.

### Can I set replicas: 3 to get high availability?

No. Raising `replicas` in this StatefulSet creates three *independent* databases with separate volumes and no replication between them. Streaming replication and failover need extra configuration, which is exactly what operators like CloudNativePG automate.

### Should I use a Deployment if I only have one replica?

Use a StatefulSet anyway. Even with one replica, a Deployment's rolling update can briefly run two pods against the same volume, and it can't create a PVC per pod from a template.

### How do I expose PostgreSQL outside the cluster?

Avoid it if you can. Use `kubectl port-forward` for admin access. If an external service really needs it, use a `LoadBalancer` restricted by source IP ranges and require TLS in `pg_hba.conf`. Web traffic is different: [secure it with cert-manager, Traefik, and Let's Encrypt](/blog/how-to-secure-kubernetes-with-cert-manager-traefik-and-lets-encrypt).

## Conclusion

Running PostgreSQL on Kubernetes comes down to a few deliberate choices. Use a StatefulSet with `volumeClaimTemplates` for stable storage, keep credentials in a Secret, point `PGDATA` at a subdirectory, add `pg_isready` probes, and expose the database only through internal Services. Add a `pg_dump` CronJob on day one, test your restores, and plan major upgrades as a dump and restore. When the data starts to matter, move to an operator or a managed database for replication and point-in-time recovery. Finally, monitor the app in front of the database with [GreenCrew](https://greencrew.space) so a database failure reaches you on Discord, Slack, or Telegram before your users notice.
