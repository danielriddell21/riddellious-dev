# Architecture

## Traffic flow

```
Internet
  └─→ Cloudflare DNS + WAF (free tier)
        ├─→ Cloudflare Pages: riddellious.dev   (static homepage, no cluster)
        │     ├─→ /            public tool index
        │     └─→ /admin       Cloudflare Access (self-hosted app, owner only)
        ├─→ Cloudflare Tunnel: unum
        │     └─→ cloudflared pod (unum namespace)
        │           ├─→ unum-hash     :8080
        │           ├─→ unum-json     :8080
        │           ├─→ unum-diff     :8080
        │           └─→ unum-diagram  :8080
        ├─→ Cloudflare Tunnel: fiatlux
        │     └─→ cloudflared pod (fiatlux namespace)
        │           └─→ fiatlux      :8080
        └─→ Cloudflare Tunnel: platform      (Grafana/ArgoCD = OIDC SSO)
              └─→ cloudflared pod (platform namespace)
                    ├─→ otel-collector      :4318  (bearer token auth)
                    ├─→ grafana             :3000  (OIDC → Cloudflare Access)
                    ├─→ umami               :3000  (Umami's own login)
                    └─→ argocd-server.argocd :80   (OIDC → Cloudflare Access)

Hetzner K8s (Nuremberg) ← ArgoCD watches manifests/{unum,fiatlux,platform}/ on trunk
```

The apex is the one hostname that is **not** a tunnel: `riddellious.dev` is a
static site (`site/`) on Cloudflare Pages, so the homepage stays up even when the
cluster is down, and costs the 8GB node nothing. Terraform owns the Pages project,
the apex CNAME and the Access app on `/admin`; a GitHub Action uploads the files.

Each namespace runs its own `cloudflared` pod connected to its own Cloudflare tunnel; the three tunnels share one VM and are provisioned by the same `cloudflare_tunnel` Terraform module. The four unum tools (`hash`/`json`/`diff`/`diagram`) run from the same image (`ghcr.io/danielriddell21/unum`) — the Deployment `args` field selects the tool. The shared observability + analytics stack (otel-collector, Prometheus, Loki, Tempo, Grafana, Umami, Postgres) lives in the **platform** namespace; only otel-collector is publicly exposed (via the platform tunnel).

## Telemetry stack (platform namespace)

```
unum + fiatlux pods
  ├── /metrics          → Prometheus scrapes (cross-namespace: <svc>.unum:8080)
  └── OTLP push         → otel-collector.platform:4319 (no auth, in-cluster)

CLI/TUI users (Homebrew installs etc.)
  └── OTLP push         → otel.riddellious.dev → otel-collector:4318 (bearer auth)

otel-collector (platform)
  ├── traces            → Tempo (queryable via Grafana)
  └── metrics :8889     → Prometheus scrapes

Grafana ← Prometheus + Loki + Tempo   (public at grafana.${domain}, own login)
Umami   ← JS snippet via /umami/* proxy on unum/fiatlux pods; websites seeded
          into Umami's Postgres by the umami-seed Job (no manual UI step)
          (admin UI public at umami.${domain}, own login)
        ← homepage (Cloudflare Pages) loads script.js from umami.${domain}
          directly — it is off-cluster, so there is no proxy to go through
```

Grafana and ArgoCD are publicly exposed via the platform tunnel behind Cloudflare
Access OIDC SSO; Umami is exposed with its own login. Prometheus, Loki, and Tempo
have **no authentication** and are never exposed publicly — query them through
Grafana. The observability stack refers to itself by bare service names (same
namespace); consumers in `unum`/`fiatlux` reach it cross-namespace via
`<svc>.platform`.

## Repository structure

```
riddellious-dev/
  site/                       # static homepage → Cloudflare Pages (not ArgoCD)
    index.html                # public index of tools + simulator
    admin/index.html          # grafana/argocd/umami links, behind Access
    style.css                 # shared by both pages
    _headers                  # security headers (CSP allows only Umami)
  manifests/                  # one subdirectory per namespace
    unum/                     # watched by argocd/unum.yaml
      namespace.yaml
      configmap.yaml
      umami-config.yaml       # per-tool Umami website IDs (namespace-local)
      cloudflared/            # unum tunnel client
      hash/ json/ diff/ diagram/   # unum tools (Deployment + Service per dir)
    fiatlux/                  # watched by argocd/fiatlux.yaml
      namespace.yaml
      configmap.yaml          # fiatlux-config (world config, mounted /etc/fiatlux)
      umami-config.yaml       # kosmos website ID (namespace-local)
      cloudflared/            # fiatlux tunnel client
      fiatlux/                # simulator pod: fiatlux + ollama sidecar (pulls qwen2.5:3b via initContainer) + PVC
      sqld/                   # libSQL primary; strategy: Recreate (single-writer)
    platform/                 # watched by argocd/platform.yaml — shared services
      namespace.yaml
      cloudflared/            # platform tunnel client (otel/grafana/umami/argocd)
      otel-collector/ grafana/         # deployment + service + configmap + sealed-secret
      umami/                  # deployment + service + sealed-secret + seed-job (PostSync hook)
      postgres/               # statefulset + service + sealed-secret
      prometheus/ loki/ tempo/         # statefulset + service + configmap
  argocd/                     # Applied once manually during cluster bootstrap;
    unum.yaml                 #   ArgoCD lives in its own argocd namespace and
    fiatlux.yaml              #   reconciles workloads into unum, fiatlux,
    platform.yaml             #   and platform.
    sealed-secrets.yaml       # Sealed Secrets controller (kube-system)
  terraform/
    main.tf                   # providers + S3 backend
    hetzner.tf                # cx32 server + SSH key (k3s via cloud-init)
    cloudflare.tf             # three module calls (unum/fiatlux/platform) + moved blocks
    pages.tf                  # Pages project + apex CNAME + custom domain (homepage)
    access.tf                 # Access SaaS-OIDC apps (grafana/argocd) + self-hosted /admin
    modules/cloudflare_tunnel/   # reusable: tunnel + config + per-hostname CNAME
    variables.tf
    outputs.tf
    backend.hcl               # Cloudflare R2 state backend (fill in ACCOUNT_ID)
    terraform.tfvars.example
  docs/
    architecture.md           # this file
    terraform.md
    secrets.md
    operations.md
  .github/
    actions/
      tf-plan-summary/        # composite action — writes plan to job summary
    workflows/
      terraform.yaml          # validate → plan → apply
      pages.yaml              # wrangler upload of site/ on push to trunk
```

## Image tags

| Tag | Source |
|---|---|
| `edge` | Latest push to `trunk` in unum repo |
| `sha-<commit>` | Pinned to a specific commit |
| `v<version>` | Tagged release |
| `latest` | Release promoted to latest in GitHub UI |
| `trunk` | Moving tag tracking the upstream `trunk` branch — used by fiat-lux (no tagged releases yet) with `imagePullPolicy: Always` |

unum deployments use a pinned `v<version>` tag. fiat-lux currently rides `:trunk` until it cuts a versioned release. Edit the `image:` field in the relevant deployment and push to roll forward or back.
