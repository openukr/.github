# openUKR &middot; Universal Key Rotation

> **Secure, compliant, and automated key management for Kubernetes.**

<p align="center">
  <img src="https://img.shields.io/badge/Status-Alpha-orange?style=flat-square" alt="Status" />
  <img src="https://img.shields.io/badge/Kubernetes-Native-326ce5?style=flat-square&logo=kubernetes&logoColor=white" alt="K8s Native" />
  <img src="https://img.shields.io/badge/License-Apache_2.0-green?style=flat-square" alt="License" />
  <img src="https://img.shields.io/badge/Compliance-BSI_TR--02102--1-blue?style=flat-square" alt="Compliance" />
</p>

<!-- ──────────────────────────────────────────────────────────────── -->
<!-- PRE-RELEASE BANNER — Remove this entire block once v1.0.0 is  -->
<!-- officially tested, security-audited and released.              -->
<!-- ──────────────────────────────────────────────────────────────── -->

> [!CAUTION]
> **⚠️ Pre-Release — Not suitable for production use.**
>
> openUKR is under active development. The code has not yet been fully tested or security-audited. No official release exists.

<!-- END PRE-RELEASE BANNER -->

## 🚀 Mission

**openUKR** is dedicated to solving the complex challenge of managing cryptographic identities in distributed systems. We believe that security shouldn't be a manual burden. By automating the entire lifecycle of cryptographic keys—generation, rotation, distribution, and publication—we enable organizations to maintain high security standards (like **BSI TR-02102-1** and **NIST SP 800-57**) without operational overhead.

---

## 🛠️ Core Project

### [openUKR Operator](https://github.com/openukr/openukr)
The heart of our ecosystem is the **openUKR Operator**, a Kubernetes-native controller that manages `KeyProfile` resources.

**Key Capabilities:**
- **Automated Rotation:** Scheduled key updates with zero downtime.
- **Graceful Rollover:** Support for overlapping key validity to ensure service continuity.
- **Compliance First:** Built-in validation against cryptographic standards (e.g., rejecting weak algorithms).
- **Flexible Publishing:** Distribute public keys via HTTP, filesystem, or directly to Kubernetes Secrets.
- **Observability:** Prometheus metrics for rotation status, errors, and key age monitoring.

---

## 🔒 Security & Compliance

Security is not an afterthought; it is the foundation.
- **Strict Defaults:** We enforce secure algorithms by default.
- **Transparency:** All key operations are auditable via Kubernetes events and status fields.
- **Policy Enforcement:** Prevent accidental misconfigurations through validating webhooks.

For security reports, please see our [Security Policy](https://github.com/openukr/.github/blob/main/SECURITY.md).

---

## 📚 Documentation

| Document | Description |
|---|---|
| [Roadmap](https://github.com/openukr/.github/blob/main/ROADMAP.md) | Planned features and version milestones |
| [Compliance Guide](https://github.com/openukr/.github/blob/main/docs/operations/COMPLIANCE_GUIDE.md) | Operational requirements (etcd encryption, NTP, DPIA) |
| [Disaster Recovery](https://github.com/openukr/.github/blob/main/docs/operations/DISASTER_RECOVERY.md) | RTO/RPO definitions and recovery procedures |
| [Security Policy](https://github.com/openukr/.github/blob/main/SECURITY.md) | Hardening details, vulnerability reporting, and compliance mapping |
| [Contributing](https://github.com/openukr/.github/blob/main/CONTRIBUTING.md) | Contribution guidelines |

---

## 🤝 Get Involved

We are an open community and welcome contributions from everyone. Whether you're fixing a bug, improving documentation, or proposing a new feature, your help is appreciated!

- **Contributing:** Check out our [Contribution Guidelines](https://github.com/openukr/.github/blob/main/CONTRIBUTING.md).
- **Roadmap:** See our [Roadmap](https://github.com/openukr/.github/blob/main/ROADMAP.md) for planned features and milestones.

---

<p align="center">
  <sub>Built with ❤️ by the openUKR Community.</sub>
</p>
