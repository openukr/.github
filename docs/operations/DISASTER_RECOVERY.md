# openUKR — Disaster Recovery Plan

<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->

> [!CAUTION]
> **This document describes planned procedures. openUKR has not yet been fully tested. Do not use in production.**

<!-- END PRE-RELEASE BANNER -->

> **Standards**: BSI CON.1 · ISO/IEC 27001 A.5.29, A.5.30 · NIST SP 800-34

## 1. Overview

openUKR is a **Tier-0 service** — an outage affects all services that depend on asymmetric keys. This document defines recovery objectives, procedures, and test plans.

## 2. Recovery Objectives

| Metric | Target | Rationale |
|---|---|---|
| **RTO** (Recovery Time Objective) | **< 15 minutes** | Controller restart + key generation |
| **RPO** (Recovery Point Objective) | **0** (no key loss) | Keys are stored in K8s Secrets — independent of the controller |
| **MTTR** (Mean Time to Recovery) | **< 10 minutes** | Automated via Kubernetes Deployment |

## 3. Failure Scenarios

### 3.1 Controller Pod Failure

| Property | Value |
|---|---|
| Severity | Low |
| Automatic Recovery | ✅ Kubernetes Deployment restart |
| RTO | < 30 seconds |
| Impact | Rotations delayed, existing keys remain valid |

**Procedure**: No manual action required. Kubernetes restarts the pod. Leader election ensures only one controller is active.

### 3.2 etcd Data Loss

| Property | Value |
|---|---|
| Severity | Critical |
| Automatic Recovery | ❌ Manual |
| RTO | < 15 minutes |
| Impact | All CRDs + Secrets lost |

**Procedure**:
1. Restore etcd backup (cluster operations)
2. `kubectl get keyprofiles -A` — verify CRDs are restored
3. Restart controller — it will regenerate missing secrets
4. Verify key inventory: `kubectl get cm openukr-key-inventory -n openukr-system`

**Prevention**:
- etcd backups every 15 minutes (cluster-level responsibility)
- Encryption-at-rest for etcd enabled (required, see [Compliance Guide](COMPLIANCE_GUIDE.md))

### 3.3 Key Compromise

| Property | Value |
|---|---|
| Severity | Critical |
| Automatic Recovery | ❌ Manually initiated |
| RTO | < 5 minutes |
| Impact | Potential misuse until rotation |

**Procedure**:
1. **Emergency Rotation** (post-MVP, M7):
   ```bash
   kubectl annotate keyprofile <name> openukr.io/emergency-rotate=compromised
   ```
2. **Until M7 is available**: Manually delete the secret → controller regenerates
   ```bash
   kubectl delete secret <secret-name> -n <namespace>
   ```
3. Notify affected services about the new key
4. Create incident report (preserve audit logs)

### 3.4 Controller Image Corruption

| Property | Value |
|---|---|
| Severity | Medium |
| Automatic Recovery | ❌ Manual |
| RTO | < 10 minutes |
| Impact | Controller fails to start |

**Procedure**:
1. Identify a known-good image
2. `helm upgrade openukr charts/openukr --set image.tag=<known-good>`
3. Validate preflight checks (review log output)

## 4. Prerequisites

| Prerequisite | Responsibility | Status |
|---|---|---|
| etcd encryption-at-rest | Cluster Admin | Required |
| etcd backup (every 15 min) | Cluster Admin | Required |
| Leader election enabled | openUKR Helm Chart | Default: true |
| `kubectl` access | Operations | Required |
| Access to container registry | Operations | Required |

## 5. Recovery Test Plan

| Test | Frequency | Procedure |
|---|---|---|
| Controller pod kill | Monthly | `kubectl delete pod` → verify recovery < 30s |
| CRD restoration | Quarterly | etcd backup restore on staging → verify KeyProfiles |
| Emergency rotation | Quarterly | Test KeyProfile with emergency annotation (from M7) |
| Full DR test | Annually | `make test-restore` on isolated kind cluster (from M9) |

## 6. Communication

| Event | Recipients | SLA |
|---|---|---|
| Controller outage > 5 min | Dev team | Automatic via alert |
| etcd data loss | CISO + Dev team | Within 1h |
| Key compromise | CISO + Dev team + affected parties | Immediately |

## 7. Document History

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-02-13 | openUKR | Initial version (S0.8) |
