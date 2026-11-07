# How to Set Up a MicroK8s Kubernetes Cluster on Ubuntu (snap install, add-node, Addons, and HA)

Author: [greencrew](https://greencrew.space/blog)

Tags: Kubernetes, MicroK8s, Ubuntu, Linux, DevOps

Description: To set up a MicroK8s Kubernetes cluster on Ubuntu, run sudo snap install microk8s --classic on every node, add your user to the microk8s group, run microk8s add-node on the first node, and paste the printed microk8s join command on each other node. This guide covers addons (dns, hostpath-storage, ingress, metallb), firewall ports, kubectl access with microk8s config, a test deployment, upgrades, removing nodes, and high availability with 3+ nodes.

Date: 2026-11-06

**Short answer:** Install MicroK8s on each Ubuntu server with `sudo snap install microk8s --classic` (pin a release with `--channel=<version>/stable`), then add your user to the `microk8s` group and wait for `microk8s status --wait-ready`. On the first node, run `microk8s add-node`. It prints a `microk8s join <ip>:25000/<token>` command, which you run on the second node. Repeat for every node (`add-node` generates a new token each time). With three or more control plane nodes, MicroK8s turns on high availability automatically. Enable `dns`, `hostpath-storage`, and `ingress`, and copy `microk8s config` to your laptop so you can use plain `kubectl`.

## What Is MicroK8s?

MicroK8s is a lightweight, CNCF-conformant Kubernetes distribution from Canonical, packaged as a single snap. It bundles the API server, scheduler, controller manager, kubelet, containerd, and a Calico network, all managed by one `microk8s` command. Features like DNS, storage, ingress, and load balancers are **addons** that you switch on with `microk8s enable`.

It runs fine on a single machine, but the real value is how easy clustering is. There's no kubeadm init, no manual certificate copying, and no external etcd. MicroK8s uses **dqlite** (distributed SQLite) as its datastore and replicates it automatically once you have three control plane nodes.

Official references: [MicroK8s documentation](https://microk8s.io/docs), [Kubernetes documentation](https://kubernetes.io/docs/home/).

### MicroK8s vs k3s vs kubeadm

| | MicroK8s | k3s | kubeadm |
| --- | --- | --- | --- |
| Install | `snap install microk8s --classic` | `curl -sfL https://get.k3s.io \| sh -` | Install container runtime, kubelet, kubeadm, then `kubeadm init` |
| Datastore | dqlite (auto HA at 3 nodes) | SQLite, embedded etcd, or external DB | etcd (you design HA) |
| Default CNI | Calico | Flannel | None, you install one |
| Default ingress | Addon (NGINX ingress controller) | Traefik built in | None |
| Add a node | `microk8s add-node` + `join` | Server token + agent install | `kubeadm join` with token and CA hash |
| Upgrades | `snap refresh` per node | Re-run installer or system-upgrade-controller | `kubeadm upgrade` per node |
| Best for | Ubuntu fleets, labs, edge, small prod | Edge, ARM, minimal footprint | Full control, "vanilla" clusters |

If you're already on Ubuntu and want a working multi-node cluster in minutes, MicroK8s is the shortest path.

## Prerequisites

- **Ubuntu 24.04 LTS** servers (22.04 LTS works the same way). Three nodes is ideal for HA. One is fine for testing.
- At least 2 vCPU and 4 GB RAM per node for real workloads. MicroK8s can start on less, but your apps need room.
- Static private IPs, with each node able to reach the others.
- A user with `sudo`.
- **Unique hostnames that resolve on every node.** This is the most common join failure, so set it up now.

Set hostnames and add them to `/etc/hosts` on **every** node:

```bash
sudo hostnamectl set-hostname k8s-1   # k8s-2, k8s-3 on the others
```

```text
# /etc/hosts
10.0.0.11  k8s-1
10.0.0.12  k8s-2
10.0.0.13  k8s-3
```

## Step 1: Install MicroK8s with Snap

On every node:

```bash
sudo apt update && sudo apt upgrade -y
snap info microk8s | sed -n '/channels:/,$p'   # list available release channels
sudo snap install microk8s --classic --channel=<version>/stable
```

Replace `<version>` with the Kubernetes minor release you want from the `snap info` output. Use the **same channel on every node**. If you leave out `--channel`, snap installs the default stable track, which works, but pinning makes upgrades deliberate instead of surprising.

`--classic` is required because MicroK8s needs full host access (networking, mounts, containerd), which strict snap confinement doesn't allow.

## Step 2: Add Your User to the microk8s Group

By default, `microk8s` commands need `sudo`. Add yourself to the group so you don't:

```bash
sudo usermod -a -G microk8s $USER
mkdir -p ~/.kube
sudo chown -f -R $USER ~/.kube
newgrp microk8s     # or log out and back in
```

Then wait for the node to be ready:

```bash
microk8s status --wait-ready
microk8s kubectl get nodes
```

You should see one node in `Ready` state. Optional quality-of-life alias:

```bash
sudo snap alias microk8s.kubectl kubectl
```

## Step 3: Open the Firewall Ports Between Nodes

If you use UFW, allow cluster traffic **only from your private network**, never from the internet. These are the ports from the [MicroK8s services and ports reference](https://microk8s.io/docs/services-and-ports) that matter for a basic multi-node cluster:

| Port | Protocol | Used by | Purpose |
| --- | --- | --- | --- |
| 16443 | TCP | kube-apiserver | Kubernetes API (kubectl, nodes) |
| 10250 | TCP | kubelet | API server to kubelet (logs, exec) |
| 25000 | TCP | cluster-agent | `microk8s join` and cluster management |
| 19001 | TCP | dqlite | Datastore replication between control plane nodes |
| 4789 | UDP | Calico VXLAN | Pod-to-pod traffic across nodes |

```bash
sudo ufw allow from 10.0.0.0/24 to any port 16443 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 10250 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 25000 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 19001 proto tcp
sudo ufw allow from 10.0.0.0/24 to any port 4789 proto udp
sudo ufw allow 80,443/tcp     # only on nodes that receive public ingress traffic
```

Check the linked page for the full list (kube-proxy, metrics, and others) if you lock things down further. Many cloud providers also have a separate security group or VPC firewall. Open the same ports there.

## Step 4: Join Nodes to the Cluster (add-node and join)

On the **first** node (`k8s-1`):

```bash
microk8s add-node
```

The output looks like this:

```text
From the node you wish to join to this cluster, run the following:
microk8s join 10.0.0.11:25000/3f8a...c21e/9b0d...

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 10.0.0.11:25000/3f8a...c21e/9b0d... --worker
```

Copy the `join` command and run it on `k8s-2`. **Each token works once and expires quickly**, so run `microk8s add-node` again on `k8s-1` before you join `k8s-3`.

| Join type | Command | Runs control plane? | Counts toward HA? |
| --- | --- | --- | --- |
| Control plane node | `microk8s join <ip>:25000/<token>` | Yes | Yes |
| Worker node | `microk8s join <ip>:25000/<token> --worker` | No, kubelet + containerd only | No |

Verify from any control plane node:

```bash
microk8s kubectl get nodes -o wide
```

```text
NAME    STATUS   ROLES    AGE   VERSION
k8s-1   Ready    <none>   20m   v1.x.x
k8s-2   Ready    <none>   4m    v1.x.x
k8s-3   Ready    <none>   1m    v1.x.x
```

## Step 5: Confirm High Availability (3+ Nodes)

When the third control plane node joins, MicroK8s promotes the dqlite datastore to a replicated, highly available setup. You don't have to configure anything:

```bash
microk8s status
```

```text
microk8s is running
high-availability: yes
  datastore master nodes: 10.0.0.11:19001 10.0.0.12:19001 10.0.0.13:19001
  datastore standby nodes: none
```

With three voters, the cluster survives one control plane node going down. Keep an **odd number** of control plane nodes (3 or 5). Add capacity beyond that with `--worker` nodes. Details: [MicroK8s high availability](https://microk8s.io/docs/high-availability).

## Step 6: Enable the Essential Addons

Addons are cluster-wide, so enable them once, from any node:

```bash
microk8s enable dns
microk8s enable hostpath-storage
microk8s enable ingress
microk8s enable metallb:10.0.0.200-10.0.0.220
```

| Addon | What it gives you | Notes |
| --- | --- | --- |
| `dns` | CoreDNS for `service.namespace.svc.cluster.local` names | Needed by almost every app |
| `hostpath-storage` | A default StorageClass (`microk8s-hostpath`) for PVCs | Data stays on one node's disk, so it's fine for labs. Use a networked storage addon or CSI driver for real HA storage |
| `ingress` | NGINX ingress controller on ports 80/443 of each node | Check the class name with `kubectl get ingressclass` |
| `metallb` | `LoadBalancer` services get real IPs from your range | The range must be free IPs on the nodes' LAN |
| `dashboard` | Kubernetes web UI | Access with `microk8s dashboard-proxy`, never expose it publicly |
| `cert-manager` | Automatic TLS certificates | See [cert-manager with Let's Encrypt](/blog/how-to-secure-kubernetes-with-cert-manager-traefik-and-lets-encrypt) |

List everything available with `microk8s status` or the [addons reference](https://microk8s.io/docs/addons).

## Step 7: Use kubectl from Your Laptop (microk8s config)

Typing `microk8s kubectl` works, but you'll want your normal `kubectl`, Helm, and tools like k9s from your workstation. Export the kubeconfig:

```bash
# on k8s-1
microk8s config > microk8s.kubeconfig
```

Copy it to your laptop and merge it, or use it directly:

```bash
scp user@k8s-1:microk8s.kubeconfig ~/.kube/microk8s
export KUBECONFIG=~/.kube/microk8s
kubectl get nodes
```

The file points to `https://<node-ip>:16443` and contains admin credentials, so treat it like a root password. If the API is only reachable on a private network, connect over a VPN or SSH tunnel instead of opening 16443 to the internet.

## Step 8: Deploy a Test Application

Create a deployment, a service, and an ingress to prove the whole path works:

```yaml
# hello.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  replicas: 3
  selector:
    matchLabels: { app: hello }
  template:
    metadata:
      labels: { app: hello }
    spec:
      containers:
        - name: hello
          image: nginxdemos/hello:plain-text
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet: { path: /, port: 80 }
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector: { app: hello }
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
spec:
  ingressClassName: public
  rules:
    - host: hello.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: hello, port: { number: 80 } }
```

```bash
kubectl apply -f hello.yaml
kubectl get pods -o wide            # pods should spread across nodes
curl -H "Host: hello.example.com" http://10.0.0.11/
```

Run the `curl` a few times. The `Server name` in the response changes as the ingress balances across pods. If `ingressClassName: public` doesn't match, use the name shown by `kubectl get ingressclass`.

Ready for a real app? Follow [how to deploy an Express app on Kubernetes](/blog/how-to-deploy-an-express-app-on-kubernetes) (build the image first with [how to build a Node.js application with Docker](/blog/how-to-build-a-nodejs-application-with-docker)), add a database with [how to deploy PostgreSQL on Kubernetes](/blog/how-to-deploy-postgresql-on-kubernetes), or package everything with Helm in [how to scale Node.js and MongoDB on Kubernetes with Helm](/blog/how-to-scale-nodejs-and-mongodb-on-kubernetes-with-helm).

## Step 9: Upgrade MicroK8s Safely

Snap refreshes stay within your pinned channel, which means patch releases only. To move to a new Kubernetes minor version, upgrade **one node at a time** and **one minor version at a time**:

```bash
kubectl drain k8s-1 --ignore-daemonsets --delete-emptydir-data
# on k8s-1
sudo snap refresh microk8s --channel=<next-version>/stable
microk8s status --wait-ready
kubectl uncordon k8s-1
```

Repeat for each node and wait for `Ready` before you move on. To control *when* automatic patch refreshes happen, use snap's refresh window, for example `sudo snap set system refresh.timer=sat,03:00`. See the [MicroK8s upgrade guide](https://microk8s.io/docs/upgrading).

## Step 10: Remove a Node from the Cluster

Drain the node first so its workloads move elsewhere, then leave the cluster:

```bash
kubectl drain k8s-3 --ignore-daemonsets --delete-emptydir-data
# on k8s-3
microk8s leave
# on a remaining control plane node
microk8s remove-node k8s-3
```

If the node is dead and can't run `leave`, use `microk8s remove-node k8s-3 --force` from a healthy node. Dropping from 3 to 2 control plane nodes turns HA off, so replace the node soon.

## Troubleshooting MicroK8s Clusters

Start every investigation with `microk8s inspect`. It checks services and writes a tarball of logs you can dig through.

| Symptom | Cause | Fix |
| --- | --- | --- |
| `microk8s join` hangs or times out | Port 25000 blocked, or the token expired | Open 25000/tcp between nodes, then run `add-node` again for a fresh token |
| Node joins but stays `NotReady` | Calico can't reach other nodes (4789/udp blocked) or hostnames don't resolve | Open 4789/udp, add all hostnames to `/etc/hosts` on every node |
| `Insufficient permissions to access MicroK8s` | User isn't in the `microk8s` group yet | `sudo usermod -a -G microk8s $USER`, then `newgrp microk8s` |
| Pods can't resolve service names | `dns` addon not enabled | `microk8s enable dns` |
| PVC stuck in `Pending` | No default StorageClass | `microk8s enable hostpath-storage` |
| Ingress returns 404 | Wrong `ingressClassName` or `Host` header | Check `kubectl get ingressclass`, curl with the right `Host` |
| `LoadBalancer` service stuck `<pending>` | MetalLB not enabled, or range exhausted | `microk8s enable metallb:<range>` with free LAN IPs |
| `high-availability: no` with 3 nodes | One node joined with `--worker` | Remove it and rejoin without `--worker` |
| kubectl from laptop: connection refused on 16443 | API port firewalled, or kubeconfig points to a private IP | Use a VPN/SSH tunnel, or allow 16443 from your admin IP only |

When the app behind the ingress fails, the NGINX ingress controller returns 502/503 errors much like a standalone Nginx does. [How to fix 502 Bad Gateway in Nginx](/blog/how-to-fix-502-bad-gateway-nginx) explains the same upstream logic.

## Monitor Your Cluster's Public Endpoints

`kubectl get pods` tells you what Kubernetes thinks. It can't tell you whether a user on the internet can actually load your site. DNS, the cloud firewall, MetalLB, the ingress controller, or an expired certificate can each break things while every pod shows `Running`.

Add an outside check. [GreenCrew](https://greencrew.space) monitors your public URLs over HTTP and alerts you on Discord, Slack, or Telegram when they stop responding correctly. It also gives you a public status page to share with your users. Point it at a lightweight health route exposed through the same ingress your traffic uses, so a failure anywhere in the path shows up. Setup is covered in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Is MicroK8s good enough for production?

Yes, for many teams. It's a conformant Kubernetes distribution, it supports HA with three or more nodes, and Canonical publishes long-term support channels. The main production gaps are usually ones you fill yourself: replicated storage (hostpath isn't), backups, and monitoring.

### How many nodes do I need for a MicroK8s HA cluster?

At least three control plane nodes. dqlite needs a majority to keep quorum, so three nodes tolerate one failure and five tolerate two. Additional capacity can come from `--worker` nodes, which don't take part in quorum.

### Can I mix Ubuntu 22.04 and 24.04 nodes?

MicroK8s runs from a snap, so the Kubernetes components are the same on both releases. What has to match is the **MicroK8s channel**. Keep every node on the same `--channel` to avoid version skew between nodes.

### Should I use the MicroK8s ingress or put Nginx in front?

Use the ingress addon for routing inside the cluster. Putting a standalone Nginx in front only makes sense if you already run one as an edge proxy for several backends. That pattern is covered in [how to set up an Nginx reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy). For HTTPS, prefer cert-manager inside the cluster over running Certbot on the host.

## Conclusion

A multi-node MicroK8s cluster on Ubuntu takes four commands per node: `snap install microk8s --classic`, add yourself to the `microk8s` group, `microk8s add-node` on the first node, and `microk8s join` on the rest. Open ports 16443, 10250, 25000, 19001, and 4789/udp on your private network. Join three control plane nodes so HA turns on automatically, and enable `dns`, `hostpath-storage`, `ingress`, and `metallb`. Then export `microk8s config` and manage everything with plain `kubectl`. Upgrade one node and one minor version at a time.

Once your apps are live, add [GreenCrew](https://greencrew.space) to watch them from the outside. You'll get Discord, Slack, or Telegram alerts and a public status page, so you know about downtime before your users tell you.
