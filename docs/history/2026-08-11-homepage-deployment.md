# 2026-08-11 — Homepage dashboard deployment

## Summary

Deployed the [Homepage](https://gethomepage.dev) dashboard as the public
landing page for the homelab at <https://home.ddzhomelab.duckdns.org>,
managed by Argo CD like every other application in this repository.
The dashboard auto-discovers service tiles from Kubernetes Ingress
annotations, shows live cluster/node CPU and memory widgets, and carries a
static bookmarks group for external homelab links.

## Timeline

1. **Design** — requirements settled interactively: hostname
   `home.ddzhomelab.duckdns.org`, fully public (no auth, deliberate
   choice), tiles via Kubernetes ingress auto-discovery. Spec written and
   committed (`9654acb`, `docs/superpowers/specs/2026-08-11-homepage-dashboard-design.md`).
2. **Chart selection** — no official Homepage Helm chart exists; chose the
   de-facto community chart `homepage` 2.1.0 from
   `https://jameswynn.github.io/helm-charts` (bjw-s common 1.5.1). The
   chart's default image (v1.2.0) is stale, so the image was pinned to
   `v1.13.2` following the repo convention.
3. **HTTPS-redirect wrinkle** — the chart has no `extraManifests` escape
   hatch and Traefik Middleware references are namespace-scoped, so n8n's
   middleware could not be reused. Solved with an Argo CD **multi-source
   Application**: the Helm chart plus `manifests/homepage/` in this repo,
   which holds the redirect Middleware.
4. **Implementation** — manifests written, chart render validated locally
   with `helm template` against the exact `valuesObject` before pushing.
   Committed and pushed as `d3494d6`.
5. **Bootstrap re-apply** — pushing was not sufficient: `bootstrap.yaml`
   is not GitOps-managed, so the in-cluster root Application kept the old
   `include` list. Resolved with the documented one-time
   `kubectl apply` of `bootstrap.yaml` over SSH.
6. **Verification** — see below; everything green the same day.

## Changes (commit `d3494d6`)

| File | Change |
| --- | --- |
| `apps/homepage.yaml` | New multi-source Argo CD Application: chart `homepage` 2.1.0, image `v1.13.2`, RBAC for auto-discovery, `HOMEPAGE_ALLOWED_HOSTS`, Traefik ingress with cert-manager TLS, Kubernetes + search widgets, homelab bookmarks |
| `manifests/homepage/middleware.yaml` | New Traefik Middleware `homepage-https-redirect` (HTTP→HTTPS, permanent) |
| `apps/n8n.yaml` | Added `gethomepage.dev/*` annotations to the n8n ingress so it auto-appears as a tile under "Automation" |
| `bootstrap.yaml` | `include` extended to `{cert-manager.yaml,n8n.yaml,homepage.yaml}` |
| `README.md` | Architecture diagram, layout table, expected Applications, and a new Homepage section (incl. the annotation contract for adding tiles) |

## Verification results

- Argo CD: `homepage` Application **Synced / Healthy** at `d3494d6`; pod
  Running with image `ghcr.io/gethomepage/homepage:v1.13.2`.
- TLS: Let's Encrypt certificate issued via the existing
  `letsencrypt-production` ClusterIssuer (HTTP-01); valid
  2026-08-11 → 2026-11-09.
- Redirect: `http://home.ddzhomelab.duckdns.org/` → `308 Permanent
  Redirect` to HTTPS; HTTPS serves `200`.
- Auto-discovery: `/api/services` returns n8n under group "Automation"
  with icon, description, and href — discovered from the ingress
  annotations, not static config.
- Widgets and bookmarks confirmed via `/api/widgets` and `/api/bookmarks`.

## Operational notes

- **Adding a dashboard tile for any future app:** annotate its Ingress
  with `gethomepage.dev/enabled: "true"` plus `name`, `group`, `icon`,
  `description`. No dashboard edits needed.
- **Changing the active app set:** editing `include` in `bootstrap.yaml`
  requires a manual `kubectl apply` of that file — the root Application
  does not manage itself.
- **Deliberately out of scope:** authentication (dashboard is public by
  choice), per-service API widgets (need a secrets solution first), and
  activation of the Gitea / Uptime Kuma templates.
