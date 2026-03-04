# sre-runbooks

Operational runbooks, incident post-mortems, and chaos experiment reports
for the SRE Lab platform.

> "A runbook that doesn't exist when you need it is worse than no runbook at all."

## Incident Reports

| Date | Title | Severity | Duration | Status |
|---|---|---|---|---|
| [2026-02-28](./incident-reports/2026-02-28-storage-incident.md) | Storage controller deadlock — iSCSI/CRI-O conflict | P1 | ~4 hours | ✅ Resolved |
| [2026-03-04](./incident-reports/2026-03-04-prometheus-operator-sync.md) | PrometheusOperatorSyncFailed — config key error | P3 | ~30 days (silent) | ✅ Resolved |

## Platform

All runbooks reference the production-grade SRE Lab platform running on AWS.
See [sre-lab-infra](https://github.com/clementokoro/sre-lab-infra) for full architecture.

**Stack:** Kubernetes v1.29 · ArgoCD · Prometheus · Grafana · Loki · Tempo · DataDog · PagerDuty
