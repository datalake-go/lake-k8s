# lake-k8s

> Deployment artifacts for the datalake-go lakehouse: pre-baked Spark Connect image, Helm chart, Skaffold dev loop, docker-compose for laptop mode.

The operational substrate. Sibling repos ([lakehouse](https://github.com/datalake-go/lakehouse), [lake-orm](https://github.com/datalake-go/lake-orm), [spark-connect-go](https://github.com/datalake-go/spark-connect-go)) all talk to a Spark Connect endpoint — this repo ships the cluster that endpoint lives on, in every form you need it: container, chart, compose, dev loop.

## Features

- **Pre-baked Spark Connect image.** Apache Spark 4.0 + Iceberg + Delta + Hadoop-AWS, JARs resolved at build time and dropped into `$SPARK_HOME/jars/`. First boot is fast, not 60–90s of Maven resolution. Published to `ghcr.io/datalake-go/lake-k8s/spark-connect:<version>`.
- **Helm chart for production.** Spark Connect deployment, Iceberg catalog conf, S3 backend auth, resource limits, Prometheus servlet scrape target on port 4040.
- **docker-compose for laptop mode.** Spark Connect on `:15002`, SeaweedFS on `:8333`, REST catalog on `:8181`. `docker compose up` and the whole datalake-go stack is reachable. No auth, no cloud account required.
- **Skaffold for inner-loop dev.** Point at kind / minikube / k3d and iterate on chart values without a full deploy cycle each time.
- **Single source of truth for catalog defaults.** The `lakeorm` catalog name, the `default` database, the `lakeorm-local` bucket — all the magic strings other repos default to — live here.

## Install

```bash
# laptop
git clone https://github.com/datalake-go/lake-k8s
cd lake-k8s && docker compose up

# production (once the chart lands)
helm install lake-k8s datalake-go/spark-connect \
  --set catalog.warehouse=s3://my-bucket/warehouse
```

Status: v0 scaffold. Full targets in [#8](https://github.com/datalake-go/lake-k8s/issues/8).

## Architecture

### Intuition

Every datalake-go project needs a Spark Connect cluster to talk to. Stock `apache/spark` images spend 60–90 seconds resolving Iceberg and Delta JARs from Maven on first boot, which makes inner-loop dev painful and CI flaky. Production deployments need Helm. Local dev needs docker-compose. These belong in one place, not three copied into every repo.

### Approach

One pre-baked container image with every JAR already resolved. One Helm chart wraps the canonical production shape. One docker-compose carries the laptop-mode stack. Sibling repos reference this repo's compose file rather than carrying their own copies — when the image or config changes, it changes once.

### Implementation

```
Image build                        Deployment targets
    │                                    │
    ▼                                    ├── kind / minikube        (Skaffold dev loop)
ghcr.io/datalake-go/lake-k8s/            ├── docker-compose up      (laptop mode)
  spark-connect:<version>                └── helm install           (production)
(Spark 4.0 + Iceberg + Delta                │
 + Hadoop-AWS, JARs baked in)               ▼
                                      Spark Connect endpoint (sc://host:15002)
                                           │
                                           ▼
                                      consumed by lake-orm, lakehouse,
                                      lake-goose, any database/sql client
```

Paired with:
- [`spark-connect-go`](https://github.com/datalake-go/spark-connect-go) — the Go client that dials these endpoints
- [`lakehouse`](https://github.com/datalake-go/lakehouse) — the composed runtime that uses the compose stack as its local default
