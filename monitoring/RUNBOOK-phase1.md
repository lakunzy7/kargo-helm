# Phase 1 Runbook — Central Monitoring for Two kind Clusters

A step-by-step, beginner-friendly guide to how the monitoring stack was set up.
Follow it top to bottom to rebuild Phase 1 from scratch.

---

## 1. The big picture (read this first)

We have **two local Kubernetes clusters** running in Docker via [kind](https://kind.sigs.k8s.io/):

| Cluster | Role | Control-plane IP* | What runs here |
|---|---|---|---|
| **cluster-2** | **Hub** (the central pane) | `172.18.0.3` | Prometheus, Loki, Tempo, Grafana, Alertmanager, Alloy |
| **cluster-1** | **Spoke** (sends data to the hub) | `172.18.0.2` | Alloy only (plus your apps, ArgoCD, Kargo) |

\* IPs come from the shared `kind` Docker bridge network (`172.18.0.0/16`). **They can change when Docker restarts** — always re-check (see step 2).

**The idea:** the spoke collects its own metrics/logs/traces and *pushes* them to the hub. The hub stores everything and Grafana shows both clusters in one place.

```
   cluster-1 (spoke)                         cluster-2 (hub)
   ┌────────────────┐                         ┌─────────────────────────────┐
   │ Alloy          │  metrics :30090  ─────► │ Prometheus (remote-write)   │
   │  - scrapes      │  logs    :30100  ─────► │ Loki                        │
   │  - tails logs   │  traces  :30317  ─────► │ Tempo                       │
   │  - recv traces  │                         │ Grafana  ◄── you look here  │
   └────────────────┘                         │ Alertmanager                │
                                              │ Alloy (hub's own logs/traces)│
                                              └─────────────────────────────┘
```

The hub exposes three **NodePorts** (fixed ports on the cluster node) so the spoke
can reach it across the Docker bridge by the node's DNS name
`cluster-2-control-plane`.

**Design choices (lab-sized, cost-conscious):**
- Self-hosted, **local disk (PVC)** only — no S3/GCS/MinIO, no Thanos.
- Short retention: Prometheus 2 days, Loki & Tempo 48h.
- Loki runs in **SingleBinary** mode (the scalable modes need object storage).

---

## 2. Prerequisites & sanity checks

You need: `docker`, `kubectl`, `helm`, and the two kind clusters already created.

**Check the clusters exist and see their kubectl contexts:**
```bash
kubectl config get-contexts
# expect: kind-cluster-1  and  kind-cluster-2
```

**Re-check the live node IPs (do this every session — they drift!):**
```bash
for n in $(docker ps --format '{{.Names}}'); do
  ip=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' "$n")
  echo "$n -> $ip"
done
# confirm cluster-2-control-plane is the IP your spoke will target
```

> Tip: the spoke config targets the hub by **name** (`cluster-2-control-plane`),
> not by IP, so normal IP drift does not break it. The IP only matters for manual checks.

---

## 3. Add the Helm chart repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana             https://grafana.github.io/helm-charts
helm repo update
```

**Pinned chart versions used in Phase 1** (use these exact versions for a reproducible build):

| Component | Chart | Version |
|---|---|---|
| Prometheus + Grafana + Alertmanager | `prometheus-community/kube-prometheus-stack` | `87.0.0` |
| Loki | `grafana/loki` | `7.0.0` |
| Tempo | `grafana/tempo` | `1.24.4` |
| Alloy (collector) | `grafana/alloy` | `1.10.0` |

---

## 4. Build the HUB (cluster-2)

Always point kubectl at the hub for this whole section:
```bash
kubectl config use-context kind-cluster-2
kubectl create namespace monitoring   # ignore "already exists"
```

### 4a. Prometheus + Grafana + Alertmanager
Config file: [`hub/kube-prometheus-stack.values.yaml`](hub/kube-prometheus-stack.values.yaml)

```bash
helm upgrade --install kube-prom-stack \
  prometheus-community/kube-prometheus-stack --version 87.0.0 \
  -n monitoring -f monitoring/hub/kube-prometheus-stack.values.yaml
```

**Two things this file does that trip people up:**
1. **`enableRemoteWriteReceiver: true`** + **`service.type: NodePort` (30090)** — this is
   what lets the spoke *push* its metrics into the hub Prometheus.
2. **The `cluster` label fix.** Prometheus `externalLabels` only stamp labels on
   *outgoing* data (remote-write/alerts), **not** on the hub's own locally-scraped
   metrics. So without help, cluster-1's data would say `cluster=cluster-1` but the
   hub's own data would have a *blank* cluster label. We fix this by adding a
   `relabelings` rule (`cluster=cluster-2`) to **every built-in ServiceMonitor**
   (kubelet, kube-state-metrics, apiserver, coredns, etc.). That's why the file is long.

### 4b. Loki (logs store)
Config file: [`hub/loki.values.yaml`](hub/loki.values.yaml)

```bash
helm upgrade --install loki grafana/loki --version 7.0.0 \
  -n monitoring -f monitoring/hub/loki.values.yaml
```
Runs as a **SingleBinary** on a local PVC. The file explicitly sets every other
topology component (`read`, `write`, `backend`, `ingester`, …) to `replicas: 0`
and disables `minio`/`gateway`/caches — those are for large, object-storage setups.

### 4c. Tempo (traces store)
Config file: [`hub/tempo.values.yaml`](hub/tempo.values.yaml)

```bash
helm upgrade --install tempo grafana/tempo --version 1.24.4 \
  -n monitoring -f monitoring/hub/tempo.values.yaml
```
Single-binary, local PVC, 48h retention. OTLP receivers on 4317 (gRPC) / 4318 (HTTP)
are on by chart default.

### 4d. Hub Alloy (collects the hub's OWN logs + receives traces)
Config file: [`hub/alloy.values.yaml`](hub/alloy.values.yaml)

```bash
helm upgrade --install alloy grafana/alloy --version 1.10.0 \
  -n monitoring -f monitoring/hub/alloy.values.yaml
```
The hub's metrics are scraped by Prometheus directly, so this Alloy only tails
cluster-2 pod logs (→ local Loki) and receives OTLP traces (→ local Tempo). It
stamps `cluster=cluster-2` on the logs.

### 4e. Open the ingest NodePorts (so the spoke can reach the hub)
Config file: [`hub/nodeport-ingest.yaml`](hub/nodeport-ingest.yaml)

```bash
kubectl apply -f monitoring/hub/nodeport-ingest.yaml
```
This creates two extra `NodePort` Services:
- **`loki-ingest`** → port **30100** (log push)
- **`tempo-ingest`** → ports **30317** (OTLP gRPC) and **30318** (OTLP HTTP)

(Prometheus remote-write port **30090** is opened by its own chart values in 4a.)

**Hub ingest ports summary — the spoke pushes to these:**

| Signal | Hub NodePort | Path |
|---|---|---|
| Metrics | `30090` | `/api/v1/write` |
| Logs | `30100` | `/loki/api/v1/push` |
| Traces | `30317` (gRPC) / `30318` (HTTP) | OTLP |

### 4f. Wait for the hub to come up
```bash
kubectl -n monitoring get pods
# wait until everything is Running / Completed
```

---

## 5. Build the SPOKE (cluster-1)

Switch context to the spoke:
```bash
kubectl config use-context kind-cluster-1
kubectl create namespace monitoring   # ignore "already exists"
```

### 5a. Spoke Alloy (the only thing the spoke needs)
Config file: [`spoke/alloy.values.yaml`](spoke/alloy.values.yaml)

```bash
helm upgrade --install alloy grafana/alloy --version 1.10.0 \
  -n monitoring -f monitoring/spoke/alloy.values.yaml
```

This one Alloy does **all three jobs** and ships each to the hub by DNS name
`cluster-2-control-plane`:
- **Metrics** — scrapes node cAdvisor + annotated pods, stamps `cluster=cluster-1`,
  `remote_write` → `http://cluster-2-control-plane:30090/api/v1/write`
- **Logs** — tails pod logs, stamps `cluster=cluster-1`,
  push → `http://cluster-2-control-plane:30100/loki/api/v1/push`
- **Traces** — listens on OTLP 4317/4318, batches, exports →
  `cluster-2-control-plane:30317`

---

## 6. Verify each signal (the important part)

### 6a. Metrics — are fresh cluster-1 samples reaching the hub?
```bash
kubectl config use-context kind-cluster-2
kubectl -n monitoring port-forward svc/prometheus-operated 19090:9090 &
# How old is the newest cluster-1 sample (seconds)? Should be small (<60).
curl -s 'http://localhost:19090/api/v1/query' \
  --data-urlencode 'query=time()-max(timestamp(up{cluster="cluster-1"}))'
kill %1
```
You should also see **both** `cluster="cluster-1"` and `cluster="cluster-2"` when you
run `count by (cluster) (up)` — that proves the label fix from 4a worked.

### 6b. Logs — are both clusters streaming into Loki?
Open Grafana (below) → **Explore** → Loki → query `{cluster="cluster-1"}` then
`{cluster="cluster-2"}`. Both should return lines.

### 6c. Traces — does the spoke→hub path actually work?
There's no app emitting traces by default, so inject a **synthetic span** into the
spoke and check it lands in the hub Tempo. This is exactly how Phase 1 was validated:

```bash
# 1) make a tiny OTLP span
cat > /tmp/trace.json <<'EOF'
{"resourceSpans":[{"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"trace-path-test"}}]},"scopeSpans":[{"spans":[{"traceId":"5b8efff798038103d269b633813fc60c","spanId":"eee19b7ec3c1b174","name":"verify-span","kind":1,"startTimeUnixNano":"1750668000000000000","endTimeUnixNano":"1750668000100000000"}]}]}]}
EOF

# 2) POST it to the SPOKE Alloy OTLP/HTTP receiver
kubectl config use-context kind-cluster-1
POD=$(kubectl -n monitoring get pods -l app.kubernetes.io/name=alloy -o jsonpath='{.items[0].metadata.name}')
kubectl -n monitoring port-forward $POD 14318:4318 & sleep 3
curl -s -o /dev/null -w "spoke accepted: HTTP %{http_code}\n" \
  -X POST -H 'Content-Type: application/json' --data @/tmp/trace.json \
  http://localhost:14318/v1/traces
kill %1

# 3) confirm it arrived in the HUB Tempo
kubectl config use-context kind-cluster-2
kubectl -n monitoring port-forward svc/tempo 13200:3200 & sleep 3
curl -s -o /dev/null -w "tempo lookup: HTTP %{http_code}\n" \
  http://localhost:13200/api/traces/5b8efff798038103d269b633813fc60c
curl -s http://localhost:13200/metrics | grep '^tempo_distributor_spans_received_total'
kill %1
```
Success looks like: spoke `HTTP 200`, tempo lookup `HTTP 200`, and
`tempo_distributor_spans_received_total` ≥ 1.

> Note: Tempo returns trace/span IDs **base64-encoded** in the JSON body, so they
> look different from the hex you sent — that's normal, not a bug.

### 6d. Alerting — is Alertmanager wired up?
The stack ships a built-in **`Watchdog`** alert that always fires (it's a heartbeat).
Seeing it active confirms the rules → Alertmanager path works.

### 6e. Open Grafana
```bash
kubectl config use-context kind-cluster-2
kubectl -n monitoring get secret kube-prom-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d ; echo   # the admin password
kubectl -n monitoring port-forward svc/kube-prom-grafana 3000:80
# browse http://localhost:3000  (user: admin)
```
Prometheus is auto-added as a datasource; **Loki** and **Tempo** are added by the
hub values file (4a) pointing at their in-cluster Services.

---

## 7. Common problems & fixes

| Symptom | Likely cause | Fix |
|---|---|---|
| Spoke logs: `400 Bad Request: out of bounds` on remote_write | Sample timestamps outside the window Prometheus accepts — usually **clock skew** (common on WSL2 after the host sleeps) or a restart replaying old samples | Resync the node/host clock; or widen Prometheus `tsdb.out_of_order_time_window`. |
| Hub's own metrics have a blank `cluster` label | `externalLabels` don't tag local TSDB | Already handled — the per-ServiceMonitor `relabelings` in 4a stamp `cluster=cluster-2`. |
| Spoke can't reach hub | IP drift / wrong target | Spoke targets the **name** `cluster-2-control-plane`, so re-check it resolves; verify the NodePorts (30090/30100/30317) are open on the hub. |
| `tempo_distributor_spans_received_total` missing entirely | No span has **ever** arrived (Prometheus only creates the counter on first use) | Run the trace injection test in 6c. |
| Loki pods crashloop mentioning object storage | A scalable component got enabled | Confirm only `singleBinary` runs; everything else must be `replicas: 0` (see `loki.values.yaml`). |

---

## 8. File map

```
monitoring/
├── hub/                                   # everything that runs on cluster-2
│   ├── kube-prometheus-stack.values.yaml  # Prometheus+Grafana+Alertmanager (+cluster-label fix)
│   ├── loki.values.yaml                   # Loki SingleBinary, local PVC, 48h
│   ├── tempo.values.yaml                  # Tempo single-binary, local PVC, 48h
│   ├── alloy.values.yaml                  # hub Alloy: cluster-2 logs + trace receiver
│   └── nodeport-ingest.yaml               # opens 30100 (Loki) + 30317/30318 (Tempo)
└── spoke/
    └── alloy.values.yaml                  # spoke Alloy: metrics+logs+traces → hub
```

---

## 9. What's next (out of scope for Phase 1)

- **Phase 2 — GitOps:** deliver this whole stack via ArgoCD (to match the existing
  guestbook pattern) instead of `helm install`. **Blocked**: ArgoCD currently has
  cluster-2 registered at a stale IP and must be re-registered first.
```

