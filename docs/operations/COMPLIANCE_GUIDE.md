# openUKR — Compliance Guide

<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->

> [!CAUTION]
> **Dieses Dokument beschreibt die geplante Compliance-Architektur. openUKR wurde noch nicht vollständig getestet oder auditiert. Nicht in Produktion einsetzen.**

<!-- END PRE-RELEASE BANNER -->

> **Compliance References**: G-2, G-9 · BSI APP.4.4 · ISO/IEC 27001 · DSGVO Art. 32 · NIST SP 800-57

## 1. Zweck

Dieses Dokument beschreibt die **Voraussetzungen und Konfigurationen**, die von der Betriebsumgebung erfüllt werden müssen, um einen **compliance-konformen Betrieb** von openUKR sicherzustellen. Es referenziert direkt die Anforderungen aus BSI IT-Grundschutz, ISO/IEC 27001, DSGVO, NIS2 und DORA.

> [!IMPORTANT]
> **openUKR allein kann nicht alle Compliance-Anforderungen erfüllen.** Einige Maßnahmen erfordern Konfiguration auf Cluster- oder Organisations-Ebene.

## 2. Verschlüsselung im Ruhezustand (Encryption at Rest)

### Anforderung
- **BSI APP.4.4.A9**: Verschlüsselung von etcd
- **DSGVO Art. 32 Abs. 1a**: Verschlüsselung personenbezogener Daten

### Konfiguration

etcd Encryption-at-Rest **MUSS** aktiviert sein, bevor openUKR in Produktion geht:

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
# kube-apiserver Flag
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

### Verifikation

```bash
# Prüfen ob Secrets verschlüsselt gespeichert sind
kubectl get secrets -n openukr-system -o yaml | head -1
# Wenn 'k8s:enc:aescbc:...' in etcd → verschlüsselt
```

