# How to Secure Kubernetes with cert-manager, Traefik, and Let's Encrypt (Automatic HTTPS for Ingress)

Author: [greencrew](https://greencrew.space/blog)

Tags: Kubernetes, cert-manager, Traefik, Let's Encrypt, TLS, DevOps

Description: To get automatic HTTPS in Kubernetes, install Traefik as your ingress controller, install cert-manager with helm install cert-manager jetstack/cert-manager --set crds.enabled=true, create a Let's Encrypt ClusterIssuer that uses the HTTP-01 solver with ingressClassName: traefik, then add the cert-manager.io/cluster-issuer annotation and a tls block to your Ingress. This guide covers staging vs production issuers, IngressRoute with a Certificate resource, HTTP to HTTPS redirects, DNS-01 wildcard certificates, and debugging failed challenges.

Date: 2026-11-06

**Short answer:** Install Traefik with its Helm chart (k3s already ships it), then install cert-manager with `helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --set crds.enabled=true`. Create a `ClusterIssuer` that points at the Let's Encrypt ACME server and uses an `http01` solver with `ingressClassName: traefik`. On each Ingress, add `cert-manager.io/cluster-issuer: letsencrypt-prod` and a `tls` block with a host and a `secretName`. cert-manager solves the challenge, stores the certificate in that Secret, Traefik serves it, and cert-manager renews it before it expires.

## What Is cert-manager (and Why Use It with Traefik)?

[cert-manager](https://cert-manager.io/docs/) is a Kubernetes controller that requests, stores, and renews X.509 certificates. It adds custom resources (`Issuer`, `ClusterIssuer`, `Certificate`, and others) and talks to certificate authorities such as Let's Encrypt over the [ACME protocol](https://letsencrypt.org/how-it-works/). Certificates end up as ordinary `kubernetes.io/tls` Secrets, so any ingress controller can use them.

[Traefik](https://doc.traefik.io/traefik/) is a reverse proxy and ingress controller. It watches Kubernetes `Ingress` objects and its own `IngressRoute` CRDs, and it reloads routes without restarting. Together they give you the Kubernetes version of what Certbot does on a single server. For the single-server version, see [how to install Let's Encrypt SSL on Nginx with Certbot](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot).

| Component | Job |
| --- | --- |
| Traefik | Accepts traffic on ports 80/443 and routes it to Services |
| cert-manager | Requests certificates and renews them before they expire |
| ClusterIssuer | Cluster-wide config that tells cert-manager which CA and challenge type to use |
| Certificate | The certificate you want: DNS names, Secret name, and issuer |
| TLS Secret | Holds `tls.crt` and `tls.key`, which Traefik reads |

### cert-manager vs Certbot vs Traefik Built-in ACME

Traefik can request Let's Encrypt certificates by itself, so why add cert-manager? Here's how the three options compare:

| | cert-manager | Certbot | Traefik built-in ACME |
| --- | --- | --- | --- |
| Where it runs | Kubernetes controller | Single host (cron/systemd timer) | Inside Traefik |
| Certificate storage | Kubernetes Secrets | Files in `/etc/letsencrypt` | `acme.json` file or Traefik's storage |
| Works with any ingress controller | Yes | Not in Kubernetes | Traefik only |
| Multiple Traefik replicas | Yes (shared Secrets) | N/A | Hard without shared storage; Traefik docs point to Traefik Enterprise or cert-manager for HA |
| Wildcards via DNS-01 | Yes, many DNS providers | Yes, with plugins | Yes |
| Declarative (GitOps-friendly) | Yes, YAML resources | No | Partly |
| Best for | Production clusters | VMs and bare-metal Nginx | Single-replica Traefik, quick setups |

If you run more than one Traefik replica, or you might switch ingress controllers later, cert-manager is the safer choice.

## Prerequisites

- A Kubernetes cluster with `kubectl` access and [Helm](https://helm.sh/docs/intro/install/) installed. For a small self-hosted cluster, see [how to set up a MicroK8s Kubernetes cluster on Ubuntu](/blog/how-to-set-up-a-microk8s-kubernetes-cluster-on-ubuntu).
- A domain whose A/AAAA record points to the external IP of your Traefik Service (a LoadBalancer IP or your node's IP).
- Ports 80 and 443 reachable from the internet. HTTP-01 validation always hits port 80.
- An app running behind a Service. If you don't have one, follow [how to deploy an Express app on Kubernetes](/blog/how-to-deploy-an-express-app-on-kubernetes).

## Step 1: Install Traefik with Helm

**On k3s, skip this step.** k3s installs Traefik by default in `kube-system`. You customize it with a `HelmChartConfig` resource instead of reinstalling it.

Everywhere else, install the official chart ([Traefik Helm docs](https://doc.traefik.io/traefik/getting-started/install-traefik/#use-the-helm-chart)):

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik \
  --namespace traefik --create-namespace
```

Check that it's running and get the external IP:

```bash
kubectl get pods -n traefik
kubectl get svc -n traefik traefik
```

The chart creates an `IngressClass` named `traefik`. Confirm it exists, because your issuer and Ingress both reference it:

```bash
kubectl get ingressclass
```

Point your DNS record at the `EXTERNAL-IP` shown for the Service. If it stays `<pending>` on bare metal, you need a load balancer implementation such as MetalLB, or the MicroK8s `metallb` addon.

## Step 2: Install cert-manager with Helm

Follow the [cert-manager Helm installation docs](https://cert-manager.io/docs/installation/helm/). Current charts install the CRDs when you set `crds.enabled=true`. Older guides use `installCRDs=true`, which was the previous name for the same option.

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

Wait for the three deployments (controller, webhook, and cainjector) to become ready:

```bash
kubectl get pods -n cert-manager
kubectl wait --for=condition=Available deployment --all -n cert-manager --timeout=120s
```

If you create an issuer before the webhook is ready, you'll get `failed calling webhook "webhook.cert-manager.io"`. Wait a few seconds and apply it again.

## Step 3: Create Staging and Production ClusterIssuers

Let's Encrypt has a [staging environment](https://letsencrypt.org/docs/staging-environment/) with much looser limits. Its certificates aren't trusted by browsers, which is fine for testing. Always test against staging first so a misconfiguration doesn't burn through your production [rate limits](https://letsencrypt.org/docs/rate-limits/).

Save this as `cluster-issuers.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

```bash
kubectl apply -f cluster-issuers.yaml
kubectl get clusterissuer
```

Both issuers should show `READY True`. That means cert-manager registered an ACME account and stored its private key in the Secret named in `privateKeySecretRef`.

| Field | Meaning |
| --- | --- |
| `server` | ACME directory URL (staging or production) |
| `email` | Contact address for the ACME account |
| `privateKeySecretRef` | Secret where cert-manager stores the ACME account key |
| `solvers[].http01.ingress.ingressClassName` | Ingress class for the temporary challenge Ingress |

Use a `ClusterIssuer` when one issuer should serve every namespace. Use a namespaced `Issuer` when teams need separate accounts or credentials. See the [ACME issuer docs](https://cert-manager.io/docs/configuration/acme/).

## Step 4: Add TLS to an Ingress with the cert-manager Annotation

This is the simplest pattern. You annotate a normal Kubernetes [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls), and cert-manager's ingress-shim creates the `Certificate` for you.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - app.example.com
      secretName: myapp-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get certificate -n default -w
```

While the challenge is in progress, cert-manager creates a short-lived solver Pod and Ingress that serve `/.well-known/acme-challenge/<token>`. Once Let's Encrypt validates the token, `READY` turns `True` and the `myapp-tls` Secret holds the certificate.

Run `curl -vI https://app.example.com` and check that the issuer is a staging CA. If it is, change the annotation to `letsencrypt-prod`. Then delete the Secret so cert-manager requests a fresh, trusted certificate:

```bash
kubectl annotate ingress myapp cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite
kubectl delete secret myapp-tls
```

## Step 5 (Alternative): Use a Certificate Resource with a Traefik IngressRoute

If you use Traefik's own `IngressRoute` CRD (for middlewares, TCP routes, or weighted services), cert-manager doesn't watch it. Create the `Certificate` explicitly and reference its Secret instead ([Certificate docs](https://cert-manager.io/docs/usage/certificate/)):

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: default
spec:
  secretName: myapp-tls
  dnsNames:
    - app.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: myapp
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`app.example.com`)
      kind: Rule
      services:
        - name: myapp
          port: 80
  tls:
    secretName: myapp-tls
```

The HTTP-01 solver still creates a standard Ingress with class `traefik`, so Traefik's Kubernetes Ingress provider must stay enabled. It is enabled by default in the Helm chart. Check the [Traefik IngressRoute reference](https://doc.traefik.io/traefik/routing/providers/kubernetes-crd/) for the `apiVersion` your Traefik version uses.

## Step 6: Redirect HTTP to HTTPS in Traefik

You have two options.

**Option A: a redirect Middleware (recommended).** Create it once, then attach it to the Ingresses that should redirect:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: default
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

Attach it with an annotation in the form `<namespace>-<name>@kubernetescrd`:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
```

cert-manager's challenge Ingress is a separate router with a longer, more specific path. It keeps answering on plain HTTP, so renewals keep working.

**Option B: entrypoint-level redirection.** This redirects every request on port 80 to HTTPS. Set it with Traefik's static configuration flags through the Helm chart:

```yaml
# traefik-values.yaml
additionalArguments:
  - "--entryPoints.web.http.redirections.entryPoint.to=websecure"
  - "--entryPoints.web.http.redirections.entryPoint.scheme=https"
```

```bash
helm upgrade traefik traefik/traefik -n traefik -f traefik-values.yaml
```

Let's Encrypt [follows redirects](https://letsencrypt.org/docs/challenge-types/#http-01-challenge) during HTTP-01 validation, so this usually works. If challenges start failing after you enable it, go back to Option A. Newer Traefik chart versions also expose redirects as chart values, so check the [entrypoint docs](https://doc.traefik.io/traefik/routing/entrypoints/#redirection) for your version.

Once HTTPS works, add HSTS and other response headers with a `headers` Middleware. The header choices are the same as in [how to add security headers in Nginx](/blog/how-to-add-security-headers-in-nginx).

## Step 7: Get a Wildcard Certificate with DNS-01

HTTP-01 can't issue wildcard certificates (`*.example.com`). For wildcards, or for clusters that aren't reachable from the internet, use the [DNS-01 solver](https://cert-manager.io/docs/configuration/acme/dns01/). Here's a Cloudflare example:

```bash
kubectl create secret generic cloudflare-api-token \
  -n cert-manager --from-literal=api-token=YOUR_TOKEN
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-dns-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-example
  namespace: default
spec:
  secretName: wildcard-example-tls
  dnsNames:
    - example.com
    - "*.example.com"
  issuerRef:
    name: letsencrypt-dns
    kind: ClusterIssuer
```

For a `ClusterIssuer`, the credentials Secret must live in cert-manager's "cluster resource namespace", which is `cert-manager` by default. cert-manager supports Route53, Google Cloud DNS, Azure DNS, DigitalOcean, and others natively, and more providers through webhooks.

| Challenge | Port needed | Wildcards | Best for |
| --- | --- | --- | --- |
| HTTP-01 | 80, public | No | Most public sites |
| DNS-01 | None | Yes | Wildcards, private clusters, many subdomains |

## Troubleshooting cert-manager and Let's Encrypt

cert-manager works through a chain of resources: **Certificate → CertificateRequest → Order → Challenge**. When something is stuck, describe each one in order and read the `Events` and `Status` sections:

```bash
kubectl get certificate,certificaterequest,order,challenge -A
kubectl describe certificate myapp-tls -n default
kubectl describe certificaterequest -n default
kubectl describe order -n default
kubectl describe challenge -n default
kubectl logs -n cert-manager deploy/cert-manager
```

The optional [`cmctl`](https://cert-manager.io/docs/reference/cmctl/) CLI summarizes the whole chain with `cmctl status certificate myapp-tls -n default`.

| Symptom | Cause | Fix |
| --- | --- | --- |
| ClusterIssuer not `READY` | Wrong ACME URL, or the webhook wasn't ready | `kubectl describe clusterissuer`, fix the URL, re-apply |
| Challenge stuck `pending` with "connection refused" or timeout | Port 80 not reachable, or DNS points to the wrong IP | Check the DNS record, firewall, and Traefik Service external IP |
| Challenge returns 404 | Solver Ingress uses the wrong class, or Traefik ignores it | Set `ingressClassName: traefik` in the solver; check `kubectl get ingress -A` |
| Certificate never created from Ingress | Annotation missing, or no `tls` block | Add `cert-manager.io/cluster-issuer` and `tls.secretName` |
| Browser warns "not trusted" | Still using the staging issuer | Switch to `letsencrypt-prod` and delete the Secret |
| Traefik serves "TRAEFIK DEFAULT CERT" | Secret missing, or in another namespace | The Secret must be in the same namespace as the Ingress/IngressRoute |
| `429 rateLimited` in Order events | Hit a Let's Encrypt production limit | Wait it out, use staging while debugging, see the [rate limits docs](https://letsencrypt.org/docs/rate-limits/) |
| DNS-01 fails with auth errors | Token lacks permissions, or Secret in the wrong namespace | Give the token DNS edit rights; put the Secret in `cert-manager` |
| `ERR_SSL_PROTOCOL_ERROR` | Plain HTTP served on 443, or the TLS router is missing | Check the `websecure` entrypoint and the `tls` section. The concepts carry over from [fixing ERR_SSL_PROTOCOL_ERROR](/blog/how-to-fix-err-ssl-protocol-error-nginx) |

## How Certificate Renewal Works

Let's Encrypt certificates are short-lived (currently 90 days), and Let's Encrypt has announced plans to shorten them further. By default, cert-manager renews a certificate once it's two-thirds of the way through its lifetime. You can tune this with `spec.renewBefore` on a `Certificate`. Renewal reuses the same issuer and challenge, so whatever worked for the first issuance must keep working. Don't remove the port 80 route or rotate a DNS token without updating the Secret.

To see when a certificate expires and when cert-manager plans to renew it:

```bash
kubectl get certificate myapp-tls -n default \
  -o jsonpath='{.status.notAfter}{"\n"}{.status.renewalTime}{"\n"}'
```

## Monitor Your HTTPS Endpoints

cert-manager reports problems inside the cluster, but nobody reads Kubernetes events at 3 a.m. Check your site from the outside, the way users reach it. If a renewal fails and the certificate expires, or Traefik falls back to its default certificate, HTTPS requests fail and an external check sees it right away.

[GreenCrew](https://greencrew.space) monitors your public HTTPS URLs and sends Discord, Slack, or Telegram alerts when they stop responding correctly. It also gives you a public status page to share with users. Monitor one URL per important host, ideally a health route like `/health`. Setup takes a few minutes, as shown in [how to get website downtime alerts on Discord, Slack, and Telegram](/blog/how-to-get-website-downtime-alerts-discord-slack-telegram).

## Frequently Asked Questions

### Do I need cert-manager if Traefik has built-in Let's Encrypt support?

Not for a single Traefik replica on a small cluster. Built-in ACME works and is simpler. You need cert-manager when you run several Traefik replicas, want certificates stored as Kubernetes Secrets other workloads can use, manage everything declaratively in Git, or might switch ingress controllers later.

### Can I use cert-manager with k3s's bundled Traefik?

Yes. Install cert-manager as shown in Step 2 and use `ingressClassName: traefik`, which matches the class k3s creates. To change k3s's Traefik settings, such as redirects, apply a `HelmChartConfig` in `kube-system` instead of installing a second Traefik.

### Why does my Ingress still show the Traefik default certificate?

Traefik can't find a valid Secret for that host. Usually the Certificate isn't `Ready` yet, `secretName` doesn't match, or the Secret is in a different namespace. Run `kubectl describe certificate` and follow the chain down to the Challenge.

### Can I use a self-signed or private CA instead of Let's Encrypt?

Yes. cert-manager supports `SelfSigned` and `CA` issuers for internal services and test clusters. For a quick non-Kubernetes equivalent, see [how to create a self-signed SSL certificate for Nginx](/blog/how-to-create-a-self-signed-ssl-certificate-for-nginx).

## Conclusion

Automatic HTTPS in Kubernetes comes down to four pieces: Traefik receives the traffic, cert-manager talks to Let's Encrypt, a ClusterIssuer holds the ACME settings, and each Ingress asks for a certificate with one annotation and a `tls` block. Test against staging first, then switch to production. Add a redirect Middleware for HTTP to HTTPS, and use DNS-01 when you need wildcards. When something breaks, walk the Certificate → CertificateRequest → Order → Challenge chain.

Then put the same apps behind TLS across the rest of this series, starting with [Express](/blog/how-to-deploy-an-express-app-on-kubernetes), [Node.js and MongoDB with Helm](/blog/how-to-scale-nodejs-and-mongodb-on-kubernetes-with-helm), and [PostgreSQL](/blog/how-to-deploy-postgresql-on-kubernetes). Finally, add an outside check with [GreenCrew](https://greencrew.space) so a failed renewal or a broken route reaches you on Discord, Slack, or Telegram before your users notice.
