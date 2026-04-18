# Runbook: OTel/FluentBit Agent CrashLoopBackOff

**Severity:** P2
**Applies to:** All GKE clusters

---

## Symptoms
- `kubectl get pods -n observability` shows CrashLoopBackOff
- Telemetry gaps in Grafana dashboards
- Prometheus alerts firing: OtelCollectorDown or FluentBitDown

---

## Triage Steps

```bash
# Identify crashing pod
kubectl get pods -n observability
kubectl get pods -n logging

# Get crash reason
kubectl describe pod <pod-name> -n observability
kubectl logs <pod-name> -n observability --previous

# Check events
kubectl get events -n observability --sort-by='.metadata.creationTimestamp' | tail -20
```

---

## Common Causes & Fixes

### Bad ConfigMap
```bash
kubectl get configmap otel-collector-config -n observability -o yaml
kubectl rollout undo deployment/otel-collector -n observability
```

### Missing GCP Secret
```bash
kubectl get secret gcp-credentials -n observability
kubectl create secret generic gcp-credentials \
  --from-file=key.json=/path/to/sa-key.json -n observability
```

### Node Out of Resources
```bash
kubectl describe nodes | grep -A5 "Conditions:"
kubectl delete pod <pod-name> -n observability
```

---

## Prevention
- Validate OTel config in CI before deploying
- Set PodDisruptionBudget: minimum 1 collector always running
- Use --dry-run in Argo CD pipelines
