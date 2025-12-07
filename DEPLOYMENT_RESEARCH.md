# FlixStream-Ops: Deployment Strategy Research

Konfigurasi deployment strategy untuk penelitian perbandingan:
- **Waktu deployment**
- **Downtime metrics**
- **Rollback time**
- **Resource usage**

---

## 📊 Comparison Summary

| Metric | Manual (Recreate) | Rolling Update | Blue-Green |
|--------|------------------|----------------|------------|
| **Strategy Type** | Recreate | RollingUpdate | Argo Rollouts |
| **Namespace** | `flix-manual` | `flix-rolling` | `flix-bluegreen` |
| **Expected Downtime** | ~30-60 detik | ~5-15 detik | ~0-2 detik |
| **Replicas** | 1 | 2 | 2 |
| **Resource Surge** | 0% | +50% | +100% |
| **Rollback Type** | Manual redeploy | Automatic | Instant switch |
| **Best For** | Testing | Staging | Production |

---

## 🔧 Strategy Details

### 1. Manual (Recreate) - `overlays/manual/`

**Karakteristik:**
- ❌ **High downtime** - Semua pods dihentikan sebelum pods baru di-deploy
- ✅ Simple dan predictable
- ✅ Tidak perlu resource tambahan
- ❌ Rollback manual (redeploy versi lama)

```yaml
strategy:
  type: Recreate
```

**Use Case:** Development, testing, baseline untuk perbandingan downtime

---

### 2. Rolling Update - `overlays/rolling/`

**Karakteristik:**
- ⚠️ **Medium downtime** - Pods di-replace bertahap
- ✅ `maxUnavailable: 0` - Zero capacity loss
- ✅ `maxSurge: 1` - Controlled resource usage
- ✅ Auto rollback via readiness probe
- ✅ PodDisruptionBudget untuk availability

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
minReadySeconds: 5
```

**Best Practices Applied:**
- `maxUnavailable: 0` ensures no capacity loss during update
- `minReadySeconds: 5` prevents premature traffic routing
- PodDisruptionBudgets (PDBs) for each service

**Use Case:** Staging, pre-production

---

### 3. Blue-Green (Argo Rollouts) - `overlays/bluegreen/`

**Karakteristik:**
- ✅ **Zero/near-zero downtime** - Traffic switch instant
- ✅ Preview service untuk testing sebelum promote
- ✅ Instant rollback (switch balik ke blue)
- ❌ Butuh 2x resource selama deployment
- ✅ Manual promotion untuk kontrol penuh

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: bluegreen-<service>
      previewService: bluegreen-<service>-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 60
```

**Best Practices Applied:**
- `autoPromotionEnabled: false` - Manual control untuk research
- `scaleDownDelaySeconds: 60` - Safe rollback window
- `revisionHistoryLimit: 3` - Quick rollback history
- Preview service untuk testing green deployment

**Use Case:** Production, zero-downtime requirement

---

## 🚀 Deployment Commands

### Prerequisites

```bash
# Install Argo Rollouts (untuk Blue-Green)
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install kubectl plugin
brew install argoproj/tap/kubectl-argo-rollouts
```

### Deploy Manual Strategy

```bash
# Deploy
kubectl apply -k overlays/manual

# Watch pods
kubectl get pods -n flix-manual -w

# Expected: ~30-60s downtime
```

### Deploy Rolling Update Strategy

```bash
# Deploy
kubectl apply -k overlays/rolling

# Watch pods
kubectl get pods -n flix-rolling -w

# Expected: ~5-15s downtime
```

### Deploy Blue-Green Strategy

```bash
# Deploy (creates green version paused)
kubectl apply -k overlays/bluegreen

# Check rollout status
kubectl argo rollouts get rollout bluegreen-api-gateway -n flix-bluegreen

# Test preview before promote
curl http://bluegreen-api-gateway-preview.flix-bluegreen:8080/health

# Promote (switch traffic to green)
kubectl argo rollouts promote bluegreen-api-gateway -n flix-bluegreen

# Expected: ~0-2s downtime

# Rollback if needed
kubectl argo rollouts undo bluegreen-api-gateway -n flix-bluegreen
```

---

## 📊 Measuring Metrics

### 1. Downtime Monitoring Script