**openUKR Preflight-Check**: Der Controller wird den Verschlüsselungsstatus als Prometheus-Metrik `openukr_compliance_score` melden (siehe [Roadmap Phase 7](../ROADMAP.md#phase-7--compliance--certification)).

## 3. Zeitsynchronisation (NTP)

### Anforderung
- **BSI APP.4.4.A14**: Zeitsynchronisation für Audit-Logs
- **NIST SP 800-57**: Korrekte Zeitstempel für Key-Lifecycle

### Begründung

openUKR verwendet Zeitstempel für:
- `lastRotation` / `nextRotation` im CRD Status
- Key-ID-Generierung (`{alg}-{param}-{YYYYMMDD}-{6hex}`)
- Audit-Log-Einträge
- Grace Period-Berechnung

Zeitsynchronisation auf < 1 Sekunde Genauigkeit ist **PFLICHT**.

### Konfiguration

```bash
# chrony oder systemd-timesyncd auf allen Nodes
timedatectl status
# System clock synchronized: yes
# NTP service: active
```

Kubernetes Nodes müssen NTP-synchronisiert sein. Dies ist eine Cluster-Level-Verantwortung.

## 4. Datenschutz-Folgenabschätzung (DSFA)

### Anforderung
- **DSGVO Art. 35**: DSFA bei hohem Risiko für Rechte und Freiheiten

### Anwendbarkeit

Eine DSFA ist **erforderlich**, wenn openUKR Schlüssel verwaltet, die zum Schutz personenbezogener Daten eingesetzt werden. Dies ist wahrscheinlich der Fall, wenn:

1. API-Schlüssel Zugang zu Systemen mit personenbezogenen Daten ermöglichen
2. Signier-Schlüssel für Authentifizierungs-Tokens (JWT) verwendet werden
3. Verschlüsselungs-Schlüssel für Kundendaten eingesetzt werden

### Inhalt der DSFA

| Abschnitt | Inhalt |
|---|---|
| Verarbeitungszweck | Automatisierte Rotation kryptographischer Schlüssel |
| Rechtsgrundlage | Art. 6 Abs. 1f DSGVO (berechtigtes Interesse an IT-Sicherheit) |
| Risikobeschreibung | Key-Kompromittierung, -Verlust, -Missbrauch |
| Technische Maßnahmen | Tabelle unten |
| Organisatorische Maßnahmen | Zugangskontrollen, Vier-Augen-Prinzip |

### Technische Maßnahmen (openUKR)

| Maßnahme | Umsetzung | Status |
|---|---|---|
| Verschlüsselung im Ruhezustand | etcd Encryption-at-Rest | Voraussetzung |
| Schlüsselrotation | Automatisiert via KeyProfile CRD | ✅ Implementiert |
| Zugriffskontrolle | K8s RBAC, Zwei-Stufen-RBAC | ✅ Geplant (siehe [Roadmap Phase 4](../ROADMAP.md#phase-4--observability--operations)) |
| Integritätsprüfung | Fingerprint-Verifikation | ✅ Implementiert |
| Audit-Trail | Structured JSON Logs | ✅ Geplant (siehe [Roadmap Phase 4](../ROADMAP.md#41-structured-audit-events)) |
| Löschung (Recht auf Vergessen) | Key-Cleanup nach Grace Period | ✅ Geplant (siehe [Roadmap Phase 3](../ROADMAP.md#31-grace-period-implementation)) |
| Transportverschlüsselung | mTLS für HTTP Publisher | ✅ Geplant (siehe [Roadmap Phase 3](../ROADMAP.md#32-mtls-for-http-publisher)) |
| Memory-Wiping | `Wipe()` nach Key-Generierung | ✅ Implementiert |

### Template — DSFA-Verweis

> Die Datenschutz-Folgenabschätzung für den Einsatz von openUKR muss durch den Verantwortlichen (Art. 4 Nr. 7 DSGVO) durchgeführt werden. openUKR stellt die technischen Maßnahmen bereit; organisatorische Maßnahmen liegen in der Verantwortung der einsetzenden Organisation.

## 5. RBAC Best Practices

### Anforderung
- **BSI APP.4.4.A7**: Least Privilege für ServiceAccounts
- **NIST SP 800-57**: Zugriffskontrolle für Key-Management-Systeme

### Empfehlungen

| Prinzip | Empfehlung |
|---|---|
| Namespace-Isolation | Jeder KeyProfile-Namespace erhält eigene Role |
| ServiceAccount pro Funktion | Controller, Webhook, Monitoring: separate SAs |
| Kein `cluster-admin` | openUKR benötigt NUR ClusterRole für CRDs + TokenReview |
| Secret-Zugriff einschränken | `resourceNames` filtern update-Zugriff auf spezifische Secrets |

### Verantwortungsmatrix

| Kompetenz | openUKR Controller | Cluster-Admin | App-Team |
|---|---|---|---|
| CRD lesen/schreiben | ✅ | — | — |
| Secrets erzeugen (im Namespace) | ✅ | — | — |
| Secrets lesen | — | — | ✅ |
| RBAC konfigurieren | — | ✅ | — |
| KeyProfile erstellen | — | — | ✅ |

## 6. Container-Härtung

### Anforderung
- **BSI SYS.1.6**: Container-Härtung
- **CIS Kubernetes Benchmark 5.7**: Seccomp

### openUKR Defaults (im Helm Chart)

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

### Prüfung

```bash
# Verifizieren, dass der Pod als nonroot läuft
kubectl get pod -n openukr-system -o jsonpath='{.items[0].spec.securityContext}'
# Erwartung: runAsNonRoot=true, runAsUser=65532
```

## 7. Netzwerk-Isolation

### Anforderung
- **BSI NET.1.1**: Netzwerksegmentierung

### openUKR NetworkPolicy

openUKR liefert NetworkPolicies in `config/network-policy/`:

| Policy | Beschreibung |
|---|---|
| `allow-metrics-traffic.yaml` | Nur Pods mit Label `metrics: enabled` dürfen Metriken abrufen |
| `allow-webhook-traffic.yaml` | Nur K8s API-Server darf den Webhook kontaktieren |

## 8. Audit-Log-Anforderungen

### Anforderung
- **BSI OPS.1.1.5**: Protokollierung
- **NIS2 Art. 21**: Cybersecurity Risk Management

### Pflicht-Events

openUKR loggt folgende Events als strukturiertes JSON:

| Event | Beschreibung | Severity |
|---|---|---|
| `key.generated` | Neuer Schlüssel erzeugt | INFO |
| `key.rotated` | Rotation abgeschlossen | INFO |
| `key.published` | Schlüssel veröffentlicht | INFO |
| `key.revoked` | Alter Schlüssel gelöscht | INFO |
| `key.emergency_rotation` | Notfall-Rotation | WARN |
| `integrity.violation` | Secret-Manipulation erkannt | ERROR |
| `validation.rejected` | Webhook lehnt Konfiguration ab | WARN |
| `preflight.failed` | Startup-Check fehlgeschlagen | ERROR |
| `publish.failed` | Publishing fehlgeschlagen | ERROR |
| `rotation.overdue` | Rotation überfällig | WARN |
| `algorithm.deprecated` | Veralteter Algorithmus | WARN |
| `config.invalid` | Ungültige Konfiguration | ERROR |

### Aufbewahrungsfristen

| Norm | Mindest-Aufbewahrung |
|---|---|
| DSGVO Art. 5 Abs. 1e | So kurz wie nötig |
| BSI OPS.1.1.5 | 90 Tage (empfohlen) |
| NIS2 | 12 Monate (empfohlen) |
| DORA | 5 Jahre (Finanzsektor) |

**Empfehlung**: 12 Monate Aufbewahrung, danach archivieren gemäß organisatorischer Richtlinie.

## 9. Checkliste für die Inbetriebnahme

| # | Prüfung | Verantwortung | ☐ |
|---|---|---|---|
| 1 | etcd Encryption-at-Rest aktiviert | Cluster-Admin | |
| 2 | NTP auf allen Nodes synchronisiert | Cluster-Admin | |
| 3 | cert-manager installiert + funktional | Cluster-Admin | |
| 4 | NetworkPolicies angewendet | Cluster-Admin | |
| 5 | DSFA durchgeführt (falls anwendbar) | Datenschutzbeauftragter | |
| 6 | RBAC-Rollen geprüft | Cluster-Admin | |
| 7 | Audit-Log-Weiterleitung konfiguriert (SIEM) | Operations | |
| 8 | Monitoring + Alerting aktiv | Operations | |
| 9 | DR-Plan gelesen + verstanden | Operations | |
| 10 | Backup-Strategie für etcd dokumentiert | Cluster-Admin | |

## 10. Dokumentenhistorie

| Version | Datum | Autor | Änderung |
|---|---|---|---|
| 1.0 | 2026-02-13 | openUKR | Initiale Version (S0.8) |
