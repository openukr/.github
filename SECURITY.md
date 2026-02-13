# openUKR Security

<!-- ──────────────────────────────────────────────────────────────── -->
<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->
<!-- ──────────────────────────────────────────────────────────────── -->

> [!CAUTION]
> **This project is under active development and has not yet been fully tested or audited.**
> No official release exists. Do not use in production environments.

<!-- END PRE-RELEASE BANNER -->

Security is the highest priority for openUKR. This document describes our security policies, vulnerability reporting procedures, and hardening measures.

## 1. Security Architecture

openUKR is designed following the principle of **Security by Design**:

- **Minimal Privileges**: Containers run as non-root (UID 1000), read-only filesystem, drop all capabilities, Seccomp RuntimeDefault.
- **Crypto Hardening**: Uses Go standard crypto library (BSI TR-02102-1 compliant). Key material is overwritten in memory (`Wipe()`).
- **Integrity**: Every key receives a SHA-256 fingerprint in the CRD status.
- **Isolation**: Strict namespace separation enforced via webhook validation.
- **Auditing**: Critical operations produce structured logs.
- **Path Traversal Protection**: Filesystem Publisher enforces absolute paths and rejects `..` segments.
- **HTTPS Enforcement**: HTTP Publisher enforces HTTPS unless explicitly disabled.
- **Entropy Preflight**: Controller only starts when `crypto/rand` is functional.

## 2. Compliance & Standards

openUKR supports compliance with the following standards:

- **BSI IT-Grundschutz** (APP.4.4 Kubernetes, SYS.1.6 Containers)
- **ISO/IEC 27001** (A.10 Cryptography, A.12 Operations Security)
- **NIST SP 800-57** (Key Management Recommendations)
- **GDPR** (Art. 32 Security of Processing)

For configuration details, see the [Compliance Guide](./docs/operations/COMPLIANCE_GUIDE.md).

## 3. Hardening Measures

### Container
- Base Image: `gcr.io/distroless/static:nonroot`
- No shell, no package managers in the image.
- Binaries are stripped (`-s -w`) and built without path information (`-trimpath`).
- Seccomp Profile: `RuntimeDefault`

### Communication
- Controller ↔ K8s API: TLS 1.2+ (de facto TLS 1.3)
- Webhook: TLS 1.2+ (via cert-manager)
- HTTP Publisher: HTTPS enforced, response body limited to 1 MB
- HTTP/2 disabled (CVE-2023-44487 mitigation)

### Storage
- Secrets: Stored in Kubernetes Secrets (etcd encryption-at-rest is required).
- No persistent volumes for key material in the controller.
- Private key material is overwritten in memory after use (`Wipe()`).
- Files on filesystem: `0444` (read-only), directories: `0750`

## 4. Reporting Vulnerabilities

> [!IMPORTANT]
> **Please do NOT open a public issue for security vulnerabilities.**

If you discover a security vulnerability in openUKR, please report it confidentially:

1. **Email**: Send a detailed description to the project maintainers (contact details will be published on GitHub upon release).
2. **Content**: Describe the attack vector, affected components, and (if possible) a proof of concept.
3. **Response Time**: We will acknowledge receipt within 48 hours and aim to resolve the issue within 14 days.

## 5. Security Updates

Security updates are delivered via new container images and Helm chart versions.

### Versioning
- **Patch Releases** (e.g., 1.0.1): Contain bugfixes and security patches. Immediate update recommended.
- **Minor Releases** (e.g., 1.1.0): New features, backward compatible.
- **Major Releases** (e.g., 2.0.0): Breaking changes, migration path documented.

## 6. Document History

| Version | Date | Change |
|---|---|---|
| 1.1 | 2026-02-13 | Path traversal + HTTPS enforcement, vulnerability reporting, pre-release notice |
| 1.0 | 2026-02-13 | Initial version |
