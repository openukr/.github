# openUKR — Compliance Guide

<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->

> [!CAUTION]
> **This document describes the planned compliance architecture. openUKR has not yet been fully tested or audited. Do not use in production.**

<!-- END PRE-RELEASE BANNER -->

> **Compliance References**: G-2, G-9 · BSI APP.4.4 · ISO/IEC 27001 · GDPR Art. 32 · NIST SP 800-57

## 1. Purpose

This document describes the **prerequisites and configurations** that must be met by the operating environment to ensure **compliance-conformant operation** of openUKR. It directly references requirements from BSI IT-Grundschutz, ISO/IEC 27001, GDPR, NIS2, and DORA.

> [!IMPORTANT]
> **openUKR alone cannot fulfill all compliance requirements.** Some measures require configuration at the cluster or organization level.

## 2. Encryption at Rest

### Requirement
- **BSI APP.4.4.A9**: Encryption of etcd
- **GDPR Art. 32 (1a)**: Encryption of personal data

### Configuration

etcd encryption-at-rest **MUST** be enabled before openUKR goes into production:

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

```bash
# kube-apiserver flag
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Verification

```bash
# Check whether secrets are stored encrypted
kubectl get secrets -n openukr-system -o yaml | head -1
# If 'k8s:enc:aescbc:...' appears in etcd → encrypted
```

**openUKR Preflight Check**: The controller will report encryption status as a Prometheus metric `openukr_compliance_score` (see [Roadmap Phase 7](../ROADMAP.md#phase-7--compliance--certification)).

## 3. Time Synchronization (NTP)

### Requirement
- **BSI APP.4.4.A14**: Time synchronization for audit logs
- **NIST SP 800-57**: Correct timestamps for key lifecycle

### Rationale

openUKR uses timestamps for:
- `lastRotation` / `nextRotation` in CRD status
- Key ID generation (`{alg}-{param}-{YYYYMMDD}-{6hex}`)
- Audit log entries
- Grace period calculation

Time synchronization to < 1 second accuracy is **REQUIRED**.

### Configuration

```bash
# chrony or systemd-timesyncd on all nodes
timedatectl status
# System clock synchronized: yes
# NTP service: active
```

Kubernetes nodes must be NTP-synchronized. This is a cluster-level responsibility.

## 4. Data Protection Impact Assessment (DPIA)

### Requirement
- **GDPR Art. 35**: DPIA required when processing poses high risk to rights and freedoms

### Applicability

A DPIA is **required** when openUKR manages keys used to protect personal data. This is likely the case when:

1. API keys grant access to systems containing personal data
2. Signing keys are used for authentication tokens (JWT)
3. Encryption keys are used for customer data

### DPIA Content

| Section | Content |
|---|---|
| Processing Purpose | Automated rotation of cryptographic keys |
| Legal Basis | Art. 6(1)(f) GDPR (legitimate interest in IT security) |
| Risk Description | Key compromise, loss, or misuse |
| Technical Measures | Table below |
| Organizational Measures | Access controls, four-eyes principle |

### Technical Measures (openUKR)

| Measure | Implementation | Status |
|---|---|---|
| Encryption at rest | etcd encryption-at-rest | Prerequisite |
| Key rotation | Automated via KeyProfile CRD | ✅ Implemented |
| Access control | K8s RBAC, two-tier RBAC | ✅ Planned (see [Roadmap Phase 4](../ROADMAP.md#phase-4--observability--operations)) |
| Integrity verification | Fingerprint verification | ✅ Implemented |
| Audit trail | Structured JSON logs | ✅ Planned (see [Roadmap Phase 4](../ROADMAP.md#41-structured-audit-events)) |
| Deletion (right to erasure) | Key cleanup after grace period | ✅ Planned (see [Roadmap Phase 3](../ROADMAP.md#31-grace-period-implementation)) |
| Transport encryption | mTLS for HTTP Publisher | ✅ Planned (see [Roadmap Phase 3](../ROADMAP.md#32-mtls-for-http-publisher)) |
| Memory wiping | `Wipe()` after key generation | ✅ Implemented |

### Template — DPIA Reference

> The data protection impact assessment for the use of openUKR must be conducted by the data controller (Art. 4(7) GDPR). openUKR provides the technical measures; organizational measures are the responsibility of the deploying organization.

## 5. RBAC Best Practices

### Requirement
- **BSI APP.4.4.A7**: Least privilege for ServiceAccounts
- **NIST SP 800-57**: Access control for key management systems

### Recommendations

| Principle | Recommendation |
|---|---|
| Namespace isolation | Each KeyProfile namespace receives its own Role |
| ServiceAccount per function | Controller, webhook, monitoring: separate SAs |
| No `cluster-admin` | openUKR only requires ClusterRole for CRDs + TokenReview |
| Restrict secret access | Use `resourceNames` to filter update access to specific secrets |

### Responsibility Matrix

| Capability | openUKR Controller | Cluster Admin | App Team |
|---|---|---|---|
| Read/write CRDs | ✅ | — | — |
| Create secrets (in namespace) | ✅ | — | — |
| Read secrets | — | — | ✅ |
| Configure RBAC | — | ✅ | — |
| Create KeyProfiles | — | — | ✅ |

## 6. Container Hardening

### Requirement
- **BSI SYS.1.6**: Container hardening
- **CIS Kubernetes Benchmark 5.7**: Seccomp

### openUKR Defaults (in Helm Chart)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  seccompProfile:
    type: RuntimeDefault
  capabilities:
    drop:
      - ALL
```

