# Contributing to openUKR

<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->

> [!CAUTION]
> **openUKR is under active development and has not yet been tested or audited.
> Contributions are welcome, but please note that APIs and project structure may still change.**

<!-- END PRE-RELEASE BANNER -->

We welcome contributions! Please follow these guidelines to get started.

## Development Setup

1. **Go 1.22+** required.
2. **Kubernetes Environment**: `kind` or `minikube` is recommended for local development.

## Making Changes

1. Create a feature branch (`git checkout -b feature/my-feature`).
2. Implement your changes.
3. Run tests:
   ```bash
   make test
   ```
4. Commit with descriptive messages (Conventional Commits preferred).
   ```bash
   git commit -m "feat(controller): add support for RSA-4096"
   ```
5. Submit a Pull Request.

## Code Style

- We use standard `gofmt`.
- We use `golangci-lint` (see `.golangci.yml`).

## Security

If you find a security vulnerability, please refer to our [Security Policy](SECURITY.md). Do **NOT** open a public issue.