```bash
#!/bin/bash
# downtime-monitor.sh

SERVICE_URL=$1  # e.g., "http://manual-api-gateway.flix-manual:8080/health"
START_TIME=$(date +%s%3N)
TOTAL_REQUESTS=0
FAILED_REQUESTS=0
DOWNTIME_MS=0
LAST_FAIL_TIME=0

echo "Monitoring $SERVICE_URL..."
echo "Press Ctrl+C to stop and see results"

while true; do
  CURRENT_TIME=$(date +%s%3N)
  
  if curl -sf "$SERVICE_URL" > /dev/null 2>&1; then
    if [ $LAST_FAIL_TIME -ne 0 ]; then
      DOWNTIME_MS=$((DOWNTIME_MS + CURRENT_TIME - LAST_FAIL_TIME))
      LAST_FAIL_TIME=0
    fi
  else
    FAILED_REQUESTS=$((FAILED_REQUESTS + 1))
    if [ $LAST_FAIL_TIME -eq 0 ]; then
      LAST_FAIL_TIME=$CURRENT_TIME
    fi
    echo "$(date '+%H:%M:%S') - DOWN"
  fi
  
  TOTAL_REQUESTS=$((TOTAL_REQUESTS + 1))
  sleep 0.5
done

# On Ctrl+C
trap "echo; echo 'Results:'; echo 'Total Requests: $TOTAL_REQUESTS'; echo 'Failed Requests: $FAILED_REQUESTS'; echo 'Total Downtime: ${DOWNTIME_MS}ms'" EXIT
```

**Usage:**
```bash
# Start monitoring before deployment
./downtime-monitor.sh "http://manual-api-gateway.flix-manual:8080/health"

# In another terminal, trigger deployment
kubectl apply -k overlays/manual
```

### 2. PromQL Queries (Blackbox Exporter)

```promql
# Downtime in last hour
sum_over_time((1 - probe_success{environment="manual"})[1h:10s]) * 10

# Availability percentage
avg_over_time(probe_success{environment="rolling"}[1h]) * 100

# Failed probes count
count_over_time((probe_success{environment="bluegreen"} == 0)[1h:])
```

---

## 📝 Research Results Template

Run deployments and record results:

| Metric | Manual | Rolling | Blue-Green |
|--------|--------|---------|------------|
| **Deployment Time** | ___s | ___s | ___s |
| **Total Downtime** | ___s | ___s | ___s |
| **Failed Requests** | ___ | ___ | ___ |
| **Rollback Time** | ___s | ___s | ___s |
| **Max CPU Spike** | ___% | ___% | ___% |
| **Max Memory Spike** | ___Mi | ___Mi | ___Mi |

### Research Hypothesis:
- Manual (Recreate) = Highest downtime, simplest
- Rolling Update = Medium downtime, gradual
- Blue-Green = Minimal/zero downtime, instant rollback

---

## 📁 Directory Structure

```
flixstream-ops/
├── DEPLOYMENT_RESEARCH.md
├── base/
│   ├── namespace/
│   ├── api-gateway/
│   ├── user-service/
│   ├── catalog-service/
│   ├── recommendation-service/
│   ├── search-service/
│   ├── payment-service/
│   └── frontend/
└── overlays/
    ├── manual/           # Recreate strategy
    │   ├── kustomization.yaml
    │   └── patches/
    ├── rolling/          # RollingUpdate strategy
    │   ├── kustomization.yaml
    │   ├── patches/
    │   └── pdb/
    └── bluegreen/        # Argo Rollouts Blue-Green
        ├── kustomization.yaml
        ├── patches/
        └── rollouts/
```

---

## 🔗 Quick Reference

| Action | Command |
|--------|---------|
| Build manual overlay | `kustomize build overlays/manual` |
| Build rolling overlay | `kustomize build overlays/rolling` |
| Build bluegreen overlay | `kustomize build overlays/bluegreen` |
| Deploy manual | `kubectl apply -k overlays/manual` |
| Deploy rolling | `kubectl apply -k overlays/rolling` |
| Deploy bluegreen | `kubectl apply -k overlays/bluegreen` |
| Promote bluegreen | `kubectl argo rollouts promote <name> -n flix-bluegreen` |
| Rollback bluegreen | `kubectl argo rollouts undo <name> -n flix-bluegreen` |
| Watch rollout | `kubectl argo rollouts get rollout <name> -n flix-bluegreen --watch` |
