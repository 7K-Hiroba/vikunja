# Vikunja

[Vikunja](https://vikunja.io) — open-source, self-hostable to-do and project management app — packaged for the [Hiroba](https://github.com/7K-Hiroba/Hiroba) ecosystem.

The chart deploys the official `vikunja/vikunja` container image; only the chart and platform wiring are custom.

## Structure

```text
├── helm/
│   ├── base/           # Vikunja workload (Deployment, Service, HTTPRoute, PVC)
│   └── platform/       # CNPG Postgres, ExternalSecrets, ServiceMonitor, alerts
├── compositions/
│   └── crossplane/     # App-specific Crossplane compositions (XRDs, Compositions)
├── gitops/
│   ├── argocd/         # ArgoCD Application manifests
│   └── fluxcd/         # FluxCD Kustomization manifests
├── docs/               # TechDocs content
├── .github/workflows/  # CI/CD (references 7K-Hiroba/workflows-library)
└── catalog-info.yaml   # Backstage catalog entry
```

## Quick start

```bash
# Workload
helm install vikunja ./helm/base

# Platform dependencies (Postgres, ExternalSecrets, observability) — optional
helm install vikunja-platform ./helm/platform \
  --set postgres.enabled=true \
  --set externalSecrets.enabled=true
```

For a working install with the platform chart's defaults you'll also need:

- A `Gateway` resource the [`HTTPRoute`](helm/base/templates/httproute.yaml) can attach to
- The CloudNativePG operator (for `postgres.enabled=true`)
- A `ClusterSecretStore` with entries at `vikunja/postgres` and `vikunja/service`

## Documentation

Full documentation is available at [hiroba.7kgroup.org/apps/vikunja](https://hiroba.7kgroup.org/apps/vikunja), or locally under [docs/](docs/).

## Contributing

Please read the [Contributing Guide](https://github.com/7K-Hiroba/Hiroba/blob/main/CONTRIBUTING.md) before opening pull requests.

## Part of the Hiroba ecosystem

Packaged with [Hiroba](https://github.com/7K-Hiroba/Hiroba) by 7KGroup.
