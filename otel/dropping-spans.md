# Runbook: OTel Collector Dropping Spans

**Severity:** P1 / P2
**Applies to:**  GKE

---

## Symptoms
- Grafana shows span count drop > 10% over 5-minute window
- `otelcol_processor_dropped_spans` Prometheus metric is nonzero
- Jaeger showing incomplete traces or missing services

---

## Immediate Triage

```bash
# 1. Check OTel Collector pod health
kubectl get pods -n observability -l app=otel-collector

# 2. Check memory — most common cause
kubectl top pods -n observability -l app=otel-collector

# 3. Check Collector metrics
kubectl port-forward svc/otel-collector 8888:8888 -n observability
curl -s localhost:8888/metrics | grep -E "dropped|refused|queue"

# 4. Check memory_limiter triggering
curl -s localhost:8888/metrics | grep otelcol_processor_refused
```

---

## Root Causes & Fixes

### Cause 1: Memory limiter rejecting data
```bash
kubectl get configmap otel-collector-config -n observability -o yaml | grep memory_limiter -A5
kubectl scale deployment otel-collector --replicas=5 -n observability
```

### Cause 2: Export queue full
```bash
curl -s localhost:8888/metrics | grep otelcol_exporter_queue
# Fix: increase queue_size in config to 5000
```

### Cause 3: GCP Pub/Sub quota exceeded
```bash
gcloud pubsub topics describe telemetry-raw --project=my-gcp-project
# Fix: request quota increase or add batching
```

### Cause 4: OOMKilled
```bash
kubectl describe pod <pod-name> -n observability | grep -A5 "Last State"
kubectl set resources deployment otel-collector \
  --limits=memory=1Gi --requests=memory=512Mi -n observability
```

---

## Escalation
- Unresolved in 30 min → page SRE lead
- Create Jira ticket: Project=OBS, Label=P1-incident
- Post in #observability-oncall Slack
