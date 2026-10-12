# Memory API Chart

Helm chart for deploying the Memory API Server on Kubernetes. This chart is a fork of the `chatbot-api-chart` adapted for the memory service, including `pgvector` database support.

## What it deploys

A stateless Deployment plus a ClusterIP Service, backed by a bundled **PostgreSQL** subchart whose
image is overridden to `pgvector/pgvector`, since memory retrieval needs the `vector` extension.

| Object | Name |
|---|---|
| Deployment | `<release>-memory-api-chart` |
| Service | `<release>-memory-api-chart` (port 80 to container port `environment.PORT`, 3000) |
| ServiceAccount | `<release>-memory-api-chart` (when `serviceAccount.create`) |
| HorizontalPodAutoscaler | `<release>-memory-api-chart` (when `autoscaling.enabled`) |
| Ingress | `<release>-memory-api-chart` (when `ingress.enabled`) |

The container name inside the pod is `{{ .Chart.Name }}`, i.e. `memory-api-chart`, so use
`-c memory-api-chart` with `kubectl exec`. The Postgres pod is `<release>-postgresql-0`.

## Values

Values live in three files. There is no plain `values.yaml`.

| File | Purpose |
|---|---|
| `base.values.yaml` | Common defaults shared across environments |
| `lcl.values.yaml` | Local overlay: `NODE_ENV: development`, ingress host `memory-chart.local` |
| `snbx.values.yaml` | Sandbox overlay: `NODE_ENV: production`, 2 replicas, TLS ingress |

> In the cluster, FluxCD supplies values from generated ConfigMaps via `valuesFrom:`, not from these
> files directly. They are the source the ConfigMaps are generated from.

### Image

`image.registry` and `image.repository` are separate values, joined by the `memory-api-chart.image`
helper, so a relocating operator overrides only the registry half, per release or through
`global.imageRegistry`. `image.tag` is rewritten in git by FluxCD's ImageUpdateAutomation.

### Database

The environment files override the bundled Postgres image to
`pgvector/pgvector:0.8.1-pg18-trixie`, which is what provides vector similarity search:

```yaml
postgresql:
  image:
    registry: docker.io
    repository: pgvector/pgvector
    tag: 0.8.1-pg18-trixie
  auth:
    username: ps_memory
    database: ps_memory_db
    existingSecret: boucio-memory-db-secret
```

`auth.existingSecret` points at a Secret that External Secrets Operator populates, carrying both
`password` for Postgres and `DATABASE_URL` for the app, so no credential lands in a ConfigMap.
`auth.username` and `auth.database` must stay in sync with the DSN the ExternalSecret template
assembles, since the two build the same connection string from opposite ends.

### Key values

| Key | Description |
|---|---|
| `service.port` | Service port, 80 |
| `environment.PORT` | Container port, 3000 |
| `postgresql.enabled` | Bundled PostgreSQL toggle |
| `environment` | App env vars (see `memory-api-server/.env.example` for the full reference) |

## Probes

`livenessProbe` and `readinessProbe` render only when set in values, and the shipped values leave
both unset. Pods count as ready as soon as the container starts.

## Local usage

```bash
helm dependency build                        # fetch the postgresql subchart
helm lint . -f base.values.yaml -f lcl.values.yaml
helm template test . -f base.values.yaml -f lcl.values.yaml
helm install memory-api . -f base.values.yaml -f lcl.values.yaml
```

The values files layer: `base` first, then exactly one environment file.

> The chart must be published to the chart registry by CI before FluxCD can reconcile it. Pushing
> chart source to git is not enough.

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
