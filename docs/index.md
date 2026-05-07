---
sidebar_position: 1
---

# Vikunja

[Vikunja](https://vikunja.io) — open-source, self-hostable to-do and project management app — packaged for the [Hiroba](https://github.com/7K-Hiroba/Hiroba) ecosystem.

## Overview

This repository ships two Helm charts: [`vikunja`](./helm-base.md) for the workload (Deployment, Service, HTTPRoute) and [`vikunja-platform`](./helm-platform.md) for the cross-cutting dependencies (PostgreSQL via CNPG, secrets via External Secrets, observability). The container image is consumed unchanged from `docker.io/vikunja/vikunja` — we don't maintain a custom build.

## Architecture

```text
                      ┌────────────────────────────┐
   HTTPRoute ──▶ Service (80) ─▶ Vikunja pod (3456)
                      │            ├─ /api/v1/info  (probe)
                      │            ├─ /metrics      (Prometheus)
                      │            └─ /app/vikunja/files (PVC or emptyDir)
                      │
                      ▼
                Postgres (CNPG, vikunja-pg-rw)
                      ▲
        Secret `vikunja` ◀── ExternalSecret ◀── ClusterSecretStore
        (DB password, JWT secret, OIDC client secret)
```

## Prerequisites

- Access to the Kubernetes cluster
- Helm v3.x installed
- kubectl configured
- A parent `Gateway` (HTTP + HTTPS listeners, TLS configured) — typically from a gateway chart such as `hiroba-gateway`
- For the platform chart's defaults: CloudNativePG, External Secrets Operator, Prometheus Operator

## Quick start

```bash
# Workload
helm install vikunja ./helm/base

# Platform dependencies (Postgres, ExternalSecret, ServiceMonitor)
helm install vikunja-platform ./helm/platform \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true \
  --set observability.serviceMonitor.enabled=true
```

See the per-stack pages for full configuration:

- **[Helm base chart](./helm-base.md)** — Deployment, Service, HTTPRoute, env vars, persistence
- **[Helm platform chart](./helm-platform.md)** — PostgreSQL, ExternalSecrets, observability
- **[Crossplane compositions](./crossplane.md)** — Infrastructure capabilities this app exposes (none today)
