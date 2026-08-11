# Homepage Dashboard — Design

**Date:** 2026-08-11
**Status:** Approved (design review with dad, 2026-08-11)

## Goal

Deploy the [Homepage](https://gethomepage.dev) dashboard as the public landing
page for the homelab at `https://home.ddzhomelab.duckdns.org`, managed by
Argo CD like every other app in this repository. Service tiles are populated
by Kubernetes ingress auto-discovery so future apps appear on the dashboard
by adding annotations to their ingress, with no dashboard edits.

## Decisions

| Decision | Choice | Rationale |
| --- | --- | --- |
| Hostname | `home.ddzhomelab.duckdns.org` | Short landing-page name; DuckDNS wildcards sub-subdomains, no DNS change needed |
| Access | Fully public, no auth | Owner's explicit choice; dashboard is a show-off piece. Widget data (cluster stats) is visible to anyone |
| Tile source | Kubernetes ingress auto-discovery | New apps self-register via `gethomepage.dev/*` ingress annotations |
| Chart | `homepage` 2.1.0 from `https://jameswynn.github.io/helm-charts` | No official chart exists; this community chart (bjw-s common 1.5.1) is the de-facto standard and supports RBAC, ingress, and config via values |
| Image | `ghcr.io/gethomepage/homepage:v1.13.2` (pinned) | Chart's default appVersion (v1.2.0) is stale; repo convention pins app images |
| HTTPS redirect | Argo CD multi-source Application | Chart has no `extraManifests` escape hatch; second source renders the Traefik Middleware from `manifests/homepage/` in this repo. Keeps parity with n8n's redirect behavior |

## Changes

### 1. `apps/homepage.yaml` (new)

Argo CD `Application`:

- `metadata`: name `homepage`, namespace `argocd`, sync-wave `"0"`,
  resources finalizer (matches n8n).
- `spec.sources` (multi-source):
  1. Helm chart `homepage`, `repoURL: https://jameswynn.github.io/helm-charts`,
     `targetRevision: 2.1.0`, inline `valuesObject`.
  2. `repoURL: https://github.com/ddz99/homelab-gitops.git`,
     `targetRevision: main`, `path: manifests/homepage`.
- `spec.destination`: in-cluster, namespace `homepage`.
- `syncPolicy`: automated with `prune` + `selfHeal`, `CreateNamespace=true`.

Key values in `valuesObject`:

- `image.tag: v1.13.2`
- `enableRbac: true` and a created service account — grants the cluster-wide
  read access (ingresses, nodes, pods, namespaces) needed by auto-discovery
  and the Kubernetes widgets.
- `env`: `HOMEPAGE_ALLOWED_HOSTS=home.ddzhomelab.duckdns.org` (mandatory
  since Homepage v1.0; requests are rejected without it).
- `config.kubernetes.mode: cluster` — enables ingress auto-discovery.
- `config.widgets`: Kubernetes widget (cluster + node CPU/memory) and a
  search widget.
- `config.bookmarks`: static external links — homelab-gitops repo,
  homelab-hetzner repo, Hetzner Cloud console, DuckDNS.
- `config.services`: empty (tiles come from auto-discovery).
- `config.settings`: dashboard title `DDZ Homelab`.
- `ingress.main`: enabled, `ingressClassName: traefik`,
  host `home.ddzhomelab.duckdns.org`, TLS secret `homepage-tls`, annotations:
  - `cert-manager.io/cluster-issuer: letsencrypt-production`
  - `traefik.ingress.kubernetes.io/router.middlewares: homepage-homepage-https-redirect@kubernetescrd`
- `resources`: requests `10m` CPU / `128Mi`, limits `500m` CPU / `512Mi`.

### 2. `manifests/homepage/middleware.yaml` (new)

Traefik `Middleware` named `homepage-https-redirect` in namespace `homepage`
with `redirectScheme: {scheme: https, permanent: true}` — same spec as the
n8n middleware. (Middleware references are namespace-scoped, so n8n's cannot
be reused.)

### 3. `apps/n8n.yaml` (edit)

Add auto-discovery annotations to the n8n ingress so it appears as the first
discovered tile:

```yaml
gethomepage.dev/enabled: "true"
gethomepage.dev/name: n8n
gethomepage.dev/group: Automation
gethomepage.dev/icon: n8n.svg
gethomepage.dev/description: Workflow automation
```

### 4. `bootstrap.yaml` (edit)

Extend the active set:

```yaml
include: "{cert-manager.yaml,n8n.yaml,homepage.yaml}"
```

### 5. `README.md` (edit)

- Add `apps/homepage.yaml` as Active in the layout table and mention
  `manifests/homepage/`.
- Update the `include` example and expected Applications list.
- Add a short "Homepage" section (chart version, image tag, namespace, URL,
  auto-discovery annotation contract) in the style of the n8n section.

## Error handling and rollout

- Argo CD reconciles on push; an invalid chart or values degrades only the
  `homepage` Application. cert-manager and n8n are untouched.
- TLS is issued by the existing `letsencrypt-production` ClusterIssuer via
  HTTP-01; no DNS work needed (DuckDNS wildcard already resolves).
- Removing `homepage.yaml` from the `include` prunes the whole app cleanly
  (automated prune is on).

## Testing

1. Pre-push: `helm template` the chart with the exact `valuesObject`
   (extracted to a temp file) and validate the rendered manifests; validate
   YAML of all changed files.
2. Post-push: `kubectl get applications -n argocd` shows `homepage` Synced +
   Healthy; `curl -I http://home.ddzhomelab.duckdns.org` returns a 30x to
   https; the dashboard loads with a valid certificate and shows the n8n
   tile under "Automation" plus cluster stats.

## Out of scope

- Authentication (explicit owner decision to stay public).
- Per-service API widgets (e.g. the n8n widget needs an API key secret;
  add later, ideally after a secrets story like Sealed Secrets exists).
- Activating the Gitea / Uptime Kuma templates.
