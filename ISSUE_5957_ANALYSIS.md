# Analysis: Issue #5957 — Grafana Dashboard: Not Ready ExternalSecrets from ClusterExternalSecrets Not Shown

## Summary

When a `ClusterExternalSecret` (CES) is faulty, the Grafana dashboard does not reflect the failure. This is caused by two compounding problems: a bug in the CES metrics code that prevents `Ready=False` from ever being recorded, and a gap in the dashboard that has no panel for CES-level failures.

---

## Root Cause 1: Bug in `UpdateClusterExternalSecretCondition`

**File:** `pkg/controllers/clusterexternalsecret/cesmetrics/cesmetrics.go:70-74`

```go
func UpdateClusterExternalSecretCondition(ces *esv1.ClusterExternalSecret, condition *esv1.ClusterExternalSecretStatusCondition) {
    if condition.Status != v1.ConditionTrue {
        // This should not happen
        return  // <-- BUG: exits early when Ready=False
    }
    // ... only reaches here when Ready=True
}
```

### What happens

The comment "This should not happen" is **incorrect**. `SetClusterExternalSecretCondition` (in `util.go:48-51`) calls this function directly with whatever condition `NewClusterExternalSecretCondition` produces — and that function explicitly returns `Ready=False` when `failedNamespaces` is non-empty (`util.go:37-43`).

There are **three call sites** in the controller that can pass `Ready=False`:
1. `clusterexternalsecret_controller.go:167` — when namespace selection fails
2. `clusterexternalsecret_controller.go:179` — when provisioning fails in some namespaces
3. `clusterexternalsecret_controller.go:262` — when removing old secrets fails

### Consequences

- `clusterexternalsecret_status_condition{condition="Ready",status="False"}` is **never set to 1**
- Worse: if a CES was previously ready, the old `{status="True"}=1` metric **persists stale** because it is never toggled back to 0. The CES appears healthy in metrics even after it starts failing.
- A CES that was **never** ready will have **no** `clusterexternalsecret_status_condition` metric at all (neither True nor False).

### Correct reference implementation

`UpdateExternalSecretCondition` in `pkg/controllers/externalsecret/esmetrics/esmetrics.go:90-173` handles this correctly:
- `ConditionFalse`: deletes stale `{status="True"}` metrics, sets `{status="True"}=0`, then sets `{status="False"}=1`
- `ConditionTrue`: deletes stale `{status="False"}` metrics, sets `{status="False"}=0`, then sets `{status="True"}=1`

`UpdatePushSecretCondition` in `pkg/controllers/pushsecret/psmetrics/psmetrics.go:80-129` also handles both statuses correctly.

### Same bug exists in ClusterPushSecret

`UpdateClusterPushSecretCondition` in `pkg/controllers/clusterpushsecret/cpsmetrics/cpsmetrics.go:69-73` has the **identical bug** with the same guard and comment. This is a separate fix but worth noting.

---

## Root Cause 2: Missing Dashboard Panel for Not Ready ClusterExternalSecrets

**File:** `deploy/charts/external-secrets/files/monitoring/grafana-dashboard.json`

### Current dashboard structure (SLIs row, line 29-975)

| Panel | ID | Query | Shows |
|-------|----|-------|-------|
| SecretStore error rate | 118 | `controller_runtime_reconcile_total{controller=~"secretstore",result="error"}` | Reconcile error ratio |
| ClusterSecretStore error rate | 121 | `controller_runtime_reconcile_total{controller=~"clustersecretstore",result="error"}` | Reconcile error ratio |
| ExternalSecret error rate | 119 | `controller_runtime_reconcile_total{controller=~"externalsecret",result="error"}` | Reconcile error ratio |
| ClusterExternalSecret error rate | 120 | `controller_runtime_reconcile_total{controller=~"clusterexternalsecret",result="error"}` | Reconcile error ratio |
| PushSecret error rate | 122 | `controller_runtime_reconcile_total{controller=~"pushsecret",result="error"}` | Reconcile error ratio |
| Provider error rate | 123 | `externalsecret_provider_api_calls_count{status="error"}` | Provider error ratio |
| Provider errors [15m] | 124 | `increase(externalsecret_provider_api_calls_count{status="error"}[15m])` | Error details by provider |
| **Not Ready ExternalSecrets [15m]** | **125** | `externalsecret_status_condition{condition="Ready",status="False"}` | **Table of failing ESes** |
| ES sync call errors [15m] | 126 | `increase(externalsecret_sync_calls_error[15m])` | Sync error timeseries |

