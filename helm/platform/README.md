# vikunja-platform

Platform dependencies for the [vikunja](../base/README.md) chart — provisions the cross-cutting resources Vikunja needs but that aren't part of the workload's lifecycle.

Install **alongside** `vikunja`, typically in the same namespace.

**Documentation:** <https://hiroba.7kgroup.org/docs/apps/vikunja/helm-platform>

## TL;DR

```bash
helm install vikunja-platform \
  oci://harbor.7kgroup.org/7khiroba/charts/vikunja-platform \
  --version 0.1.0 \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true \
  --set observability.serviceMonitor.enabled=true
```

## Prerequisites

- Kubernetes 1.24+
- [CloudNativePG](https://cloudnative-pg.io/) — for `postgres.enabled=true`
- [External Secrets Operator](https://external-secrets.io/) — for `externalSecrets.enabled=true`
- [Prometheus Operator](https://prometheus-operator.dev/) — for the ServiceMonitor + PrometheusRule
- Grafana with dashboard sidecar enabled (optional, for the shipped dashboard)
- [Crossplane](https://www.crossplane.io/) AWS S3 provider — only if `s3.enabled=true` with the `crossplane` provider

Each section is gated behind `<resource>.enabled` (default `false`). Enabling a feature without its required CRDs causes a hard `helm install` failure with a clear error — see [`templates/checks.yaml`](templates/checks.yaml).

## What gets installed

| Resource | Provided when | Purpose |
| --- | --- | --- |
| CNPG `Cluster` | `postgres.enabled` | Database `vikunja` owned by role `vikunja`, reachable at `<release>-pg-rw` |
| `ExternalSecret` | `externalSecrets.enabled` | Mirrors `vikunja/postgres` and `vikunja/service` from your secret store into a Secret named `vikunja` (consumed by the base chart's `envFrom`); add OIDC client secret mapping as needed |
| `ServiceMonitor` | `observability.serviceMonitor.enabled` | Scrapes `/metrics` on the workload's `http` port (Vikunja must have `VIKUNJA_SERVICE_ENABLEMETRICS=true`) |
| `PrometheusRule` | `observability.prometheusRules.enabled` | `VikunjaTargetDown`, `VikunjaHighErrorRate`, `VikunjaHighLatency` |
| Grafana dashboard | `observability.grafanaDashboard.enabled` | Shipped as a `ConfigMap` with the sidecar discovery label |
| Crossplane `Bucket` / Garage init | `s3.enabled` | S3 bucket — only needed for backups or future S3 integrations; Vikunja itself stores files locally |

## Configuration

Full values in [`values.yaml`](values.yaml), validated by [`values.schema.json`](values.schema.json). Artifact Hub renders the schema as an interactive form.

The chart is split so that **workload-lifecycle** resources (Deployment, HPA, PDB, HTTPRoute, files PVC) live in [`vikunja`](../base/README.md), while **cross-cutting dependencies** (ESO, ServiceMonitor, dashboards, DB) live here. Operators can opt out of platform wiring without losing the app.

## Wiring back into the base chart

Once `externalSecrets.enabled=true`, point the base chart's `envFrom` at the generated Secret:

```yaml
# helm/base values override
envFrom:
  - secretRef:
      name: vikunja
```

The default `data` mappings produce these keys (matching Vikunja env var names):

- `VIKUNJA_DATABASE_PASSWORD` ← `vikunja/postgres` → `password`
- `VIKUNJA_SERVICE_JWTSECRET` ← `vikunja/service` → `jwtSecret`

Add an OIDC client secret mapping when SSO is enabled in the base chart:

- `VIKUNJA_AUTH_OPENID_PROVIDERS_<PROVIDER_ID>_CLIENTSECRET` ← your remote key → `clientSecret`

See the [full docs](https://hiroba.7kgroup.org/docs/apps/vikunja/helm-platform) for details.

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba). Source and issues: <https://github.com/7K-Hiroba/vikunja>.
