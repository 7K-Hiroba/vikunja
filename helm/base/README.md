# vikunja

Helm chart for [Vikunja](https://vikunja.io) — open-source, self-hostable to-do and project management.

This chart deploys the workload (Deployment, Service, HTTPRoute, optional PVC) using the official `vikunja/vikunja` image. For Postgres, secrets, and observability, install the companion [vikunja-platform](../platform/README.md) chart.

**Documentation:** <https://hiroba.7kgroup.org/docs/apps/vikunja/helm-base>

## TL;DR

```bash
helm install vikunja \
  oci://harbor.7kgroup.org/7khiroba/charts/vikunja \
  --version 0.1.0
```

## Prerequisites

- Kubernetes 1.24+
- Gateway API CRDs installed in the cluster (the chart provisions an `HTTPRoute`)
- A running `Gateway` that the `HTTPRoute` can attach to
- A reachable PostgreSQL instance (the [`vikunja-platform`](../platform/README.md) chart can provision one via CNPG)

## What gets installed

| Resource | Purpose |
| --- | --- |
| `Deployment` | Vikunja API + frontend, listening on port 3456 |
| `Service` | ClusterIP fronting the deployment |
| `HTTPRoute` | Gateway API routing |
| `ServiceAccount` | Pod identity (token automount disabled) |
| `PersistentVolumeClaim` | Optional — uploaded files at `/app/vikunja/files`, enabled via `persistence.files.enabled` |
| `HorizontalPodAutoscaler` | Optional — enabled via `autoscaling.enabled` (single replica only when files live on RWO storage) |
| `PodDisruptionBudget` | Optional — enabled via `podDisruptionBudget.enabled` |

## Configuration

Vikunja is configured entirely through environment variables — see [the upstream config reference](https://vikunja.io/docs/config-options/). The `env` block in [`values.yaml`](values.yaml) ships a sensible default for a Postgres-backed install:

```yaml
env:
  - name: VIKUNJA_SERVICE_PUBLICURL
    value: "https://vikunja.example.com"
  - name: VIKUNJA_DATABASE_TYPE
    value: postgres
  - name: VIKUNJA_DATABASE_HOST
    value: vikunja-pg-rw
  ...
```

Pull credentials (database password, JWT secret) from a Secret via `envFrom` — typically the one populated by the platform chart's `ExternalSecret`:

```yaml
envFrom:
  - secretRef:
      name: vikunja
```

### OpenID Connect (SSO)

Vikunja supports OIDC authentication through environment variables. Add the provider settings to `env` and the client secret to the platform ExternalSecret:

```yaml
env:
  # ... existing vars ...
  - name: VIKUNJA_AUTH_LOCAL_ENABLED
    value: "false"                         # disable local login
  - name: VIKUNJA_AUTH_OPENID_ENABLED
    value: "true"
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_NAME
    value: Keycloak
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_AUTHURL
    value: https://keycloak.example.com/realms/myrealm
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_CLIENTID
    value: vikunja
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_SCOPE
    value: "openid profile email"
```

Replace `KEYCLOAK` with the uppercased provider ID of your choice. The client secret is injected via `envFrom` from the platform chart's ExternalSecret — do **not** set it in `env`.

User registration is disabled by default (`VIKUNJA_SERVICE_ENABLEREGISTRATION=false`).

### Declaring OIDC providers in a config file

Vikunja's config library (Viper) cannot discover provider keys from environment variables alone — the provider key must exist in a `config.yml` for Viper to look for the associated env vars. Enable the `configMap` value to mount a minimal config file:

```yaml
configMap:
  enabled: true
  data:
    config.yml: |
      auth:
        openid:
          enabled: true
          providers:
            keycloak:
```

The file only declares the provider key; all actual values come from `env` and `envFrom` and override the empty entries at runtime.

All values are validated against [`values.schema.json`](values.schema.json). Artifact Hub renders the schema as an interactive form on the chart page.

## Notes

- `readOnlyRootFilesystem: true` is enforced. The chart always mounts an `emptyDir` at `/tmp` and a writable volume at `/app/vikunja/files` (PVC if `persistence.files.enabled`, otherwise `emptyDir`).
- Default `replicaCount: 1`. Enabling `autoscaling` without an RWX volume for files will cause upload inconsistency between replicas.

## Part of the Hiroba ecosystem

Scaffolded with [Hiroba](https://github.com/7K-Hiroba/Hiroba). Source and issues: <https://github.com/7K-Hiroba/vikunja>.
