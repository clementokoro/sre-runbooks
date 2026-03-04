# Post-Mortem: P1 Storage Incident
**Date:** February 28, 2026  
**Severity:** P1 — Full platform degradation  
**Duration:** ~4 hours  
**Author:** Clement Okoro  
**Status:** Resolved ✅

---

## Summary

A storage controller deadlock between CRI-O and iSCSI caused progressive pod
eviction across all three Kubernetes nodes in the VMware-based lab. The incident
was resolved by migrating the entire platform to AWS EC2, eliminating the iSCSI
dependency and improving overall cluster stability.

---

## Timeline

| Time (UTC) | Event |
|---|---|
| ~14:00 | Storage I/O latency begins spiking on worker nodes |
| ~14:20 | Pods begin failing liveness probes — evictions start |
| ~14:35 | CRI-O logs show `iSCSI connection timeout` on every container start |
| ~14:50 | All 3 nodes report NotReady — cluster effectively down |
| ~15:10 | Root cause identified: CRI-O/iSCSI deadlock under concurrent write pressure |
| ~15:30 | Decision made: migrate to AWS rather than attempt in-place fix |
| ~18:00 | AWS infrastructure provisioned, cluster rebuilt, all applications healthy |

---

## Root Cause Analysis

**Immediate cause:** CRI-O's container image layer management sent concurrent
iSCSI I/O requests that exceeded the per-session queue depth of the VMware vSAN
virtual iSCSI target. The iSCSI target deadlocked, causing CRI-O to hang waiting
for I/O completion — making it unable to start new containers and triggering
cascading liveness probe failures across all pods simultaneously.

**Contributing factors:**
1. Lab was running on VMware Workstation shared storage with no I/O queue depth
   tuning — an unsupported configuration for production-grade container workloads.
2. No circuit breaker between the storage layer and CRI-O — a single storage
   failure propagated to a full cluster failure.
3. Node-level health check was not monitoring iSCSI session state, so storage
   degradation was not detected before pod evictions began.

---

## Impact

- **User-facing impact:** 100% of SRE lab services unavailable for ~4 hours
- **Data impact:** None — PostgreSQL WAL logs intact, zero data loss
- **Alert coverage:** Alertmanager fired correctly within 2 minutes of the first
  pod eviction. PagerDuty call generated. Slack notification sent to #alerts-sre.

---

## Resolution

The CRI-O/iSCSI conflict is a fundamental incompatibility at the VMware Workstation
storage layer. The correct resolution was to eliminate the dependency by migrating
to AWS EC2 with EBS-backed storage.

**Steps taken:**
1. Provisioned 4 AWS EC2 t3.medium instances in a dedicated VPC
2. Rebuilt Kubernetes cluster using kubeadm on Amazon Linux
3. Re-applied all GitOps manifests via ArgoCD — fully declarative rebuild
4. All 14 ArgoCD Applications reached Synced + Healthy within 2 hours
5. Pre-flight health check (`lab-check.sh`) passed 10/10 green checks

---

## Action Items

| Action | Status |
|---|---|
| Migrate platform from VMware to AWS EC2 | ✅ Complete (Mar 1) |
| Add EBS storage metrics to node health monitoring | ✅ Complete via CloudWatch |
| Document AWS architecture decisions | ✅ Complete (Section 0.9 of lab guide) |
| Implement Velero S3 backups before next drill | ✅ Complete (Section 12) |

---

## Lessons Learned

**What went well:**
- Alertmanager detected the failure and fired to PagerDuty/Slack correctly.
- The fully declarative GitOps architecture meant the platform could be rebuilt
  from scratch in 2 hours with zero manual configuration — no `kubectl apply`.
- PostgreSQL data was intact due to WAL durability — no data loss.

**What to improve:**
- Storage layer health should be monitored at node level, not just pod level.
- A runbook for "storage layer failure" should exist before the incident, not after.
- Circuit breakers between storage and container runtime would prevent a single
  storage failure from becoming a full cluster failure.

**Key takeaway:**
> When the entire infrastructure is declared in Git, "disaster recovery" is just
> running `argocd app sync root-app`. The platform was fully operational on new
> infrastructure in under 2 hours.
```

**Step 3** — Click **"Commit changes"**, set the commit message to:
```
docs(incident): add P1 post-mortem — 2026-02-28 iSCSI/CRI-O storage deadlock
