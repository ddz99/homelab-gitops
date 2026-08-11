# homelab-gitops

GitOps definitions for the K3s homelab managed by Argo CD.

The repository uses the app-of-apps pattern. `bootstrap.yaml` creates the root `homelab` Application, which reads selected child Applications from `apps/`. Each child Application renders a pinned Helm chart and continuously reconciles the cluster to the state stored in Git.

## Architecture

```text
Git push
   |
   v
Argo CD: homelab
   |-- cert-manager Application
   |      `-- Let's Encrypt ClusterIssuer
   `-- n8n Application
          |-- n8n Helm chart
          |-- PersistentVolumeClaim
          |-- Traefik Ingress
          `-- HTTPS redirect middleware
```

Infrastructure provisioning, K3s, Argo CD installation, and DuckDNS updates live in the separate [`homelab-hetzner`](https://github.com/ddz99/homelab-hetzner) repository.

## Repository Layout

| Path | Status | Purpose |
| --- | --- | --- |
| `bootstrap.yaml` | Bootstrap | Root Argo CD Application |
| `apps/cert-manager.yaml` | Active | cert-manager and the production Let's Encrypt issuer |
| `apps/n8n.yaml` | Active | Public n8n deployment |
| `apps/caddy-ingress.yaml` | Inactive template | Caddy ingress controller example |
| `apps/gitea.yaml` | Inactive template | Gitea example with placeholder values |
| `apps/uptime-kuma.yaml` | Inactive template | Uptime Kuma example with placeholder values |

The active set is controlled by `spec.source.directory.include` in `bootstrap.yaml`:

```yaml
include: "{cert-manager.yaml,n8n.yaml}"
```

## Prerequisites

- A running K3s cluster with Argo CD installed
- K3s's Traefik ingress controller exposed on public ports 80 and 443
- `n8n.ddzhomelab.duckdns.org` resolving to the server's public IPv4 address
- Network access to GitHub, the Helm repositories, OCI registries, and Let's Encrypt

## Bootstrap

Push this repository before applying the bootstrap manifest. Argo CD reads the child manifests from `main`, not from the local working tree.

```bash
git push origin main

ssh devops@ddzhomelab.duckdns.org 'kubectl apply --filename=-' \
  < bootstrap.yaml
```

Check reconciliation:

```bash
ssh devops@ddzhomelab.duckdns.org \
  'kubectl get applications --namespace argocd -o wide'
```

Expected active Applications:

```text
homelab
cert-manager
n8n
```

## Deployment Workflow

1. Edit a manifest under `apps/`.
2. Validate and review the change.
3. Commit and push to `main`.
4. Argo CD detects the new revision and applies it automatically.

```bash
git diff --check
git diff
git add apps/<application>.yaml
git commit -m "Describe the application change"
git push origin main
```

Both active child Applications use automated sync, pruning, and self-healing. Removing a managed resource from Git can therefore remove it from the cluster.

## n8n

n8n is deployed from the `8gears` OCI Helm chart:

- Chart version: `2.0.1`
- n8n image: `2.34.4`
- Namespace: `n8n`
- Public URL: <https://n8n.ddzhomelab.duckdns.org>
- Storage: 5 GiB `local-path` PersistentVolumeClaim
- Ingress: Traefik with HTTP-to-HTTPS redirect
- TLS: cert-manager with Let's Encrypt HTTP-01 validation

The chart configures n8n's public host, editor URL, webhook URL, HTTPS protocol, proxy trust, and UTC timezone.

Verify n8n:

```bash
ssh devops@ddzhomelab.duckdns.org \
  'kubectl get pods,pvc,ingress --namespace n8n'

curl --fail --show-error --head \
  https://n8n.ddzhomelab.duckdns.org
```

Complete n8n's owner-account setup immediately after first deployment. The setup page is publicly reachable.

## TLS

cert-manager installs its CRDs and a production `ClusterIssuer` named `letsencrypt-production`. The n8n Ingress annotation requests a certificate named `n8n-tls` through an HTTP-01 challenge handled by Traefik.

Check certificate status:

```bash
ssh devops@ddzhomelab.duckdns.org '
  kubectl get clusterissuer letsencrypt-production
  kubectl get certificate,certificaterequest,order,challenge --namespace n8n
'
```

Inspect the certificate served publicly:

```bash
openssl s_client \
  -connect n8n.ddzhomelab.duckdns.org:443 \
  -servername n8n.ddzhomelab.duckdns.org \
  </dev/null 2>/dev/null |
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

If Traefik serves `TRAEFIK DEFAULT CERT`, check that cert-manager is synced, the ClusterIssuer is ready, and the `n8n-tls` Secret exists:

```bash
ssh devops@ddzhomelab.duckdns.org '
  kubectl get application cert-manager --namespace argocd
  kubectl get clusterissuer letsencrypt-production
  kubectl get secret n8n-tls --namespace n8n
'
```

## Enabling Other Applications

The Caddy, Gitea, and Uptime Kuma files are templates and are intentionally excluded from the root Application. They contain placeholder values and must not be enabled unchanged.

Before enabling one:

1. Replace every `CHANGE_ME_*` value.
2. Update the ingress class and TLS configuration.
3. Add required Kubernetes Secrets through a secure secret-management workflow.
4. Add the file name to the include expression in `bootstrap.yaml`.

Do not enable the Caddy ingress Application alongside the built-in Traefik service without first redesigning public port ownership; both controllers would attempt to use ports 80 and 443.

## Secrets

Do not store plaintext credentials, API keys, or encryption keys in this repository. Use a GitOps-compatible secret solution such as SOPS with age, Sealed Secrets, or External Secrets before adding applications that require committed secret references.

## Data and Backups

The n8n PVC survives pod restarts and chart updates, but it resides on the Hetzner server's local disk. Destroying or replacing the server deletes the volume. Back up n8n workflows, credentials, and its data directory before infrastructure replacement.
