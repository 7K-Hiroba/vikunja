---
sidebar_position: 2
---

# Helm base chart

The base chart deploys Vikunja itself: `Deployment`, `Service`, `ServiceAccount`, optional `PersistentVolumeClaim`, optional `HorizontalPodAutoscaler`, optional `PodDisruptionBudget`, and a Gateway API `HTTPRoute`. It does **not** provision databases, secrets, or observability — see the [platform chart](./helm-platform.md) for those.

The container image is consumed unchanged from `docker.io/vikunja/vikunja`. No custom build is maintained.

Values reference: [`helm/base/values.yaml`](https://github.com/7K-Hiroba/vikunja/blob/main/helm/base/values.yaml)

## Install

```bash
helm install vikunja ./helm/base \
  --set gateway.hostnames[0]=vikunja.yourdomain.com \
  --set 'env[0].name=VIKUNJA_SERVICE_PUBLICURL' \
  --set 'env[0].value=https://vikunja.yourdomain.com'
```

In practice you'll keep overrides in a values file and reference it with `-f values-prod.yaml`.

## Configuration

Vikunja is configured entirely through environment variables — see the [upstream config reference](https://vikunja.io/docs/config-options/). The naming convention is `VIKUNJA_<SECTION>_<KEY>`.

### Default env

The chart ships a Postgres-backed default that lines up with the [`vikunja-platform`](./helm-platform.md) chart:

```yaml
env:
  - name: VIKUNJA_SERVICE_PUBLICURL
    value: "https://vikunja.example.com"   # override per environment
  - name: VIKUNJA_SERVICE_ENABLEMETRICS
    value: "true"
  - name: VIKUNJA_SERVICE_ENABLEREGISTRATION
    value: "false"                         # disable public sign-ups
  - name: VIKUNJA_DATABASE_TYPE
    value: postgres
  - name: VIKUNJA_DATABASE_HOST
    value: vikunja-pg-rw                   # CNPG service from the platform chart
  - name: VIKUNJA_DATABASE_USER
    value: vikunja
  - name: VIKUNJA_DATABASE_DATABASE
    value: vikunja
  - name: VIKUNJA_DATABASE_SSLMODE
    value: require
  - name: VIKUNJA_FILES_BASEPATH
    value: /app/vikunja/files
```

### OpenID Connect (SSO)

Vikunja supports OpenID Connect authentication via environment variables. The non-secret provider settings go in `env`; the client secret is pulled from the platform chart's ExternalSecret via `envFrom`. See the [upstream OpenID docs](https://vikunja.io/docs/openid/) for the full reference.

Uncomment and fill in the provider block in `env`:

```yaml
env:
  # ... existing env vars ...
  - name: VIKUNJA_AUTH_LOCAL_ENABLED
    value: "false"                         # disable local login; SSO only
  - name: VIKUNJA_AUTH_OPENID_ENABLED
    value: "true"
  - name: VIKUNJA_AUTH_OPENID_REDIRECTURL
    value: "https://vikunja.example.com/auth/openid/"
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_NAME
    value: Keycloak
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_AUTHURL
    value: https://keycloak.example.com/realms/myrealm
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_CLIENTID
    value: vikunja
  - name: VIKUNJA_AUTH_OPENID_PROVIDERS_KEYCLOAK_SCOPE
    value: "openid profile email"
```

Replace `KEYCLOAK` in the variable names with the uppercased provider ID you want. The redirect URL for your identity provider must end with `/auth/openid/<provider-id>` (lowercase). For example, with provider ID `keycloak`, the redirect URL registered in Keycloak should be `https://vikunja.example.com/auth/openid/keycloak`.

The client secret (`VIKUNJA_AUTH_OPENID_PROVIDERS_<ID>_CLIENTSECRET`) must **not** be set in `env`. It is injected via `envFrom` from the Secret the platform chart provisions — see the [ExternalSecrets section](./helm-platform.md#externalsecrets).

### Disabling user registration

When using SSO, you typically want to prevent random sign-ups. The chart sets `VIKUNJA_SERVICE_ENABLEREGISTRATION=false` by default. Set it to `"true"` only if you want local account creation alongside SSO.

### Injecting secrets

Don't put credentials in `env`. Use `envFrom` to pull from a `Secret` — typically the one populated by the platform chart's `ExternalSecret`:

```yaml
envFrom:
  - secretRef:
      name: vikunja
```

The platform chart's defaults populate that Secret with `VIKUNJA_DATABASE_PASSWORD`, `VIKUNJA_SERVICE_JWTSECRET`, and (when uncommented) `VIKUNJA_AUTH_OPENID_PROVIDERS_<ID>_CLIENTSECRET`. See the [platform chart docs](./helm-platform.md#externalsecrets) for the data mapping.

## Persistence

Vikunja stores uploaded files at `/app/vikunja/files` (configured via `VIKUNJA_FILES_BASEPATH`). The chart always mounts a writable volume there — either a PVC or an `emptyDir`:

```yaml
persistence:
  files:
    enabled: true             # creates a PVC named <release>-vikunja-files
    size: 5Gi
    accessMode: ReadWriteOnce
    storageClass: ""          # omit for the cluster default
    # existingClaim: my-pvc   # to reuse an existing PVC instead
```

When `enabled: false` (default), files live in an `emptyDir` and are lost on pod restart — fine for a quick demo, never for production.

## Ingress

The base chart emits a Gateway API `HTTPRoute`. A parent `Gateway` (with listeners and TLS) is expected to be provided by the platform — typically a gateway chart such as `hiroba-gateway` — so this chart only owns the route itself.

### Minimum configuration

```yaml
gateway:
  parentRefs:
    - name: default-gateway
      namespace: gateway-system
      sectionName: https   # pin to the HTTPS listener
  hostnames:
    - vikunja.yourdomain.com
```

Pin `sectionName` to an HTTPS listener to avoid silently serving plaintext. HTTP→HTTPS redirect is a listener-level concern and lives on the parent Gateway, not in this chart. Whatever hostname you set here must also match `VIKUNJA_SERVICE_PUBLICURL` in `env`.

### Custom routing rules

The default catch-all sends all traffic to the service. Override `gateway.rules` to split paths or apply filters.

## Scaling

Horizontal autoscaling is off by default and **should stay off** unless `/app/vikunja/files` is on RWX storage (or files are migrated out — Vikunja doesn't currently support an S3 backend).

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

### PodDisruptionBudget

Once you're running more than one replica, enable a PDB so node drains / cluster upgrades can't take Vikunja fully offline:

```yaml
podDisruptionBudget:
  enabled: true
  minAvailable: 1
  # maxUnavailable: "25%"   # mutually exclusive — set minAvailable: null to use this
```

## Probes

Liveness and readiness probes target `/api/v1/info` on the container's `http` port (3456). Override the defaults in `livenessProbe` / `readinessProbe` if you've fronted Vikunja with a path prefix.

## Security context

The pod runs as UID `1000` with `readOnlyRootFilesystem: true` and all Linux capabilities dropped. The chart always mounts an `emptyDir` at `/tmp` plus the files volume at `/app/vikunja/files`. If Vikunja needs to write anywhere else, add an `extraVolumes` / `extraVolumeMounts` pair rather than loosening the security context.
