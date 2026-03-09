# Post-Mortem: P3 PrometheusOperatorSyncFailed
**Date:** March 4, 2026  
**Severity:** P3 - Monitoring degraded, no user-facing impact  
**Duration:** ~30 days (silent failure since initial deployment)  
**Author:** Clement Okoro  
**Status:** Resolved

---

## Summary

`PrometheusOperatorSyncFailed` had been firing every 5 minutes since the initial
Alertmanager deployment. Root cause was a YAML configuration error: `httpConfig`
(camelCase) was used instead of `http_config` (snake_case) in the Alertmanager
receiver configuration, causing the Prometheus Operator to silently reject the
config on every reconciliation loop.

---

## Timeline

| Time | Event |
|---|---|
| ~Feb 2 | Initial Alertmanager deployment - PrometheusOperatorSyncFailed begins firing silently |
| Feb 2 - Mar 4 | Alert firing every 5 minutes - treated as non-urgent, monitoring appeared functional |
| Mar 4, 14:00 | Gap analysis before Oracle Health interview identifies alert as non-trivial |
| Mar 4, 14:30 | Root cause identified: `httpConfig` vs `http_config` snake_case error |
| Mar 4, 14:35 | Config corrected, secret re-applied, alert resolved within one reconciliation cycle |

---

## Root Cause

Two YAML key naming errors in the Alertmanager secret:
1. `httpConfig` -> should be `http_config` (snake_case required by the Prometheus Operator API)
2. Invalid `relabelings` field present in the Jira webhook receiver block

The Prometheus Operator validated the config on every reconciliation loop (every 5m)
and failed - but continued running with the last known-good config, masking the failure.

---

## Impact

- **User-facing impact:** None - Alertmanager continued routing alerts using last known-good config
- **Hidden risk:** Any config change made during the 30-day window would have been silently rejected
- **Detection gap:** Alert was visible in Grafana from day 1 but not prioritized

---

## Resolution

Corrected both YAML errors in the Alertmanager secret and re-applied:
```bash
kubectl create secret generic alertmanager-kube-prometheus-stack-alertmanager \
  --from-file=alertmanager.yaml \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -
```

`PrometheusOperatorSyncFailed` resolved within one reconciliation cycle (5 minutes).

---

## Action Items

| Action | Status |
|---|---|
| Fix `http_config` snake_case key in all receivers | Complete |
| Remove invalid `relabelings` field from Jira receiver | Complete |
| Add `amtool config check` as pre-apply validation gate | Pending |
| Document Alertmanager YAML conventions in runbook | Complete |

---

## Lessons Learned

**What went well:**
- Alert routing continued working throughout - no actual alert loss.
- Gap analysis before the interview caught a 30-day silent failure.

**What to improve:**
- Silent failures in monitoring config are more dangerous than loud failures.
- `amtool config check alertmanager.yaml` should gate every Alertmanager change before apply.
- snake_case vs camelCase must be verified against upstream API spec, not assumed.

**Key takeaway:**
> Monitoring your monitoring is not optional. A misconfigured Alertmanager that
> appears functional is more dangerous than one that fails loudly.
