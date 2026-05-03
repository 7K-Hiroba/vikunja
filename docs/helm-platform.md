---
sidebar_position: 3
---

# Helm platform chart

The platform chart provisions resources that **surround** Vikunja: PostgreSQL, secrets, and observability. It is optional — install the [base chart](./helm-base.md) alone for a minimal deployment with a SQLite database (set `VIKUNJA_DATABASE_TYPE=sqlite` in `env`).

Every section ships disabled by default so the chart is safe to install into clusters that don't have the relevant operators present. Enabling a feature without its required CRDs causes `helm install` to fail early with a clear error, rather than silently creating Custom Resources nothing is reconciling.

## Matching the base chart

The ServiceMonitor below needs to select the base chart's pods. Set `global.baseInstance` to the base chart's release name so the selector matches both `app.kubernetes.io/name` and `app.kubernetes.io/instance`:

```yaml
global:
  baseInstance: vikunja   # release name used for `helm install <name> ./helm/base`
```

Leave it empty to match by name only — acceptable when only one release of Vikunja runs in the cluster.

> The `PodDisruptionBudget` lives in the [base chart](./helm-base.md#poddisruptionbudget), not here — it's tightly coupled to the Deployment's lifecycle.

Values reference: [`helm/platform/values.yaml`](https://github.com/7K-Hiroba/vikunja/blob/main/helm/platform/values.yaml)

## Install

```bash
helm install vikunja-platform ./helm/platform \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true \
  --set observability.serviceMonitor.enabled=true
```

## PostgreSQL

Provisions a PostgreSQL cluster via [CloudNativePG](https://cloudnative-pg.io/) — Vikunja's recommended database for any non-trivial install.

### Prerequisites

- CloudNativePG operator installed in the cluster
- A `StorageClass` available for the data volume

### Configuration

```yaml
postgres:
  enabled: true
  provider: cnpg
  instances: 1            # use 3 for HA
  storage:
    size: 10Gi
    storageClass: ""      # omit for the cluster default
  database: vikunja
  owner: vikunja
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retentionPolicy: "7d"
```

The cluster is reachable at `<release>-pg-rw` (read-write) and `<release>-pg-ro` (read-only). The base chart's default `VIKUNJA_DATABASE_HOST=vikunja-pg-rw` assumes the platform release is named `vikunja`.

CNPG generates a connection-credentials Secret named `<release>-pg-app` containing the password — use the `ExternalSecrets` section below to expose it to Vikunja under the `VIKUNJA_DATABASE_PASSWORD` key, or wire it directly via `envFrom` if you trust managing the secret in-cluster.

## ExternalSecrets

Populates a Kubernetes `Secret` from an upstream store (Vault, AWS Secrets Manager, 1Password, etc.) via an `ExternalSecret` resource. The base chart's `envFrom` then pulls credentials from this `Secret`.

### Requirements

- [external-secrets operator](https://external-secrets.io/) installed in the cluster
- A `ClusterSecretStore` (or `SecretStore`) configured and reachable

### Default Vikunja mapping

```yaml
externalSecrets:
  enabled: true
  refreshInterval: 1h
  storeRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  data:
    - secretKey: VIKUNJA_DATABASE_PASSWORD
      remoteKey: vikunja/postgres
      property: password
    - secretKey: VIKUNJA_SERVICE_JWTSECRET
      remoteKey: vikunja/service
      property: jwtSecret
```

The keys (`VIKUNJA_DATABASE_PASSWORD`, `VIKUNJA_SERVICE_JWTSECRET`) match Vikunja's environment variable names exactly, so a single `envFrom: secretRef:` is enough on the workload side.

### Wiring back into the base chart

```yaml
# helm/base values override
envFrom:
  - secretRef:
      name: vikunja
```

The Secret is named after `global.appName` (defaults to `vikunja`).

## Observability

### ServiceMonitor

Scrapes Vikunja's `/metrics` endpoint via the Prometheus Operator. Vikunja only exposes metrics when `VIKUNJA_SERVICE_ENABLEMETRICS=true` — the base chart sets this by default.

```yaml
observability:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: kube-prometheus-stack  # match your Prometheus instance selector
    port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

Requires the Prometheus Operator CRDs (`monitoring.coreos.com/v1`).

### Grafana dashboard

Deploys a Vikunja dashboard as a `ConfigMap` with the Grafana sidecar label. The sidecar discovers it and imports it into Grafana automatically.

```yaml
observability:
  grafanaDashboard:
    enabled: true
    folderLabel: "Applications"
```

Dashboards live under `helm/platform/dashboards/*.json`. Each JSON file becomes a key in the ConfigMap and is run through Helm's `tpl` function — use backtick-quoted args for `include` (e.g. `` {{ include `platform.name` . }} ``) since JSON escapes break inside Go template actions. Drop additional dashboards into the directory and they ship alongside the default.

### PrometheusRules

Ships three Vikunja-specific alerts:

- `VikunjaTargetDown` (critical) — Prometheus has been unable to scrape Vikunja for 5 minutes
- `VikunjaHighErrorRate` (warning) — 5xx response ratio above 5% for 5 minutes
- `VikunjaHighLatency` (warning) — p99 request latency above 1s for 5 minutes

```yaml
observability:
  prometheusRules:
    enabled: true
```

The full alert definitions live under `observability.prometheusRules.groups` and are passed through `tpl`, so you can reference chart helpers and release context inside expressions. Override the whole list to replace the built-in alerts, or append to extend them. Requires the Prometheus Operator CRDs.

## What this chart does NOT install

- The parent `Gateway` resource — provided by a gateway chart
- TLS certificates — expected to be attached to the Gateway's HTTPS listener
- Cluster-wide operators (CNPG, external-secrets, Prometheus) — these are platform prerequisites
