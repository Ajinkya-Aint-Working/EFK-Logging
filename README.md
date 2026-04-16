## EFK Logging

# EFK Stack on EKS with Karpenter

> **Elasticsearch · Fluent Bit · Kibana** — centralized logging for EKS clusters provisioned via [Ajinkya-A3/EKS](https://github.com/Ajinkya-A3/EKS) (Terraform + Karpenter).

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Cluster Reference (Karpenter Node Pools)](#cluster-reference-karpenter-node-pools)
3. [Custom Values — What Changed & Why](#custom-values--what-changed--why)
   - [Elasticsearch](#elasticsearch-elastic-valuesyaml)
   - [Fluent Bit](#fluent-bit-fluent-bit-valuesyaml)
   - [Kibana](#kibana-kibana-valuesyaml)
4. [Prerequisites](#prerequisites)
5. [Helm Installation Steps](#helm-installation-steps)
6. [Verification](#verification)
7. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        EKS Cluster                              │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  namespace: efk                                          │   │
│  │                                                          │   │
│  │  ┌─────────────┐    ┌──────────────┐    ┌─────────────┐  │   │
│  │  │  Fluent Bit │    │Elasticsearch │    │   Kibana    │  │   │
│  │  │ (DaemonSet) │    │ (StatefulSet)│    │ (Deployment)│  │   │
│  │  │ every node  │    │  3 replicas  │    │  2 replicas │  │   │
│  │  └─────────────┘    └──────────────┘    └─────────────┘  │   │
│  │         │                  │                   │         │   │
│  │   reads /var/log      gp3 PVC 50Gi         UI for the    │   │
│  │   containers/*.log    per replica             logs       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Node Pools (Karpenter):                                        │
│   ondemand-amd64  ── taint: workload-class=critical:NoSchedule  │
│   spot-arm64      ── taint: workload-class=standard:NoSchedule  │
│   spot-amd64      ── taint: workload-class=standard:NoSchedule  │
└─────────────────────────────────────────────────────────────────┘
```

**Data flow:**
1. Fluent Bit runs as a DaemonSet on every node and tails `/var/log/containers/*.log`
2. A Lua script sets a dynamic index name (`logs-{namespace}-{container}`) and drops self-logs from the `efk` namespace
3. Logs are forwarded over HTTPS to Elasticsearch (TLS enabled, self-signed cert)
4. Kibana connects to Elasticsearch using the auto-generated credentials secret and renders dashboards

---

## Cluster Reference (Karpenter Node Pools)

The EKS cluster is provisioned using Terraform from [github.com/Ajinkya-A3/EKS](https://github.com/Ajinkya-A3/EKS). Karpenter manages three node pools, each with a dedicated taint. Understanding these taints is essential to understanding the scheduling decisions in the custom values files.

### Node Pool Layout

| Pool Name        | Capacity Type | Arch  | Taint (key=value:effect)                         | Purpose                                |
|------------------|---------------|-------|--------------------------------------------------|----------------------------------------|
| `ondemand-amd64` | On-Demand     | amd64 | `workload-class=critical:NoSchedule`             | Stable, predictable nodes for critical stateful workloads |
| `spot-arm64`     | Spot          | arm64 | `workload-class=standard:NoSchedule`             | Cheapest option; best-effort workloads |
| `spot-amd64`     | Spot          | amd64 | `workload-class=standard:NoSchedule`             | Spot fallback for amd64 workloads      |

Nodes are also labelled with `node-pool: <pool-name>` so that `nodeAffinity` rules can target them by label.

### How Taints & Tolerations Work Here

Karpenter applies a `NoSchedule` taint to every node it provisions. **No pod schedules onto a Karpenter node unless it explicitly tolerates the taint.**

- `Elasticsearch` tolerates `workload-class=critical` → lands exclusively on stable On-Demand nodes.
- `Kibana` tolerates `workload-class=standard` → can land on either Spot pool (cost-saving UI tier).
- `Fluent Bit` uses `operator: Exists` (wildcard) → tolerates **every** taint across all node pools, ensuring the DaemonSet runs on every node regardless of pool.

### Karpenter NodePool Skeleton (for reference)

```yaml
# ondemand-amd64 pool (simplified)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: ondemand-amd64
spec:
  template:
    metadata:
      labels:
        node-pool: ondemand-amd64
    spec:
      taints:
        - key: workload-class
          value: critical
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
---
# spot-arm64 pool (simplified)
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-arm64
spec:
  template:
    metadata:
      labels:
        node-pool: spot-arm64
    spec:
      taints:
        - key: workload-class
          value: standard
          effect: NoSchedule
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64"]
```

---

## Custom Values — What Changed & Why

### Elasticsearch (`elastic-values.yaml`)

| Setting | Value | Reason |
|---|---|---|
| `replicas` | `3` | HA quorum; minimum for a production-grade cluster |
| `minimumMasterNodes` | `2` | Prevents split-brain; must be `(replicas/2)+1` |
| `imageTag` | `8.5.1` | Pinned version for reproducibility |
| `esJavaOpts` | `-Xmx1g -Xms1g` | Always set to **half** the memory limit (2Gi) |
| `resources.requests/limits` | `cpu: 1000m, memory: 2Gi` | Guaranteed QoS class; Karpenter uses requests for node sizing |
| `volumeClaimTemplate.storageClassName` | `gp3` | AWS gp3 EBS — better IOPS/throughput than gp2 at same cost |
| `volumeClaimTemplate.storage` | `50Gi` | Per-replica storage; scale up if log volume grows |
| `antiAffinity` | `hard` | Guarantees each replica lands on a **different node** |
| `antiAffinityTopologyKey` | `kubernetes.io/hostname` | Spreads replicas by hostname (not zone) |
| `nodeAffinity.requiredDuringScheduling` | `node-pool: ondemand-amd64` | **Forces** Elasticsearch onto stable On-Demand nodes only |
| `tolerations` | `workload-class=critical:NoSchedule` | Required to schedule on the `ondemand-amd64` Karpenter pool |
| `podManagementPolicy` | `Parallel` | All 3 pods bootstrap simultaneously instead of serially |
| `protocol` | `https` | TLS enabled (cert auto-created by Helm chart) |
| `createCert` | `true` | Chart generates a self-signed CA; used by Fluent Bit and Kibana |
| `secret.enabled` | `true` | Auto-generates `elasticsearch-master-credentials` secret |
| `sysctlVmMaxMapCount` | `262144` | Required kernel parameter for Elasticsearch — set via init container |
| `podSecurityContext` | `fsGroup: 1000, runAsUser: 1000` | Non-root execution |
| `terminationGracePeriod` | `120s` | Gives ES time to flush segments and transfer shards before shutdown |
| `maxUnavailable` | `1` | PodDisruptionBudget — only 1 replica can be down at a time |
| `service.type` | `ClusterIP` | Internal access only; not exposed publicly |
| `ingress.enabled` | `false` | Not exposed externally (access via Kibana or kubectl port-forward) |

---

### Fluent Bit (`fluent-bit-values.yaml`)

#### Scheduling

| Setting | Value | Reason |
|---|---|---|
| `kind` | `DaemonSet` | Must run on every node to collect all container logs |
| `tolerations` | `operator: Exists` | Wildcard toleration — runs on **every** Karpenter node pool regardless of taint |
| `resources.requests` | `cpu: 100m, memory: 128Mi` | Lean requests so Karpenter sizes nodes correctly; DaemonSet multiplies across all nodes |
| `resources.limits` | `cpu: 200m, memory: 256Mi` | Prevents log-burst from starving other workloads |

#### Credentials

```yaml
env:
  - name: ES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: elasticsearch-master-credentials  # auto-created by ES chart
        key: password
  - name: ES_USERNAME
    valueFrom:
      secretKeyRef:
        name: elasticsearch-master-credentials
        key: username
```

Credentials are injected from the same secret the Elasticsearch chart creates — no manual secret management needed.

#### Lua Script — Dynamic Index Routing (`setIndex.lua`)

This is the most important customization. The Lua filter runs **after** the Kubernetes metadata filter and does two things:

1. **Drops self-logs** — any log from the `efk` namespace is silently discarded (`return -1`). This prevents Fluent Bit from sending its own logs to Elasticsearch and creating a feedback loop.

2. **Sets a dynamic index** — sets the `es_index` field to `logs-{namespace}-{container}`, which Fluent Bit uses as the Logstash prefix. This creates per-app indices like:
   - `logs-default-nginx-2025.01.01`
   - `logs-payments-api-2025.01.01`

This makes it easy to set per-index retention policies and search scoped to a specific application in Kibana.

#### Pipeline Config

| Section | Key Change |
|---|---|
| **INPUT** | Tails `/var/log/containers/*.log` with multiline support (Docker + CRI); also captures kubelet systemd logs |
| **FILTER (kubernetes)** | Enriches logs with pod name, namespace, container, labels |
| **FILTER (lua)** | Runs `setIndex.lua` — drops efk namespace, sets `es_index` field |
| **OUTPUT (kube.\*)** | Sends to `elasticsearch-master:9200` over HTTPS; uses `Logstash_Prefix_Key es_index` for dynamic index names; `Suppress_Type_Name On` required for ES 8.x |
| **OUTPUT (host.\*)** | Sends kubelet/systemd logs to a separate `node-{date}` index |

#### TLS Notes

```
tls On
tls.verify Off
```

TLS is enabled so credentials are never sent in plaintext. Verification is off because the certificate is self-signed by the Elasticsearch Helm chart. In production, replace with a cert-manager-issued cert and set `tls.verify On`.

---

### Kibana (`kibana-values.yaml`)

| Setting | Value | Reason |
|---|---|---|
| `elasticsearchHosts` | `https://elasticsearch-master:9200` | Internal DNS; uses HTTPS to match ES config |
| `elasticsearchCertificateSecret` | `elasticsearch-master-certs` | Mounts the CA cert generated by the ES chart |
| `elasticsearchCredentialSecret` | `elasticsearch-master-credentials` | Uses the same auto-generated credentials |
| `replicas` | `2` | HA for the UI tier |
| `extraEnvs NODE_OPTIONS` | `--max-old-space-size=1800` | Limits Node.js heap to 1800MB — must be < memory limit (2Gi) |
| `resources.requests` | `cpu: 250m, memory: 1Gi` | Karpenter uses this for node provisioning |
| `resources.limits` | `cpu: 1000m, memory: 2Gi` | Allows CPU burst for dashboard rendering; memory matches `NODE_OPTIONS` |
| `updateStrategy` | `RollingUpdate, maxSurge: 1, maxUnavailable: 0` | Zero-downtime deployments |
| `tolerations` | `workload-class=standard:NoSchedule` | Allows scheduling on Spot node pools |
| `affinity.nodeAffinity` (preferred) | `spot-arm64` weight 80, `spot-amd64` weight 50 | Prefers cheapest ARM64 Spot nodes; falls back to amd64 Spot |
| `service.type` | `ClusterIP` | Internal only |
| `ingress.enabled` | `false` | Disabled by default; enable with your ingress controller |

---

## Prerequisites

- EKS cluster running (from [Ajinkya-A3/EKS](https://github.com/Ajinkya-A3/EKS))
- `kubectl` configured and pointing to the cluster
- `helm` >= 3.x installed
- AWS EBS CSI driver installed (for `gp3` PVCs)
- Karpenter installed and node pools provisioned

---

## Helm Installation Steps

### 1. Add Helm Repositories

```bash
helm repo add elastic https://helm.elastic.co
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
```

### 2. Create Namespace

```bash
kubectl create namespace efk
```

### 3. Install Elasticsearch

```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace efk \
  --version 8.5.1 \
  -f elastic-values.yaml \
  --wait \
  --timeout 10m
```

> `--wait` blocks until all 3 StatefulSet pods are `Running`. Can take 3–5 minutes for PVCs to provision.

### 4. Verify Elasticsearch is Healthy

```bash
# Check all 3 pods are Running
kubectl get pods -n efk -l app=elasticsearch-master

# Port-forward and check cluster health
kubectl port-forward -n efk svc/elasticsearch-master 9200:9200 &

ES_PASS=$(kubectl get secret -n efk elasticsearch-master-credentials \
  -o jsonpath='{.data.password}' | base64 -d)

curl -s -k -u "elastic:${ES_PASS}" https://localhost:9200/_cluster/health | jq .
# "status" should be "green"
```

### 5. Install Fluent Bit

```bash
helm install fluent-bit fluent/fluent-bit \
  --namespace efk \
  --version 0.57.2 \
  -f fluent-bit-values.yaml \
  --wait
```

> Fluent Bit must be installed **after** Elasticsearch so the `elasticsearch-master-credentials` secret exists.

### 6. Install Kibana

```bash
helm install kibana elastic/kibana \
  --namespace efk \
  --version 8.5.1 \
  -f kibana-values.yaml \
  --wait \
  --timeout 5m
```

### 7. Access Kibana

```bash
kubectl port-forward -n efk svc/kibana-kibana 5601:5601
```

Open [http://localhost:5601](http://localhost:5601) in your browser.

Get the credentials:

```bash
kubectl get secret -n efk elasticsearch-master-credentials \
  -o jsonpath='{.data.username}' | base64 -d; echo
kubectl get secret -n efk elasticsearch-master-credentials \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## Verification

### Check All Pods Are Running

```bash
kubectl get pods -n efk
```

Expected output:

```
NAME                             READY   STATUS    RESTARTS   AGE
elasticsearch-master-0           1/1     Running   0          5m
elasticsearch-master-1           1/1     Running   0          5m
elasticsearch-master-2           1/1     Running   0          5m
fluent-bit-xxxxx                 1/1     Running   0          3m   # one per node
fluent-bit-yyyyy                 1/1     Running   0          3m
kibana-xxxxxxxxxxxxx-xxxxx       1/1     Running   0          2m
kibana-xxxxxxxxxxxxx-yyyyy       1/1     Running   0          2m
```

### Verify Node Placement

```bash
# Elasticsearch should be on ondemand-amd64 nodes
kubectl get pods -n efk -l app=elasticsearch-master \
  -o wide --show-labels

# Kibana should be on spot nodes
kubectl get pods -n efk -l app=kibana \
  -o wide

# Fluent Bit should have a pod on EVERY node
kubectl get pods -n efk -l app.kubernetes.io/name=fluent-bit -o wide
```

### Verify Logs Are Flowing into Elasticsearch

```bash
kubectl port-forward -n efk svc/elasticsearch-master 9200:9200 &

ES_PASS=$(kubectl get secret -n efk elasticsearch-master-credentials \
  -o jsonpath='{.data.password}' | base64 -d)

# List all indices — you should see logs-* indices
curl -s -k -u "elastic:${ES_PASS}" \
  https://localhost:9200/_cat/indices?v | grep logs-
```

Expected (example):

```
green open logs-default-nginx-2025.01.01       ... 1 1
green open logs-kube-system-coredns-2025.01.01 ... 1 1
green open node-2025.01.01                     ... 1 1
```

### Check Fluent Bit Logs for Errors

```bash
kubectl logs -n efk daemonset/fluent-bit --tail=50
```

Look for lines like:

```
[2025/01/01 00:00:00] [ info] [output:es:es.0] worker #0 connected to elasticsearch-master:9200
```

If you see TLS errors, confirm the ES cert secret exists:

```bash
kubectl get secret -n efk elasticsearch-master-certs
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| ES pods stuck in `Pending` | No `ondemand-amd64` node available | Check Karpenter provisioned the node: `kubectl get nodes -l node-pool=ondemand-amd64` |
| ES pods stuck in `Pending` | PVC not provisioning | Check EBS CSI driver: `kubectl get csidriver ebs.csi.aws.com` |
| ES cluster status `red` or `yellow` | Not all 3 replicas ready | Wait; or check `kubectl describe pod elasticsearch-master-0 -n efk` |
| Fluent Bit not on all nodes | Toleration mismatch | Confirm `tolerations: - operator: Exists` is in fluent-bit-values.yaml |
| No indices in ES | Fluent Bit auth failure | Check ES password secret: `kubectl get secret -n efk elasticsearch-master-credentials` |
| Kibana `502 Bad Gateway` | Kibana can't reach ES | Check ES service: `kubectl get svc -n efk elasticsearch-master` |
| `efk` namespace logs appearing in ES | Lua script not loaded | Check `kubectl logs -n efk daemonset/fluent-bit` for Lua errors |

---

## File Reference

```
Values-custom/
├── elastic-values.yaml      # Elasticsearch StatefulSet — 3 replicas on ondemand-amd64
├── fluent-bit-values.yaml   # DaemonSet with Lua routing, wildcard toleration
└── kibana-values.yaml       # 2-replica Deployment preferring spot nodes
```