### Verification

```bash
# Verify that the pod runs as non-root
kubectl get pod -n openukr-system -o jsonpath='{.items[0].spec.securityContext}'
# Expected: runAsNonRoot=true, runAsUser=65532
```

## 7. Network Isolation

### Requirement
- **BSI NET.1.1**: Network segmentation

### openUKR NetworkPolicy

openUKR ships NetworkPolicies in `config/network-policy/`:

| Policy | Description |
|---|---|
| `allow-metrics-traffic.yaml` | Only pods with label `metrics: enabled` may scrape metrics |
| `allow-webhook-traffic.yaml` | Only the K8s API server may contact the webhook |

## 8. Audit Log Requirements

### Requirement
- **BSI OPS.1.1.5**: Logging
- **NIS2 Art. 21**: Cybersecurity risk management

### Required Events

openUKR logs the following events as structured JSON:

| Event | Description | Severity |
|---|---|---|
| `key.generated` | New key generated | INFO |
| `key.rotated` | Rotation completed | INFO |
| `key.published` | Key published | INFO |
| `key.revoked` | Old key deleted | INFO |
| `key.emergency_rotation` | Emergency rotation | WARN |
| `integrity.violation` | Secret tampering detected | ERROR |
| `validation.rejected` | Webhook rejected configuration | WARN |
| `preflight.failed` | Startup check failed | ERROR |
| `publish.failed` | Publishing failed | ERROR |
| `rotation.overdue` | Rotation overdue | WARN |
| `algorithm.deprecated` | Deprecated algorithm | WARN |
| `config.invalid` | Invalid configuration | ERROR |

### Retention Periods

| Standard | Minimum Retention |
|---|---|
| GDPR Art. 5(1)(e) | As short as necessary |
| BSI OPS.1.1.5 | 90 days (recommended) |
| NIS2 | 12 months (recommended) |
| DORA | 5 years (financial sector) |

**Recommendation**: 12 months retention, then archive according to organizational policy.

## 9. Deployment Checklist

| # | Check | Responsibility | ☐ |
|---|---|---|---|
| 1 | etcd encryption-at-rest enabled | Cluster Admin | |
| 2 | NTP synchronized on all nodes | Cluster Admin | |
| 3 | cert-manager installed and functional | Cluster Admin | |
| 4 | NetworkPolicies applied | Cluster Admin | |
| 5 | DPIA conducted (if applicable) | Data Protection Officer | |
| 6 | RBAC roles reviewed | Cluster Admin | |
| 7 | Audit log forwarding configured (SIEM) | Operations | |
| 8 | Monitoring + alerting active | Operations | |
| 9 | DR plan read and understood | Operations | |
| 10 | Backup strategy for etcd documented | Cluster Admin | |

## 10. Document History

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-02-13 | openUKR | Initial version (S0.8) |
