# openUKR Roadmap

> Living document — last updated: 2026-02-13

<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->

> [!CAUTION]
> **openUKR is under active development. It has not yet been tested or audited. No official release exists.**

<!-- END PRE-RELEASE BANNER -->

This roadmap outlines the planned evolution of openUKR from its current Alpha state
to a production-grade, enterprise-ready key management operator.

---

## Phase 1 — Foundation (✅ Complete)

| Item | Status | Notes |
|---|---|---|
| KeyProfile CRD (v1alpha1) | ✅ | EC + RSA, rotation policy, output config |
| KeyGenerator (EC P-256/384/521, RSA 2048–4096) | ✅ | BSI TR-02102-1 validated |
| SecretWriter (split-pem, single-pem, JKS) | ✅ | OwnerRef, atomic create-or-update |
| Validating Webhook | ✅ | Namespace isolation, key spec, rotation policy |
| Defaulting Webhook | ✅ | PEM encoding, split-pem format defaults |
| Fingerprint Verification (SHA-256) | ✅ | Status field `currentKeyFingerprint` |
| Memory Wipe (`KeyPair.Wipe()`) | ✅ | Zeros raw bytes + big.Int internals |
| Prometheus Metrics | ✅ | Rotations, errors, generation latency |
| Entropy Preflight Check | ✅ | Blocks startup if `crypto/rand` fails |
| HTTP/2 disabled by default | ✅ | CVE-2023-44487 mitigation |

## Phase 2 — Publishing & Packaging (✅ Complete)

| Item | Status | Notes |
|---|---|---|
| Publisher Interface | ✅ | `pkg/publish` package |
| Filesystem Publisher | ✅ | Atomic writes, path traversal protection |
| HTTP Publisher | ✅ | HTTPS enforcement, response body limits |
| Publish-before-Distribute ordering | ✅ | Validators receive keys before services |
| Helm Chart | ✅ | `charts/openukr/` with RBAC, CRDs, security context |
| E2E Test Framework | ✅ | Ginkgo-based, KeyProfile lifecycle test |
| Seccomp Profile (RuntimeDefault) | ✅ | In Helm values |
| Distroless Container Image | ✅ | `gcr.io/distroless/static:nonroot` |

---

## Phase 3 — Enterprise Hardening (Next)

### 3.1 Grace Period Implementation
**Priority: Critical**

Currently, key rotation is instantaneous — old keys are immediately replaced.
For zero-downtime in distributed systems, the previous key must remain valid
during a configurable grace period.

- [ ] Store `previousKeyID` and `previousKeyFingerprint` in Status
- [ ] Include both old + new private keys in Secret during grace period
- [ ] Add `GracePeriod` phase to reconciliation loop
- [ ] Cleanup old key after grace period expiry
- [ ] Add metric: `openukr_grace_period_active`

### 3.2 mTLS for HTTP Publisher
**Priority: High**

The HTTP Publisher currently supports `InsecureSkipVerify` only.
Production use requires full mutual TLS.

- [ ] Load CA certificate from referenced Kubernetes Secret (`caCertSecretRef`)
- [ ] Load client certificate from referenced Secret (`clientCertSecretRef`)
- [ ] Certificate hot-reload via `certwatcher`
- [ ] Add webhook validation: reject HTTP publisher without TLS config (unless explicitly opted out)

### 3.3 Spec Change Detection
**Priority: Medium**

Currently, algorithm changes (e.g., EC → RSA) on an existing KeyProfile
are not automatically detected. Rotation only triggers on time interval.

- [ ] Add `status.algorithm` field to track current key algorithm
- [ ] Compare `spec.keySpec.algorithm` vs `status.algorithm` on reconcile
- [ ] Force immediate rotation when spec changes
- [ ] Emit Kubernetes Event: `AlgorithmChanged`

### 3.4 JKS Password from SecretRef
**Priority: Medium**

JKS format requires a password, but there is currently no way to supply it
via the CRD. Hardcoded or empty passwords are unacceptable.

- [ ] Add `output.passwordSecretRef` to CRD (name + key)
- [ ] Fetch password from referenced Secret in SecretWriter
- [ ] Validate presence when `format: jwks` is selected
- [ ] Document secure password generation guidelines

---

## Phase 4 — Observability & Operations

### 4.1 Structured Audit Events
**Priority: High**

- [ ] Emit Kubernetes Events for: rotation success, rotation failure, grace period start/end, publishing failure
- [ ] Add event reason codes (e.g., `RotationSuccess`, `PublishFailed`, `EntropyCheckFailed`)
- [ ] Include key metadata in event annotations (keyID, algorithm, fingerprint)

