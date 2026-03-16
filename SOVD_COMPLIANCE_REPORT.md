# SOVD Compliance Report & Implementierungsplan

## Classic Diagnostic Adapter (CDA) — Eclipse OpenSOVD

**Datum:** 2026-03-16  
**Repository:** https://github.com/eclipse-opensovd/classic-diagnostic-adapter  
**Basis:** ISO 17978-1 / ISO 17978-3 (SOVD), interne Requirements-Docs, GitHub Issues, Code-TODOs

---

## 1. Projekt-Überblick

| Eigenschaft | Wert |
|---|---|
| **Sprache** | Rust (Edition 2024, min. 1.88.0) |
| **Architektur** | Modularer Workspace mit 12 Crates |
| **Web-Framework** | Axum + Aide (OpenAPI) |
| **Kommunikation** | DoIP (ISO 13400) → UDS (ISO 14229) |
| **Datenbank** | MDD-Dateien (aus ODX konvertiert) |
| **Unit Tests** | 143 `#[test]` Funktionen |
| **Integrationstests** | Docker-basiert (ECU-Simulator) |
| **API-Pfad-Prefix** | `/vehicle/v15/` |

### Workspace Crates

| Crate | Funktion |
|---|---|
| `cda-main` | Startup, Config, Orchestrierung |
| `cda-sovd` | SOVD REST API (Axum Routes) |
| `cda-sovd-interfaces` | SOVD Datentypen & Traits |
| `cda-core` | Diagnostic Kernel, ECU Manager |
| `cda-interfaces` | Gemeinsame Interfaces & Datentypen |
| `cda-comm-doip` | DoIP Gateway Kommunikation |
| `cda-comm-uds` | UDS Protokoll-Handling |
| `cda-database` | MDD-Datei Parsing & Loading |
| `cda-plugin-security` | Security Plugin System |
| `cda-health` | Health Monitoring |
| `cda-tracing` | Logging & Tracing (DLT, OpenTelemetry) |
| `cda-build` | Build-Utilities |

---

## 2. SOVD Compliance Status (ISO 17978-1 / 17978-3)

### 2.1 Implementierte Features (✅)

| # | SOVD Requirement | ISO Referenz | Implementierung |
|---|---|---|---|
| 1 | **HTTP(S)-Server** | 17978-1 | `cda-sovd/src/lib.rs` — Axum async multi-connection |
| 2 | **Konfigurierbarer Port** | 17978-1 | `opensovd-cda.toml` |
| 3 | **Components Entity Collection** | 17978-3, 5.4 | `GET /vehicle/v15/components` |
| 4 | **ECU Resource Collection** | 17978-3, 5.4.2 | Pro ECU unter `/components/{ecu}` |
| 5 | **Data Resources (SID 22/2E)** | 17978-3 | `GET/PUT /data/{did}` |
| 6 | **Configuration Resources** | 17978-3 | `GET/PUT /configurations/{did}` |
| 7 | **Operations / Routines (SID 31)** | 17978-3 | `POST /operations/{rid}/executions` |
| 8 | **Sync & Async Operations** | 17978-3 | Start/RequestResults/Stop lifecycle |
| 9 | **Faults/DTCs (SID 14/19)** | 17978-3 | `GET/DELETE /faults`, `/faults/{dtc}` |
| 10 | **Session Management (SID 10)** | 17978-3, 7.16 | `GET/PUT /modes/session` |
| 11 | **Security Access (SID 27)** | 17978-3 | `GET/PUT /modes/security` |
| 12 | **Authentication (SID 29)** | 17978-3 | `GET/PUT /modes/authentication` |
| 13 | **Communication Control (SID 28)** | 17978-3 | `GET/PUT /modes/commctrl` |
| 14 | **DTC Setting (SID 85)** | 17978-3 | `GET/PUT /modes/dtcsetting` |
| 15 | **ECU Reset (SID 11)** | 17978-3 | `/operations/reset`, `/operations/ecureset` |
| 16 | **Locks (Vehicle & ECU)** | 17978-3 | `POST/GET/PUT/DELETE /locks` |
| 17 | **OpenAPI Documentation** | 17978-3 | Swagger UI + `/openapi.json` |
| 18 | **include-schema Query** | 17978-3 | `?include-schema=true` auf allen Endpoints |
| 19 | **Case-insensitive Paths** | 17978-1 | URI-Rewriting in Middleware |
| 20 | **Data Type Mapping** | 17978-3 | ODX→JSON Mapping (A_ASCIISTRING, A_FLOAT32, etc.) |
| 21 | **ECU Variant Detection** | 17978-3 | `POST /components/{ecu}` |
| 22 | **ECU State Machine** | 17978-3 | NotTested/Online/Offline/Disconnected/Duplicate/NoVariantDetected |
| 23 | **Tester Present (3E)** | 17978-1 | Intern im CDA gehandelt |
| 24 | **DoIP Gateway Init** | ISO 13400 | VIR Broadcast, VAM, TCP, Routing Activation |
| 25 | **Parallel DB Loading** | - | MDD files parallel geladen |
| 26 | **Deferred Initialization** | - | On-demand / Plugin-triggered |
| 27 | **Flash API (SID 34/36/37)** | Extension | `x-sovd2uds-download/*` Endpoints |
| 28 | **Bulk Data / Flash Files** | Extension | `/apps/sovd2uds/bulk-data/flashfiles` |
| 29 | **Functional Communication** | Extension | `/functions/functionalgroups/{group}` |
| 30 | **ComParam API** | Extension | `/operations/comparam/executions` |
| 31 | **MDD Embedded Files** | Extension | `/x-sovd2uds-bulk-data/mdd-embedded-files` |
| 32 | **Version Endpoint** | Extension | `/apps/sovd2uds/data/version` |
| 33 | **Network Structure** | Extension | `/apps/sovd2uds/data/networkstructure` |
| 34 | **Security Plugin** | Extension | JWT-Validation, `SecurityPlugin` trait |
| 35 | **Health Monitoring** | Extension | Optional Build-Feature, `/health` |
| 36 | **application/octet-stream** | Extension | Binary payload Unterstützung |
| 37 | **A_BYTEFIELD hex Format** | Extension | `?format=hex` Query Parameter |
| 38 | **Generic Service** | Extension | `/genericservice` PUT endpoint |
| 39 | **Single ECU Jobs** | Extension | `/x-single-ecu-jobs` |
| 40 | **Authorization** | 17978-3 | `/vehicle/v15/authorize` |

