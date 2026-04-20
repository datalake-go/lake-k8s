# lake-k8s

> Deployment artifacts for the datalake-go lakehouse: pre-baked Spark Connect image, Helm chart, Skaffold dev loop, docker-compose for laptop mode.

---

## Intuition

Every datalake-go project needs a Spark Connect cluster to talk to. Stock `apache/spark` images spend 60–90 seconds resolving Iceberg and Delta JARs from Maven on first boot, which makes inner-loop dev painful and CI flaky. Production deployments need Helm. Local dev needs docker-compose. These belong in one place, not three.

## Approach

One pre-baked container image with every JAR already resolved and dropped into `$SPARK_HOME/jars/` at build time — first boot becomes fast. One Helm chart wraps the canonical production shape. One docker-compose carries the laptop-mode stack (Spark Connect + SeaweedFS + default catalog conf) so `make docker-up` in any sibling repo just works.

## Implementation

Target layout (v0 scaffold today — see [#8](https://github.com/datalake-go/lake-k8s/issues/8) for the full roadmap):

- `images/spark-connect/` — Dockerfile + baked-in JAR set, published to `ghcr.io/datalake-go/lake-k8s/spark-connect:<version>`.
- `charts/spark-connect/` — Helm chart exposing catalog conf, S3 backend auth, resource limits, the Prometheus servlet scrape target.
- `skaffold.yaml` — inner-loop dev against kind / minikube / k3d.
- `docker-compose.yaml` — laptop mode: Spark Connect on `:15002`, SeaweedFS on `:8333`, Iceberg REST catalog on `:8181`.

```bash
# laptop
docker compose up

# production (once the chart lands)
helm install lake-k8s datalake-go/spark-connect \
  --set catalog.warehouse=s3://my-bucket/warehouse
```

### Architecture

```
Image build                        Deployment targets
    │                                    │
    ▼                                    ├── kind / minikube        (Skaffold dev loop)
ghcr.io/datalake-go/lake-k8s/            ├── docker-compose up      (laptop mode)
  spark-connect:<version>                └── helm install           (production)
(Spark 4.0 + Iceberg + Delta                │
 + Hadoop-AWS, JARs baked in)               ▼
                                      Spark Connect endpoint
                                      consumed by lake-orm / lake-goose /
                                      lakehouse / any database/sql client
```

Paired with:
- [`spark-connect-go`](https://github.com/datalake-go/spark-connect-go) — the Go client that dials these endpoints
- [`lakehouse`](https://github.com/datalake-go/lakehouse) — the composed runtime that uses the compose stack as its local default