### 4.2 Advanced Metrics
**Priority: Medium**

- [ ] `openukr_key_age_seconds` (gauge) — time since last rotation per profile
- [ ] `openukr_publish_duration_seconds` (histogram) — publishing latency
- [ ] `openukr_publish_errors_total` (counter) — publishing failures by target type
- [ ] `openukr_active_profiles` (gauge) — number of managed KeyProfiles
- [ ] Grafana dashboard template in `charts/openukr/dashboards/`

### 4.3 Alerting Rules
**Priority: Medium**

- [ ] PrometheusRule CRD for critical alerts:
  - Key age exceeds 2× rotation interval (rotation stalled)
  - Rotation error rate > threshold
  - Entropy check failure
- [ ] Include in Helm chart as optional component

---

## Phase 5 — Multi-Cluster & Scale

### 5.1 Namespace-Scoped Mode
**Priority: Medium**

- [ ] Add `--watch-namespaces` flag to restrict operator scope
- [ ] Generate `Role` + `RoleBinding` instead of `ClusterRole` when scoped
- [ ] Update Helm chart with `scope: cluster|namespace` value

### 5.2 Multi-Cluster Key Synchronization
**Priority: Low**

- [ ] Design federation protocol for cross-cluster public key distribution
- [ ] Evaluate integration with Liqo, Submariner, or custom CRD sync
- [ ] Add `publishTarget.type: remote-cluster`

### 5.3 Performance Optimization
**Priority: Low**

- [ ] Rate limiting for concurrent key generation (RSA 4096 is CPU-intensive)
- [ ] Workqueue tuning for large-scale deployments (1000+ KeyProfiles)
- [ ] Benchmarking suite

---

## Phase 6 — Ecosystem Integration

### 6.1 JWKS Endpoint Server
**Priority: High**

- [ ] Built-in JWKS endpoint serving aggregated public keys
- [ ] `/.well-known/jwks.json` compatible
- [ ] Integration with OIDC/OAuth2 providers
- [ ] Automatic rotation of JWKS endpoint content

### 6.2 Client Libraries
**Priority: Medium**

- [ ] Go SDK: `openukr-go` — file-watching, automatic key reload
- [ ] Python SDK: `openukr-python` — PEM loading, JWT signing
- [ ] Java SDK: `openukr-java` — JKS/PKCS12 loading
- [ ] Conformance test suite for all SDKs

### 6.3 cert-manager Integration
**Priority: Low**

- [ ] Issuer that provisions X.509 certificates alongside key pairs
- [ ] Use openUKR keys as CA private keys for cert-manager

---

## Phase 7 — Compliance & Certification

### 7.1 Compliance Automation
**Priority: High**

- [ ] Automated compliance reports (BSI TR-02102-1, NIST SP 800-57)
- [ ] CRD validation for maximum key lifetime (`maxInterval`)
- [ ] Audit log export to SIEM systems (Splunk, ELK, Loki)

### 7.2 FIPS 140-2 Support
**Priority: Medium**

- [ ] Build with `GOEXPERIMENT=boringcrypto` for FIPS-validated crypto
- [ ] Separate container image tag: `ghcr.io/openukr/openukr:v1.0.0-fips`
- [ ] Document FIPS deployment requirements

### 7.3 SOC 2 / ISO 27001 Evidence
**Priority: Low**

- [ ] Automated evidence collection for key rotation controls
- [ ] Export rotation history as compliance artifacts
- [ ] Policy-as-code for key lifecycle requirements (OPA/Gatekeeper integration)

---

## Version Milestones

| Version | Target | Focus |
|---|---|---|
| **v0.1.0** (Alpha) | ✅ Q1 2026 | Core rotation, webhooks, publishing |
| **v0.2.0** (Alpha) | Q2 2026 | Grace period, mTLS, spec change detection |
| **v0.5.0** (Beta) | Q3 2026 | JWKS endpoint, audit events, Grafana dashboards |
| **v1.0.0** (GA) | Q4 2026 | Production-ready, FIPS support, client SDKs |
| **v2.0.0** | 2027 | Multi-cluster, cert-manager integration |

---

## Contributing

We welcome contributions to any roadmap item! Please check our
[Contributing Guidelines](CONTRIBUTING.md) and open an issue to discuss
your proposed changes before submitting a PR.

Items marked with high priority are excellent candidates for contribution.
