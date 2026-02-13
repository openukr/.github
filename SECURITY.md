# openUKR Security

<!-- ──────────────────────────────────────────────────────────────── -->
<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->
<!-- ──────────────────────────────────────────────────────────────── -->

> [!CAUTION]
> **Dieses Projekt befindet sich in aktiver Entwicklung und wurde noch nicht vollständig getestet oder auditiert.**
> Es existiert kein offizielles Release. Bitte nicht in Produktionsumgebungen einsetzen.

<!-- END PRE-RELEASE BANNER -->

Sicherheit hat bei openUKR höchste Priorität. Dieses Dokument beschreibt unsere Sicherheitsrichtlinien, Meldeverfahren und Härtungsmaßnahmen.

## 1. Sicherheitsarchitektur

openUKR ist nach dem Prinzip **Security by Design** entworfen:

- **Minimal Privileges**: Container laufen als Non-Root (UID 1000), Read-Only Filesystem, Drop All Capabilities, Seccomp RuntimeDefault.
- **Crypto-Hardening**: Verwendung von Go Standard-Crypto (BSI TR-02102-1 konform). Key-Material wird im Speicher überschrieben (`Wipe()`).
- **Integrität**: Jeder Schlüssel erhält einen SHA-256 Fingerprint im CRD-Status.
- **Isolation**: Strenge Trennung von Namespaces via Webhook-Validierung.
- **Auditierung**: Kritische Operationen erzeugen strukturierte Logs.
- **Path Traversal Protection**: Filesystem Publisher erzwingt absolute Pfade und verwirft `..`-Segmente.
- **HTTPS Enforcement**: HTTP Publisher erzwingt HTTPS, sofern nicht explizit deaktiviert.
- **Entropy Preflight**: Controller startet nur, wenn `crypto/rand` funktional ist.

## 2. Compliance & Standards

openUKR unterstützt die Einhaltung folgender Standards:

- **BSI IT-Grundschutz** (APP.4.4 Kubernetes, SYS.1.6 Container)
- **ISO/IEC 27001** (A.10 Kryptographie, A.12 Betriebssicherheit)
- **NIST SP 800-57** (Key Management Recommendations)
- **DSGVO** (Art. 32 Sicherheit der Verarbeitung)

Für Details zur Konfiguration siehe [Compliance Guide](./docs/operations/COMPLIANCE_GUIDE.md).

## 3. Härtungsmaßnahmen

### Container
- Base Image: `gcr.io/distroless/static:nonroot`
- Keine Shell, keine Paketmanager im Image.
- Binaries sind stripped (`-s -w`) und ohne Pfadinformationen (`-trimpath`).
- Seccomp Profil: `RuntimeDefault`

### Kommunikation
- Controller ↔ K8s API: TLS 1.2+ (de facto TLS 1.3)
- Webhook: TLS 1.2+ (via cert-manager)
- HTTP Publisher: HTTPS erzwungen, Response Body auf 1 MB begrenzt
- HTTP/2 deaktiviert (CVE-2023-44487 Mitigation)

### Speicher
- Secrets: Standardspeicherung in Kubernetes Secrets (etcd Encryption-at-Rest wird vorausgesetzt).
- Keine persistenten Volumes für Key-Material im Controller.
- Private Key-Material wird nach Nutzung im Speicher überschrieben (`Wipe()`).
- Dateien auf Filesystem: `0444` (read-only), Verzeichnisse: `0750`

## 4. Schwachstellen melden

> [!IMPORTANT]
> **Bitte öffnen Sie KEIN öffentliches Issue für Sicherheitslücken.**

Wenn Sie eine Sicherheitslücke in openUKR entdecken, melden Sie diese bitte vertraulich:

1. **E-Mail**: Senden Sie eine detaillierte Beschreibung an die Projektverantwortlichen (Kontaktdaten werden bei Veröffentlichung auf GitHub hinterlegt).
2. **Inhalt**: Beschreiben Sie den Angriffsvektor, betroffene Komponenten und (falls möglich) einen Proof of Concept.
3. **Reaktionszeit**: Wir bestätigen den Empfang innerhalb von 48 Stunden und streben eine Behebung innerhalb von 14 Tagen an.

## 5. Security Updates

Sicherheitsupdates werden über neue Container-Images und Helm-Chart-Versionen bereitgestellt.

### Versionierung
- **Patch-Releases** (z.B. 1.0.1): Enthalten Bugfixes und Security-Patches. Sofortiges Update empfohlen.
- **Minor-Releases** (z.B. 1.1.0): Neue Features, kompatibel.
- **Major-Releases** (z.B. 2.0.0): Breaking Changes, Migrationspfad dokumentiert.

## 6. Dokumentenhistorie

| Version | Datum | Änderung |
|---|---|---|
| 1.1 | 2026-02-13 | Path Traversal + HTTPS Enforcement, Vulnerability Reporting, Pre-Release Hinweis |
| 1.0 | 2026-02-13 | Initiale Version |