### What's missing

There is **no table or panel** showing `clusterexternalsecret_status_condition{condition="Ready",status="False"}`. The only CES-related panel (id: 120) shows reconcile error rates from `controller_runtime_reconcile_total` — this is a transient rate metric that says nothing about the **current status** of the CES.

Even if Root Cause 1 were fixed, there would be nowhere in the dashboard to see which CES resources are currently not ready.

---

## Failure Scenarios

### Scenario A: CES fails to create ESes at all (e.g., namespace selector errors, RBAC issues)
- CES status: `Ready=False`, `FailedNamespaces` populated
- No ESes exist → no `externalsecret_status_condition` metrics at all
- CES metric is never set to False (Root Cause 1)
- Dashboard shows **nothing** — complete blind spot

### Scenario B: CES creates ESes but some namespaces fail
- Some ESes exist and may report their own `externalsecret_status_condition` metrics
- CES status: `Ready=False` (because some namespaces failed)
- CES metric is never set to False (Root Cause 1)
- Dashboard may show individual failing ESes but gives **no indication** they came from a CES or that the CES itself is failing

### Scenario C: CES was healthy, then becomes faulty
- CES metric was set to `{status="True"}=1`
- Now CES has failures, `UpdateClusterExternalSecretCondition` returns early
- Stale `{status="True"}=1` persists forever — dashboard shows CES as healthy when it's not

---

## Proposed Fix

### Fix 1: Correct `UpdateClusterExternalSecretCondition` (Go code)

**File:** `pkg/controllers/clusterexternalsecret/cesmetrics/cesmetrics.go`

Rewrite the function to handle both `ConditionTrue` and `ConditionFalse`, following the pattern in `UpdateExternalSecretCondition` / `UpdatePushSecretCondition`:

1. Remove the `if condition.Status != v1.ConditionTrue { return }` guard
2. When `Ready=False`:
   - Delete stale `{status="True"}` metrics (handles label changes)
   - Set `{status="True"}=0`
   - Set `{status="False"}=1`
3. When `Ready=True`:
   - Delete stale `{status="False"}` metrics (handles label changes) — this already works
   - Set `{status="False"}=0` — this already works
   - Set `{status="True"}=1` — this already works

Note: CES only has one condition type (`Ready`), unlike ExternalSecret which also has `Deleted`. This makes the logic simpler — no need for the `switch condition.Type` structure, just a `switch condition.Status`.

### Fix 2: Add "Not Ready ClusterExternalSecrets" panel to dashboard

**File:** `deploy/charts/external-secrets/files/monitoring/grafana-dashboard.json`

Add a new **table panel** in the SLIs section, next to "Not Ready ExternalSecrets [15m]" (panel id: 125), with:

```promql
sum(clusterexternalsecret_status_condition{condition="Ready",status="False"}) by (name) == 1
```

Note: CES is cluster-scoped, so group by `name` only (no `namespace`). The `namespace` label in the metric will be empty string.

### Fix 3 (separate issue): Same bug in `UpdateClusterPushSecretCondition`

**File:** `pkg/controllers/clusterpushsecret/cpsmetrics/cpsmetrics.go`

Identical fix needed. Could be done in the same PR or separately.

---

## Affected Files

| File | Change | Scope |
|------|--------|-------|
| `pkg/controllers/clusterexternalsecret/cesmetrics/cesmetrics.go` | Fix `UpdateClusterExternalSecretCondition` to handle `ConditionFalse` | Core fix |
| `deploy/charts/external-secrets/files/monitoring/grafana-dashboard.json` | Add "Not Ready ClusterExternalSecrets" table panel | Dashboard fix |
| `docs/snippets/dashboard.json` | Mirror of dashboard file (keep in sync) | Dashboard fix |
| `pkg/controllers/clusterpushsecret/cpsmetrics/cpsmetrics.go` | Fix same bug in `UpdateClusterPushSecretCondition` | Bonus fix |

---

## Testing

1. **Unit test for CES metrics:** Call `UpdateClusterExternalSecretCondition` with `Ready=False` and verify `{status="False"}=1` and `{status="True"}=0` are set. Then call with `Ready=True` and verify the reverse.
2. **Integration/manual test:** Deploy a CES with an invalid namespace selector, verify that:
   - `clusterexternalsecret_status_condition{condition="Ready",status="False"}` equals 1
   - The new dashboard panel shows the failing CES name
3. **Existing tests:** Ensure existing CES metrics tests still pass (the `Ready=True` path is unchanged in behavior).
