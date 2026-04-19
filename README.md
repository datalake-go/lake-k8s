# k8s

> Deployment artifacts for [datalakego](https://github.com/datalakego/dorm) — local via Skaffold, production via Helm.

This repo is the sanctioned way to stand up the two things a Go
`dorm.Open(...)` call expects to reach:

1. **Spark Connect server** with Iceberg + Delta + Hadoop-AWS + AWS SDK
   JARs pre-built into a single image — owning the Spark/JAR version
   matrix centrally so every user doesn't solve it themselves. The
   Iceberg catalog is the Hadoop catalog by default (filesystem IS the
   catalog, metadata lives as `metadata/` JSON files alongside the
   data). Production overlays that need REST / Glue / Unity swap the
   relevant `spark.sql.catalog.*` confs.
2. **An object store** (SeaweedFS by default for local; S3 / GCS /
   Azure via values.yaml on production overlays).

See [`datalakego/TECH_SPEC.md`](https://github.com/datalakego/dorm/blob/main/TECH_SPEC.md)
"dorm-local to production: one line of code" for why the split
exists and how the local-to-prod arc is meant to feel.

## Contents

```
k8s/
├── skaffold.yaml             # Day-1 developer entry point — `skaffold dev`
├── docker-compose.yaml       # Laptop-mode equivalent for non-k8s dev
├── chart/                    # Helm chart — the production surface
│   ├── Chart.yaml
│   ├── values.yaml           # User-tunable knobs
│   └── templates/            # Spark Connect, catalog, storage, ingress
├── images/
│   └── spark-connect/        # Dockerfile for the canonical image
├── manifests/                # Non-Helm fallback (raw YAML)
├── dorm-local.toml           # Reference config for dorm.Local()
└── examples/                 # Overlays: aws.values.yaml, gcp.values.yaml, local.values.yaml
```

## Status

v0 scaffold. Values files and templates are stubs that describe the
target shape; real chart bodies land in a follow-up commit.

## Quickstart (docker-compose — no k8s required)

```bash
docker compose up -d
# SeaweedFS (S3 API) on http://localhost:8333
# Spark Connect on sc://localhost:15002
```

Then in your Go app:

```go
db, _ := dorm.Open(
    spark.Remote("sc://localhost:15002"),
    iceberg.Format(iceberg.WithCatalog(iceberg.RESTCatalog("http://localhost:19120/api/v1"))),
    backend.MustS3("s3://dorm-local/lake?endpoint=http://localhost:8333&path_style=true&access_key=dorm&secret_key=dorm"),
)
```

## Quickstart (kind / minikube — the Day-1 developer path)

```bash
# Prerequisites: kind OR minikube, skaffold, kubectl, helm
skaffold dev
```

`skaffold dev` builds the Spark Connect image, applies the Helm chart,
watches for file changes, and hot-reloads. When it prints the service
endpoints, pass them to `dorm.Open` (or use `dorm.Local()` which reads
them from `dorm-local.toml`).

## Quickstart (production)

```bash
helm install dorm ./chart -f examples/aws.values.yaml
```

The chart ships with production-validated Spark Connect tunings
(inbound/outbound gRPC message limits, Arrow batch size, reattachable
stream duration, serializer). See `chart/templates/spark-connect.yaml`
for the annotated config.

## The canonical image

`images/spark-connect/Dockerfile` builds `ghcr.io/datalakego/spark-connect:<spark-version>`
with:

- Apache Spark 3.5.x
- Iceberg 1.6.x runtime (`iceberg-spark-runtime-3.5_2.12`)
- Delta 3.2.x runtime (`delta-spark_2.12`, `delta-storage`, `delta-kernel`)
- Hadoop-AWS 3.3.4 + `aws-java-sdk-bundle`

Version compatibility between these JARs is a minefield — getting any
of it wrong produces `ClassNotFoundException` or `NoSuchMethodError`
buried in Spark RPC errors. This image solves that problem once, here.

## License

Apache 2.0.
