# Runbook: checkout-api OOMKilled Incident

## Trigger
Pods for `checkout-api` show a climbing `RESTARTS` count, or alerting flags container restarts / request failures for the service.

## Diagnosis

**1. Confirm pod status and restart count**
```bash
kubectl get pods -l app=checkout-api -o wide
```

**2. Confirm OOMKilled as the cause**
```bash
kubectl describe pod <pod-name>
```
Look under `Last State` for:
```
State:          Running
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

**3. Check current memory limit/request vs actual usage**
```bash
kubectl get deployment checkout-api -o jsonpath='{.spec.template.spec.containers[*].resources}'
kubectl top pod -l app=checkout-api
```
Compare the configured limit against `kubectl top` usage — if usage is at or near the limit right before restarts, that confirms the limit is the problem rather than a leak.

**4. Check for a recent change to the manifest**
```bash
kubectl rollout history deployment checkout-api
kubectl get deployment checkout-api -o yaml | grep -A3 resources
```

**5. Rule out a genuine leak**
If the limit looks reasonable (not drastically undersized) and usage climbs steadily over time rather than sitting flat near the limit, treat this as a possible memory leak instead — escalate to the app owner rather than just reverting.

## Resolution

**1. Revert or correct the memory limit**
```bash
kubectl set resources deployment checkout-api \
  --limits=memory=128Mi --requests=memory=128Mi
```
Or apply a corrected manifest via `kubectl apply -f <file>` if the change is tracked in version control (preferred — see below).

**2. Confirm rollout**
```bash
kubectl rollout status deployment checkout-api
```

**3. Verify stability**
```bash
kubectl get pods -l app=checkout-api -w
```
Watch for a few minutes to confirm `RESTARTS` stops climbing.

**4. Confirm requests succeeding**
Check your usual health check / synthetic monitor or:
```bash
kubectl logs -l app=checkout-api --tail=50
```

## Prevention (from postmortem action items)
- Resource-limit changes should go through version control / PR review, not be applied directly with `kubectl set resources` in prod.
- Add a CI check that flags a proposed memory limit if it's below recent `kubectl top` observed usage (or a Prometheus query) for that workload before the change merges.