### 2.2 Teilweise implementiert / Lücken (⚠️)

| # | Feature | Status | Details |
|---|---|---|---|
| 1 | **IOControl (SID 2A)** | ❌ Nicht implementiert | Explizit als "Not supported at this time" markiert |
| 2 | **Complex Data Type Mapping** | ⚠️ TODO | `arch~sovd-api-data-types-mapping-iso17978` — "Mapping of complex data types" offen |
| 3 | **EnvDataDesc DOP Schema** | ⚠️ TODO | `cda-core/schema.rs` — "EnvDataDesc DOPs are not yet supported in JSON Schema" |
| 4 | **DTC DOP Schema** | ⚠️ TODO | `cda-core/schema.rs` — "DTC DOPs are not yet supported in JSON Schema" |
| 5 | **Error Codes & Messages** | ⚠️ TODO | Arch-Doku: `.. todo:: define` |
| 6 | **DoIP NACK Handling (#22)** | ⚠️ TODO | `cda-comm-doip` — Generic NACK nicht vollständig behandelt |
| 7 | **Tester Present Params (#53)** | ⚠️ TODO | 8 ComParams in `UdsComParams` markiert als "todo use this in #53" |
| 8 | **DoIP Retry/NACK Params (#22)** | ⚠️ TODO | 3 ComParams markiert als "todo use this in #22" |
| 9 | **Source Port konfigurierbar** | ⚠️ TODO | `cda-comm-doip/ecu_connection.rs` |
| 10 | **DTC Scope Query Parameter** | ⚠️ Backlog | GitHub Issue #178 |
| 11 | **DTC FLXC Structures** | ⚠️ Backlog | GitHub Issue #177 |
| 12 | **ComParam GET ohne Lock** | ⚠️ TODO | Arch-Doku Konflikt mit Standard offen |
| 13 | **HTTPS/TLS Konfiguration** | ⚠️ Draft | Requirement existiert, HSM-Integration TODO |
| 14 | **DoIP Protokollversionen** | ⚠️ TODO | "Define supported protocol versions" |
| 15 | **Duplicate MDD Resolution** | ⚠️ TODO | "specify how to resolve this behavior" |
| 16 | **Asynchronous Operation GET/DELETE** | ⚠️ Teilweise | `/operations/{service}/executions/{id}` Route für GET/DELETE fehlt in Router |

### 2.3 Nicht implementiert (❌)

| # | Feature | ISO Referenz | Priorität |
|---|---|---|---|
| 1 | **IOControl (SID 2A)** | 17978-3 | Medium |
| 2 | **UDS über CAN (ISO-TP)** | ISO 15765 | Low (Feature Request #195) |
| 3 | **Diagnostic DB Update Plugin** | Requirement | High |
| 4 | **DB Update Authentication** | Requirement | High |
| 5 | **DB Update Verification** | Requirement | Medium |
| 6 | **DB Update Downgrade Protection** | Requirement | Medium |
| 7 | **DB Update Crash Safety** | Requirement | High |
| 8 | **DoIP Plugin (Interception)** | Requirement | Low |
| 9 | **UDS Plugin (Interception)** | Requirement | Low |
| 10 | **Logging Plugins** | Planned | Low |
| 11 | **Safety Plugins** | Planned | Low |
| 12 | **Custom Endpoint Plugins** | Planned | Medium |

---

## 3. GitHub Issues (Offener Backlog)

| Issue | Titel | Kategorie |
|---|---|---|
| #251 | Startup ohne Config als optionales Build-Feature | Enhancement |
| #245 | RequestSeed sollte zusätzliche Parameter erlauben | Enhancement |
| #237 | rust-lang Actions statt dtolnay | CI/CD |
| #215 | Fehler bei unerwarteten Request-Parametern | Enhancement |
| #213 | `sleep` in Non-Test-Code verbieten | Code Quality |
| #204 | Deprecated axum/tower Calls ersetzen | Tech Debt |
| #195 | UDS über CAN (ISO-TP) Unterstützung | Feature |
| #192 | `GET /configurations` — Services auflisten | Enhancement |
| #181 | `/vehicle/v15` automatisch voranstellen | Enhancement |
| #179 | Neueste Proto-Version unterstützen | Dependencies |
| #178 | DTC Deletion Scope Query Parameter | Enhancement |
| #177 | DTC FLXC Structures vervollständigen | Enhancement |

---

## 4. Code-TODOs (87 Matches in 28 Dateien)

### Kritische TODOs

| Datei | Beschreibung | Ticket |
|---|---|---|
| `cda-interfaces/datatypes/com_params.rs` | 8× Tester Present Params nicht verwendet | #53 |
| `cda-interfaces/datatypes/com_params.rs` | 3× DoIP NACK/Retry Params nicht verwendet | #22 |
| `cda-comm-doip/lib.rs` | Generic NACK Handling | #22 |
| `cda-core/diag_kernel/schema.rs` | EnvDataDesc DOP nicht unterstützt | — |
| `cda-core/diag_kernel/schema.rs` | DTC DOP nicht unterstützt | — |
| `cda-core/diag_kernel/ecumanager.rs` | 26 TODOs — ECU Management Logik | — |

---

## 5. Implementierungsplan

### Legende Zeitschätzung

| Abkürzung | Bedeutung |
|---|---|
| **Dev** | Konventioneller Entwickler (manuell) |
| **Agent** | KI-Agent (Windsurf Cascade / Claude) |
| **PT** | Personentage (8h) |

---

### Phase 1: Kritische SOVD-Compliance Lücken (Prio: HIGH)

> **GitHub Issues:** [rettde/opensovd #1–#8](https://github.com/rettde/opensovd/issues)
>
> Jedes Issue enthält: Implementierungs-Tasks, Unit Tests (Pflicht), CrossCheck & Constraints, Security Audit

| # | Task | Issue | Dev | Agent | Komplexität | Abhängigkeiten |
|---|---|---|---|---|---|---|
| 1.1 | **Tester Present Parameter (#53)** — 8 ComParams in UDS-Kommunikation integrieren | [#1](https://github.com/rettde/opensovd/issues/1) | 3 PT | 1 PT | Medium | `cda-comm-uds`, `cda-interfaces` |
| 1.2 | **DoIP NACK/Retry Handling (#22)** — Generic NACK, Retry-Logik, ACK-Timeout | [#2](https://github.com/rettde/opensovd/issues/2) | 4 PT | 1.5 PT | High | `cda-comm-doip`, `cda-interfaces` |
| 1.3 | **Async Operation Execution GET/DELETE** — Route für `/operations/{service}/executions/{id}` | [#3](https://github.com/rettde/opensovd/issues/3) | 2 PT | 0.5 PT | Low | `cda-sovd` |
| 1.4 | **EnvDataDesc DOP Schema Support** — JSON Schema Generierung für Environment Data | [#4](https://github.com/rettde/opensovd/issues/4) | 3 PT | 1 PT | Medium | `cda-core/schema.rs` |
| 1.5 | **DTC DOP Schema Support** — JSON Schema Generierung für DTC DOPs | [#5](https://github.com/rettde/opensovd/issues/5) | 2 PT | 0.5 PT | Medium | `cda-core/schema.rs` |
| 1.6 | **Complex Data Type Mapping** — Verschachtelte/komplexe ODX-Datentypen nach JSON | [#6](https://github.com/rettde/opensovd/issues/6) | 5 PT | 2 PT | High | `cda-core`, `cda-database` |
| 1.7 | **Error Codes & Messages** — Standardisierte Fehlercodes gemäß ISO 17978-3 | [#7](https://github.com/rettde/opensovd/issues/7) | 2 PT | 0.5 PT | Low | `cda-sovd/error.rs` |
| 1.8 | **HTTPS/TLS Konfiguration** — Zertifikat/Key über Config oder Plugin | [#8](https://github.com/rettde/opensovd/issues/8) | 3 PT | 1 PT | Medium | `cda-sovd`, `cda-plugin-security` |
| | **Summe Phase 1** | | **24 PT** | **8 PT** | | |

#### Phase 1 — Qualitätssicherung pro Issue

Jedes der 8 Issues enthält verpflichtende Abnahmekriterien:

- **Unit Tests**: 7–11 Tests pro Issue (gesamt ~70 neue Tests)
- **CrossCheck**: Prüfung gegen ISO 14229-1, ISO 17978-3, ISO 13400-2, ISO 22901-1
- **Constraints**: Bounds-Checks, Format-Validierung, Regressionstests
- **Security Audit**: DoS-Prevention, Information Disclosure, Input Validation, Timing Attacks

### Phase 2: Diagnostic DB Update Plugin (Prio: HIGH)

| # | Task | Dev | Agent | Komplexität | Abhängigkeiten |
|---|---|---|---|---|---|
| 2.1 | **Plugin Grundstruktur** — Neues Crate `cda-plugin-db-update`, Trait-Interfaces | 3 PT | 1 PT | Medium | `cda-interfaces`, `cda-plugin-security` |
| 2.2 | **Atomares DB Update** — SOVD-API Endpoint, Rollback-Mechanismus | 5 PT | 2 PT | High | `cda-database`, `cda-core` |
| 2.3 | **Authentication & Authorization** — Zugriffskontrolle über Security Plugin | 2 PT | 0.5 PT | Medium | `cda-plugin-security` |
| 2.4 | **Integrity Verification** — Signatur/Hash-Prüfung der MDD-Dateien | 3 PT | 1 PT | Medium | — |
| 2.5 | **Downgrade Protection** — Versionsprüfung, Persistence | 2 PT | 1 PT | Medium | — |
| 2.6 | **Crash Safety** — Power-Cycle Recovery, Journaling | 4 PT | 2 PT | High | Storage Abstraction |
| | **Summe Phase 2** | **19 PT** | **7.5 PT** | | |

### Phase 3: Enhancement Backlog (GitHub Issues, Prio: MEDIUM)

| # | Task (Issue) | Dev | Agent | Komplexität |
|---|---|---|---|---|
| 3.1 | **#192 — GET /configurations auflisten** | 1 PT | 0.25 PT | Low |
| 3.2 | **#178 — DTC Scope Query Parameter** | 1.5 PT | 0.5 PT | Low |
| 3.3 | **#177 — DTC FLXC Structures** | 2 PT | 0.5 PT | Medium |
| 3.4 | **#245 — RequestSeed zusätzliche Parameter** | 1.5 PT | 0.5 PT | Low |
| 3.5 | **#215 — Fehler bei unerwarteten Parametern** | 1 PT | 0.25 PT | Low |
| 3.6 | **#251 — Startup ohne Config optional** | 2 PT | 0.5 PT | Medium |
| 3.7 | **#181 — /vehicle/v15 automatisch voranstellen** | 1.5 PT | 0.5 PT | Low |
| 3.8 | **IOControl (SID 2A) Grundimplementierung** | 5 PT | 2 PT | High |
| 3.9 | **DoIP Protokollversionen definieren** | 1 PT | 0.25 PT | Low |
| 3.10 | **Duplicate MDD Resolution Strategie** | 2 PT | 0.5 PT | Medium |
| | **Summe Phase 3** | **18.5 PT** | **5.75 PT** | |

### Phase 4: Tech Debt & Code Quality (Prio: LOW-MEDIUM)

| # | Task (Issue) | Dev | Agent | Komplexität |
|---|---|---|---|---|
| 4.1 | **#204 — Deprecated axum/tower Calls** | 1 PT | 0.25 PT | Low |
| 4.2 | **#237 — rust-lang CI Actions** | 0.5 PT | 0.25 PT | Low |
| 4.3 | **#213 — sleep Verbot in Non-Test Code** | 0.5 PT | 0.25 PT | Low |
| 4.4 | **#179 — Proto-Version Update** | 1 PT | 0.25 PT | Low |
| 4.5 | **Source Port konfigurierbar** (DoIP) | 1 PT | 0.25 PT | Low |
| 4.6 | **ECU Manager TODOs** (26 Stück bereinigen) | 5 PT | 2 PT | High |
| | **Summe Phase 4** | **9 PT** | **3.25 PT** | |

### Phase 5: Plugin-Erweiterungen & Zukunft (Prio: LOW)

| # | Task | Dev | Agent | Komplexität |
|---|---|---|---|---|
| 5.1 | **UDS Plugin (Interception)** — Requests/Responses abfangen | 4 PT | 1.5 PT | Medium |
| 5.2 | **DoIP Plugin (Interception)** — Requests/Responses abfangen | 4 PT | 1.5 PT | Medium |
| 5.3 | **Custom Endpoint Plugins** — Vendor-spezifische API-Erweiterungen | 5 PT | 2 PT | High |
| 5.4 | **UDS über CAN (#195)** — ISO-TP Transport Layer | 10 PT | 4 PT | Very High |
| 5.5 | **Logging/Safety Plugins** — Framework + Beispiel-Implementierungen | 3 PT | 1 PT | Medium |
| | **Summe Phase 5** | **26 PT** | **10 PT** | |

---

## 6. Zusammenfassung

### Compliance-Score

| Kategorie | Implementiert | Teilweise | Fehlend | Gesamt |
|---|---|---|---|---|
| **SOVD Core API (ISO 17978-3)** | 24 | 5 | 1 (IOControl) | 30 |
| **Extensions (CDA-spezifisch)** | 16 | 3 | 0 | 19 |
| **Plugins (Requirement)** | 1 (Security) | 0 | 5 | 6 |
| **Gesamt** | **41** | **8** | **6** | **55** |

**SOVD Core Compliance: ~80%** (24/30 voll implementiert)  
**Gesamt-Feature-Completion: ~75%** (41/55 voll implementiert)

### Zeitübersicht

| Phase | Beschreibung | Dev (PT) | Agent (PT) | Einsparung |
|---|---|---|---|---|
| **Phase 1** | Kritische SOVD Lücken | 24 | 8 | 67% |
| **Phase 2** | DB Update Plugin | 19 | 7.5 | 61% |
| **Phase 3** | Enhancement Backlog | 18.5 | 5.75 | 69% |
| **Phase 4** | Tech Debt | 9 | 3.25 | 64% |
| **Phase 5** | Plugin-Erweiterungen | 26 | 10 | 62% |
| **Gesamt** | | **96.5 PT** | **34.5 PT** | **~64%** |

### Empfohlene Prioritätsreihenfolge

1. **Phase 1** (SOVD-Compliance) — Pflicht für ISO-Konformität
2. **Phase 2** (DB Update Plugin) — Requirement für produktiven Einsatz
3. **Phase 3.8** (IOControl SID 2A) — Letzte fehlende SID-Abdeckung
4. **Phase 3** (restliche Enhancements) — GitHub-Community & Qualität
5. **Phase 4** (Tech Debt) — Wartbarkeit
6. **Phase 5** (Plugins) — Zukunft & Erweiterbarkeit

### Agent-Einsatz Empfehlung

| Besonders geeignet für Agent | Besser beim Entwickler |
|---|---|
| Boilerplate-Code (Route-Registrierung, Traits) | DoIP NACK Retry-Logik (Netzwerk-Timing kritisch) |
| Error Code Definitionen & Mapping | Crash Safety / Journaling (Architektur-Entscheidung) |
| Schema-Generierung (EnvDataDesc, DTC DOP) | HSM/TLS Integration (Hardware-abhängig) |
| CI/CD & Lint-Konfiguration | UDS über CAN (ISO-TP Protokoll-Stack) |
| Deprecated API Migration | Plugin Architektur-Design (Review erforderlich) |
| Test-Erstellung & Dokumentation | Performance-kritische ECU-Manager-Logik |
| GitHub Issue Fixes (Low Complexity) | Funktionale Sicherheit (Safety Plugins) |
