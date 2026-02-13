# openUKR — Disaster Recovery Plan

<!-- PRE-RELEASE BANNER — Remove this block once v1.0.0 is released -->

> [!CAUTION]
> **Dieses Dokument beschreibt geplante Verfahren. openUKR wurde noch nicht vollständig getestet. Nicht in Produktion einsetzen.**

<!-- END PRE-RELEASE BANNER -->

> **Standards**: BSI CON.1 · ISO/IEC 27001 A.5.29, A.5.30 · NIST SP 800-34

## 1. Vorbemerkung

openUKR ist ein **Tier-0 Dienst** — ein Ausfall betrifft alle Dienste, die auf asymmetrische Schlüssel angewiesen sind. Dieses Dokument definiert Recovery-Ziele, -Verfahren und -Testpläne.

## 2. Recovery-Ziele

| Metrik | Ziel | Begründung |
|---|---|---|
| **RTO** (Recovery Time Objective) | **< 15 Minuten** | Controller-Neustart + Key-Generation |
| **RPO** (Recovery Point Objective) | **0** (kein Key-Verlust) | Keys sind in K8s Secrets gespeichert — unabhängig vom Controller |
| **MTTR** (Mean Time to Recovery) | **< 10 Minuten** | Automatisiert via Kubernetes Deployment |

## 3. Schadensszenarien

### 3.1 Controller-Pod-Ausfall

| Eigenschaft | Wert |
|---|---|
| Schwere | Niedrig |
| Automatische Recovery | ✅ Kubernetes Deployment Restart |
| RTO | < 30 Sekunden |
| Auswirkung | Rotationen verzögert, bestehende Keys weiterhin gültig |

**Verfahren**: Keine manuelle Aktion nötig. Kubernetes startet den Pod neu. Leader Election stellt sicher, dass nur ein Controller aktiv ist.

### 3.2 etcd-Datenverlust

| Eigenschaft | Wert |
|---|---|
| Schwere | Kritisch |
| Automatische Recovery | ❌ Manuell |
| RTO | < 15 Minuten |
| Auswirkung | Alle CRDs + Secrets verloren |

**Verfahren**:
1. etcd-Backup wiederherstellen (Cluster-Operations)
2. `kubectl get keyprofiles -A` — Prüfen, ob CRDs wiederhergestellt
3. Controller neustart — wird fehlende Secrets neu generieren
4. Key-Inventar prüfen: `kubectl get cm openukr-key-inventory -n openukr-system`

**Prävention**:
- etcd-Backups alle 15 Minuten (Cluster-Level-Verantwortung)
- Encryption-at-Rest für etcd aktiviert (Pflicht, siehe [Compliance Guide](COMPLIANCE_GUIDE.md))

### 3.3 Schlüsselkompromittierung

| Eigenschaft | Wert |
|---|---|
| Schwere | Kritisch |
| Automatische Recovery | ❌ Manuell initiiert |
| RTO | < 5 Minuten |
| Auswirkung | Potentieller Missbrauch bis zur Rotation |

**Verfahren**:
1. **Emergency Rotation** (Post-MVP, M7):
   ```bash
   kubectl annotate keyprofile <name> openukr.io/emergency-rotate=compromised
   ```
2. **Bis M7 verfügbar**: Secret manuell löschen → Controller regeneriert
   ```bash
   kubectl delete secret <secret-name> -n <namespace>
   ```
3. Betroffene Dienste über neuen Schlüssel informieren
4. Incident-Report erstellen (Audit-Log sichern)

### 3.4 Controller-Image-Korruption

| Eigenschaft | Wert |
|---|---|
| Schwere | Mittel |
| Automatische Recovery | ❌ Manuell |
| RTO | < 10 Minuten |
| Auswirkung | Controller startet nicht |

**Verfahren**:
1. Bekanntes gutes Image identifizieren
2. `helm upgrade openukr charts/openukr --set image.tag=<known-good>`
3. Preflight-Checks validieren (Logausgabe prüfen)

## 4. Voraussetzungen

| Voraussetzung | Verantwortung | Status |
|---|---|---|
| etcd Encryption-at-Rest | Cluster-Admin | Pflicht |
| etcd-Backup (alle 15 Min) | Cluster-Admin | Pflicht |
| Leader Election aktiviert | openUKR Helm Chart | Default: true |
| `kubectl` Zugang | Operations | Pflicht |
| Zugang zu Container Registry | Operations | Pflicht |

## 5. Recovery-Test-Plan

| Test | Frequenz | Verfahren |
|---|---|---|
| Controller-Pod-Kill | Monatlich | `kubectl delete pod` → Recovery < 30s prüfen |
| CRD-Wiederherstellung | Quartalsweise | etcd-Backup restore auf Staging → KeyProfiles prüfen |
| Emergency Rotation | Quartalsweise | Test-KeyProfile mit Emergency-Annotation (ab M7) |
| Vollständiger DR-Test | Jährlich | `make test-restore` auf isoliertem Kind-Cluster (ab M9) |

## 6. Kommunikation

| Ereignis | Empfänger | SLA |
|---|---|---|
| Controller-Ausfall > 5 Min | Dev-Team | Automatisch via Alert |
| etcd-Datenverlust | CISO + Dev-Team | Innerhalb 1h |
| Key-Kompromittierung | CISO + Dev-Team + Betroffene | Sofort |

## 7. Dokumentenhistorie

| Version | Datum | Autor | Änderung |
|---|---|---|---|
| 1.0 | 2026-02-13 | openUKR | Initiale Version (S0.8) |
