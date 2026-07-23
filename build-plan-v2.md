# vecto-graph — Build Plan (v2: distributed, resume-driven architecture)

## Context

v1 of this plan was a single-container FastAPI + Neo4j demo. The user wants something closer to how recommendation engines are actually built in production, sized to still be learnable/buildable solo, and deliberately chosen so every piece is a real, resume-recognizable technology: a managed vector DB, a self-hosted stateful graph DB under real k8s constraints, an API/AI gateway fronting both client and LLM traffic, a multi-node k3s cluster on real cloud VMs, full observability, and a load-test benchmark with results to show.

Decisions locked in with the user so far:
- **Vector DB**: Qdrant Cloud (managed, free tier)
- **Graph DB**: Neo4j, **self-hosted inside k3s** (StatefulSet + PersistentVolumeClaim) — deliberately more k8s surface area (storage, stateful sets) as a learning goal
- **Cluster**: k3s, multi-node, on **Hetzner Cloud** VMs (not local, not physical hardware) — provisioned as real infrastructure
- **Observability**: Prometheus + Grafana (`kube-prometheus-stack`)
- Carried over from v1: Python, MovieLens (`ml-latest-small`) + TMDb enrichment for director/composer data, OpenAI for embeddings + generation

New decisions made in this plan (decisive, not asked as open questions, but flagged for visibility):
- **API/AI gateway**: **APISIX**, used for two distinct purposes — (1) public ingress in front of the recommendation API with rate-limiting + key-auth, and (2) an internal **AI gateway** route that `recommend-api` calls instead of hitting OpenAI directly, using APISIX's `ai-proxy` plugin. This centralizes the OpenAI API key (the app never holds it) and centralizes LLM traffic control (rate limiting, logging, retry) — this is the actual real-world pattern "AI gateway" refers to, not just a generic reverse proxy.
- **Infra as code**: **Terraform** provisions the Hetzner VMs, private network, and firewall; **cloud-init** installs k3s on boot (1 server + 2 agents). This is itself a strong resume line ("Terraform-provisioned k3s cluster on Hetzner Cloud") and keeps the cluster reproducible/destroyable.
- **Package management for third-party services**: **Helm** for anything with an official chart (Neo4j, APISIX, kube-prometheus-stack) — nobody hand-writes Neo4j's YAML in the real world. **Plain Kubernetes manifests via Kustomize** for the app's own pieces (`recommend-api` Deployment/Service, `ingestion` Job, ConfigMaps/Secrets) — small enough to hand-write, and shows you can do both.
- **Storage**: the Hetzner CSI driver, so Neo4j's PVC is backed by a real networked Hetzner Volume (not `local-path`), so the pod is not glued to one node.
- **Node sizing**: 1× CX32 (4 vCPU/8GB, runs the control plane + is pinned via nodeAffinity to also run Neo4j, since it needs the most RAM) + 2× CX22 (2 vCPU/4GB, run `recommend-api`, APISIX, and the observability stack). Estimated cost: **~€14-15/month (~$16/month)** total while the cluster is up — flagging this as real recurring spend, destroy with `terraform destroy` when not in use.
- **Benchmark tool**: **k6** (Grafana's load-testing tool) — industry-standard, and its Prometheus remote-write output means a live load test can be watched in the same Grafana dashboard the observability stack builds, which is a nice concrete demo.
- **Container registry**: GitHub Container Registry (`ghcr.io`) — free, and images pushed there are directly pullable from k3s via an image-pull secret.
- **Not in v1** (explicitly out of scope, to bound complexity): CI/CD pipeline (GitHub Actions build/push), GitOps (ArgoCD/Flux), HA/multi-server etcd, autoscaling (HPA). These are natural "what's next" talking points but not built now.

---

## Architecture overview

```
Hetzner Cloud (Terraform-provisioned, private network)
┌───────────────────────────────────────────────────────────────────┐
│ k3s cluster (1 server + 2 agents)                                 │
│                                                                    │
│  Node A (CX32, control-plane + pinned Neo4j)                      │
│    ├── k3s server                                                 │
│    └── Neo4j StatefulSet (Hetzner CSI PVC)                        │
│                                                                    │
│  Node B/C (CX22, agents)                                          │
│    ├── APISIX (Helm) ── public ingress + AI-gateway routes        │
│    ├── recommend-api Deployment (FastAPI, 2-3 replicas)           │
│    ├── ingestion Job (run on-demand, not always-on)                │
│    └── kube-prometheus-stack (Prometheus + Grafana)                │
└───────────────────────────────────────────────────────────────────┘
        │ public route (rate-limited, key-authed)         │ internal ai-proxy route
        ▼                                                  ▼
   [ client / k6 benchmark ]                          [ OpenAI API ]

External managed service: Qdrant Cloud (free tier) — vector index, queried by recommend-api
```

Request flow for `POST /api/recommend`:
1. Client → APISIX public route (key-auth + rate-limit) → `recommend-api` Service
2. `recommend-api` embeds the query (OpenAI embeddings, called directly — cheap/low-risk enough not to need gatewaying) → queries **Qdrant** for seed movie(s) by vector similarity
3. `recommend-api` runs multi-hop Cypher traversal against **self-hosted Neo4j** from the seed (director/composer/genre/co-rating overlap), ranks candidates
4. `recommend-api` sends the generation prompt to **APISIX's `ai-proxy` route**, which forwards to OpenAI chat completions (APISIX holds the OpenAI key as a plugin secret) → response flows back through APISIX → `recommend-api` → client, with the subgraph path included for auditability

---

## 1. Repo layout

```
vecto-graph/
├── infra/                        # Terraform
│   ├── main.tf                   # hcloud provider, network, firewall
│   ├── servers.tf                # 3 hcloud_server resources + cloud-init
│   ├── cloud-init/
│   │   ├── k3s-server.yaml.tpl   # installs k3s server, writes join token
│   │   └── k3s-agent.yaml.tpl    # installs k3s agent, joins via private IP
│   └── outputs.tf                # kubeconfig retrieval instructions, node IPs
├── k8s/
│   ├── helm-values/
│   │   ├── neo4j-values.yaml     # StatefulSet sizing, PVC storageClass=hcloud-volumes
│   │   ├── apisix-values.yaml
│   │   └── kube-prometheus-stack-values.yaml
│   ├── apisix/
│   │   ├── route-public-recommend.yaml   # APISIX Route CRD: public -> recommend-api
│   │   └── route-ai-proxy-openai.yaml    # APISIX Route CRD: ai-proxy -> OpenAI
│   └── app/                       # Kustomize base for hand-written app manifests
│       ├── kustomization.yaml
│       ├── recommend-api-deployment.yaml
│       ├── recommend-api-service.yaml
│       ├── ingestion-job.yaml
│       ├── configmap.yaml
│       └── secret.yaml.example    # not committed with real values
├── src/vecto_graph/                # same core Python package structure as v1:
│   ├── config.py
│   ├── db/{neo4j_client.py, qdrant_client.py}
│   ├── ingestion/ (download.py, tmdb_enrich.py, embed.py, schema.py, load.py, run_ingestion.py)
│   ├── retrieval/ (vector_search.py [now queries Qdrant], graph_traversal.py, subgraph.py)
│   ├── generation/ (prompt.py, llm.py [calls the APISIX ai-proxy URL, not OpenAI directly])
│   ├── pipeline.py
│   └── api/ (main.py with /recommend, /health, /metrics)
├── Dockerfile.api                 # recommend-api image
├── Dockerfile.ingestion           # ingestion Job image
├── benchmarks/
│   ├── recommend_load_test.js     # k6 script
│   └── report.md                  # generated after a benchmark run
└── scripts/sanity_check.py
```

## 2. Data layer changes from v1

**Qdrant** replaces Neo4j's vector index for the embedding step:
- Collection `movies`, vector size 1536, distance `Cosine`.
- Each point's `id` = `movieId` (int), `payload = {title, year}` for quick display without a Neo4j round-trip.
- `db/qdrant_client.py` wraps `qdrant-client` (official Python SDK): `upsert()` during ingestion, `query_points()` during retrieval.

**Neo4j** keeps the same schema as v1 (Movie/User/Genre/Person nodes; RATED/TAGGED/IN_GENRE/DIRECTED/COMPOSED/ACTED_IN relationships) **minus** the `embedding` vector property and the vector index — Neo4j is now graph-structure-only, Qdrant is vector-only, joined by `movieId`. This split (dedicated vector DB + dedicated graph DB) is itself the realistic "hybrid retrieval" pattern, more so than v1's single-DB shortcut.

Deploy via the **official Neo4j Helm chart** (`neo4j/neo4j`), `k8s/helm-values/neo4j-values.yaml` setting: single instance (no causal cluster — that's a further HA topic, out of scope), `volumes.data.mode: dynamic` with `storageClassName: hcloud-volumes` (installed via the Hetzner CSI driver's own Helm chart, deployed first), memory config sized to the CX32 node's RAM, and a `nodeSelector`/`nodeAffinity` pinning the pod to Node A.

## 3. Ingestion (Job, not a long-running service)

Same logical steps as v1 (download MovieLens → TMDb enrich, cached → build embedding text → OpenAI embeddings → write graph to Neo4j → write vectors to Qdrant), packaged as `Dockerfile.ingestion`, run as a **Kubernetes Job** (`k8s/app/ingestion-job.yaml`) rather than a script run from a laptop — this is what makes ingestion itself part of "the distributed system" instead of a side channel. TMDb credit cache (`data/cache/tmdb_credits.json`) is baked into a ConfigMap or, more realistically, persisted to a small PVC so re-running the Job after a code fix doesn't re-spend the ~30-40 min TMDb pass.

## 4. `recommend-api`

Same pipeline logic as v1 (`pipeline.recommend(query)`), with two changes:
- `retrieval/vector_search.py` now calls Qdrant instead of Neo4j's vector index.
- `generation/llm.py` sends the chat completion request to the **APISIX ai-proxy route's URL** (an internal cluster address, e.g. `http://apisix-gateway.apisix.svc.cluster.local/ai/openai/chat/completions`) instead of `api.openai.com` directly — `recommend-api` doesn't need `OPENAI_API_KEY` in its own env at all for generation (only for embeddings, unless we also proxy those — for v1, proxy only the chat completion calls through APISIX since that's the higher-value, more auditable traffic; embeddings stay direct to keep the ai-proxy config simple).
- Adds a `/metrics` endpoint via `prometheus-fastapi-instrumentator` for Prometheus scraping.
- Deployed via `k8s/app/recommend-api-deployment.yaml` (2-3 replicas, resource requests/limits set, readiness/liveness probes on `/health`) + matching Service.

## 5. APISIX gateway configuration

Installed via the official `apisix` Helm chart (bundles etcd) into an `apisix` namespace.

**Route 1 — public ingress** (`k8s/apisix/route-public-recommend.yaml`, applied as an APISIX `ApiSixRoute`/Admin-API object or via the Ingress controller CRDs): `POST /api/recommend` → upstream `recommend-api.default.svc.cluster.local:8000`, plugins: `key-auth` (client must send `apikey` header), `limit-req` (rate limiting).

**Route 2 — AI gateway** (`k8s/apisix/route-ai-proxy-openai.yaml`): upstream is `api.openai.com`, plugin `ai-proxy` configured with the model (`gpt-4o-mini`) and the OpenAI key stored as an APISIX secret (`Secret` CRD backed by the bundled etcd, referenced by the plugin config — not as a k8s Secret mounted into `recommend-api`). This route is only reachable from inside the cluster (no public listener), so it functions purely as an internal AI gateway.

## 6. Observability

Install `kube-prometheus-stack` via Helm (bundles Prometheus Operator, Grafana, Alertmanager, node-exporter, kube-state-metrics) into a `monitoring` namespace.

- `ServiceMonitor` for `recommend-api` (`/metrics`).
- APISIX's built-in `prometheus` plugin exposes gateway-level metrics (request rate, latency, status codes per route) — add a `ServiceMonitor` for it too.
- One Grafana dashboard (JSON checked into `k8s/helm-values/` or a `grafana/` folder) showing: request rate & p50/p95/p99 latency for `/api/recommend` (APISIX view), `recommend-api`'s internal breakdown (Qdrant query time, Neo4j query time, OpenAI call time — instrument these as histograms in `pipeline.py`), and node CPU/RAM.

## 7. Benchmark

`benchmarks/recommend_load_test.js` — k6 script: ramps virtual users (e.g. 1→50 over 2 minutes) hitting the public APISIX route with a rotating set of MovieLens-style queries, asserting p95 latency and error-rate thresholds. Run with k6's Prometheus remote-write output pointed at the cluster's Prometheus, so the run is visible live on the Grafana dashboard from step 6. After the run, export k6's summary (`--summary-export`) and write `benchmarks/report.md` with p50/p95/p99 latency, throughput (req/s), and error rate — this is the concrete "benchmarking and seeing the result" artifact.

## 8. Milestones (each independently verifiable)

- **Phase 0 — Infra.** Terraform provisions 3 Hetzner VMs + private network + firewall; cloud-init installs k3s (1 server, 2 agents joined over the private network). Verify: `kubectl get nodes` shows 3 `Ready` nodes; `terraform destroy` cleanly tears it all down (confirm this works before building further, since it's real spend).
- **Phase 1 — Storage + data layer.** Install Hetzner CSI driver (Helm). Create Qdrant Cloud collection. Deploy Neo4j via Helm with a Hetzner-backed PVC, pinned to Node A. Verify: `kubectl get pvc` bound, Neo4j Browser reachable via `kubectl port-forward`, a test point upserts/queries successfully against the Qdrant collection via its REST API.
- **Phase 2 — Ingestion.** Build/push `Dockerfile.ingestion`, run as a k8s Job. Verify: Neo4j node/relationship counts + Interstellar's DIRECTED/COMPOSED edges (same Cypher checks as v1); Qdrant collection point count matches movie count.
- **Phase 3 — App layer.** Build/push `Dockerfile.api`, deploy `recommend-api` (2-3 replicas). Verify via `kubectl port-forward` + curl, reproducing the Interstellar → Inception example, calling OpenAI directly (APISIX ai-proxy wiring comes next phase — verify the pipeline logic works before adding gateway indirection).
- **Phase 4 — Gateway.** Deploy APISIX via Helm. Wire the AI-gateway route first, repoint `generation/llm.py` at it, re-verify the same Interstellar/Inception curl test still works (now via APISIX for the LLM call). Then wire the public ingress route with key-auth + rate-limit. Verify: curl the public route with a valid key succeeds, without a key gets 401, and hammering it past the rate limit gets 429.
- **Phase 5 — Observability.** Install `kube-prometheus-stack`, wire ServiceMonitors, build the Grafana dashboard. Verify: dashboard shows live request metrics while curling the endpoint manually.
- **Phase 6 — Benchmark.** Run the k6 script against the public route. Verify: `benchmarks/report.md` produced with real latency/throughput numbers, and the load spike is visible on the Grafana dashboard during the run.

## 9. Verification summary

Same Cypher sanity checks and the Interstellar/Inception functional test as v1 (see `scripts/sanity_check.py`), plus:
```bash
# public route, authed
curl -X POST http://<cluster-public-ip>/api/recommend -H "apikey: <key>" -H "Content-Type: application/json" \
  -d '{"query": "Recommend me sci-fi thrillers like Interstellar"}'

# public route, no key -> expect 401
curl -X POST http://<cluster-public-ip>/api/recommend -H "Content-Type: application/json" -d '{"query": "..."}'

# k6 benchmark
k6 run --out experimental-prometheus-rw benchmarks/recommend_load_test.js
```

### Critical files
- `infra/servers.tf`, `infra/cloud-init/k3s-server.yaml.tpl`
- `k8s/helm-values/neo4j-values.yaml`
- `k8s/apisix/route-ai-proxy-openai.yaml`, `k8s/apisix/route-public-recommend.yaml`
- `src/vecto_graph/generation/llm.py` (APISIX-routed OpenAI call)
- `src/vecto_graph/retrieval/vector_search.py` (Qdrant client)
- `benchmarks/recommend_load_test.js`
