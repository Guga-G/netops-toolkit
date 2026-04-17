# Architecture

This document describes the internal design of NetOps Toolkit - how the pieces fit together, what each layer is responsible for and why certain trade-offs were made. It is written as a portfolio reference for anyone reviewing the source.

---

## High-level overview

NetOps Toolkit is a server-rendered Python web application that turns raw Cisco CLI output into structured, severity-classified diagnostics. A layered intelligence pipeline sits on top of deterministic parsers, progressively refining raw text into actionable incident reports.

```
                         Browser
                           |
                      FastAPI / Uvicorn
                           |
           ----------------------------------
           |               |                |
        Routes          Templates        Static
     (APIRouter)       (Jinja2)        (CSS / JS)
           |
        Services
     (business logic)
           |
     --------------------------
     |            |           |
   Parsers    Modules     Storage
  (regex /   (correlation, (JSON
  state       intelligence,  files)
  machines)   playbooks,
              GPON tools)
```

The application follows a three-tier layered architecture:

1. **Routes** handle HTTP concerns - form parsing, query parameters, response rendering.
2. **Services** encapsulate business logic - CRUD, encryption, export generation, command safety.
3. **Modules and Parsers** contain domain logic - deterministic text analysis, cross-signal correlation, historical intelligence, GPON command templates.

---

## Technology stack

| Layer | Technology | Role |
|---|---|---|
| Runtime | Python 3.10+ | Language runtime |
| Web framework | FastAPI | Route handling, request validation |
| Server | Uvicorn | ASGI server |
| Templates | Jinja2 | Server-side HTML rendering |
| Validation | Pydantic | Data models and serialization |
| Frontend | Bootstrap 5.3.7 + custom CSS | Layout, components, responsive grid |
| Device transport | telnetlib3, raw sockets | Telnet protocol for CLI access |
| Encryption | cryptography (Fernet) | Credential encryption at rest |
| Excel export | openpyxl | XLSX report generation |
| PDF export | fpdf2 | PDF report generation |
| Environment | python-dotenv | Configuration loading |

---

## Directory layout

```
netops-toolkit/
|-- run.py                              # Entry point - starts uvicorn
|-- requirements.txt                    # Pinned dependencies
|-- .env.example                        # Environment variable template
|-- app/
    |-- main.py                         # FastAPI app, router registration
    |-- config.py                       # Paths, env loading, app metadata
    |-- models/
    |   |-- device.py                   # DeviceProfile (Pydantic)
    |   |-- analysis.py                 # AnalysisRecord (Pydantic)
    |   |-- forms.py                    # Form validation (reserved)
    |-- routes/
    |   |-- dashboard.py                # GET / - KPI dashboard
    |   |-- cisco_analyzer.py           # GET/POST /cisco - analysis UI
    |   |-- device_profiles.py          # CRUD /devices
    |   |-- history.py                  # GET /history - run history
    |   |-- reports.py                  # GET /reports - export hub
    |   |-- gpon.py                     # /gpon - WebSocket + REST
    |-- services/
    |   |-- device_service.py           # Device profile CRUD
    |   |-- history_service.py          # Analysis record persistence
    |   |-- credential_crypto.py        # Fernet encrypt/decrypt
    |   |-- telnet_client.py            # TelnetCiscoClient
    |   |-- cisco_command_service.py    # Command safety validation
    |   |-- export_service.py           # CSV, JSON, XLSX, PDF generation
    |-- parsers/
    |   |-- cisco_log_parser.py         # show logging parser (1072 lines)
    |   |-- cisco_interface_parser.py   # show interface parser (874 lines)
    |-- knowledge_base/
    |   |-- cisco_messages.py           # CISCO_MESSAGE_KB - explanations
    |-- modules/
    |   |-- cisco_analyzer/
    |   |   |-- correlation_engine.py   # Cross-signal incident detection
    |   |   |-- historical_intelligence.py  # Trend/anomaly detection
    |   |   |-- playbooks.py            # Multi-step diagnostics
    |   |-- gpon_tools/
    |       |-- device_store.py         # GPON device persistence
    |       |-- command_engine.py       # Command execution with retry
    |       |-- command_builders.py     # ONU command templates
    |       |-- transport_telnet.py     # Raw socket telnet protocol
    |       |-- profiles.py             # Device profile dataclasses
    |-- templates/
    |   |-- base.html                   # Root layout (navbar, sidebar, content)
    |   |-- dashboard.html              # KPI cards
    |   |-- cisco_analyzer.html         # Analysis form and results
    |   |-- device_profiles.html        # Device CRUD table
    |   |-- history.html                # Run history with filters
    |   |-- reports.html                # Export hub
    |   |-- gpon.html                   # GPON tool UI
    |   |-- partials/                   # Reusable fragments
    |-- static/
    |   |-- css/app.css                 # Design tokens, components (970 lines)
    |   |-- js/app.js                   # Minimal client-side JS
    |-- storage/
        |-- devices.json                # Encrypted device profiles
        |-- history.json                # Analysis records
        |-- .cred_key                   # Auto-generated Fernet key (gitignored)
        |-- gpon/                       # GPON device inventory
```

**Codebase size:** ~6,000 lines of Python across 34 modules, ~2,800 lines of HTML across 10 templates, ~970 lines of CSS.

---

## Core design principle - deterministic first, intelligence second

This is the most important architectural decision in the project.

Raw CLI text is never exposed directly to downstream consumers. It is first converted to a normalized structured representation by deterministic parsers. Higher layers - correlation rules, historical trend detection, assessments - operate only on that structured representation. Two consequences follow:

1. **Every layer is testable in isolation.** Parsers can be validated against fixture inputs. Correlation rules can be unit-tested against synthetic signal combinations. The historical intelligence engine can be exercised against canned analysis records.

2. **Output is consistent across runs.** The same `show logging` output produces the same structured shape every time because the parsers are deterministic. Downstream layers receive a stable contract, not raw text variance.

```
Raw CLI text
    |
    |
Deterministic Parsers  --->  Structured JSON (events, signals, scores)
                                    |
                                    |
                          Correlation Engine  --->  Incidents with evidence
                                                          |
                                                          |
                                              Historical Intelligence
                                                  (trends, anomalies,
                                                   recurring issues)
                                                          |
                                                          |
                                                  Assessment output
                                                  (root cause, impact,
                                                   actions, commands)
```

---

## Component deep dives

### Parsers

Both parsers are deterministic state machines built on regex pattern matching. They share a common interface - accept raw text, return a dict with `summary`, `severity_counts`, `signals`, `parsed_rows` and `details`.

**CiscoLogParser** (`app/parsers/cisco_log_parser.py` - 1,072 lines)

Processes raw `show logging` output. Extracts individual syslog events with timestamp, severity, facility, mnemonic and message. Produces higher-order signals:

- **Flapping interfaces** - categorized by density into `sustained_flap` (steady state changes over minutes), `burst_flap` (rapid cluster in a short window) or `isolated_flap` (single event).
- **Optic alarms** - THRESHOLD_VIOLATION and related mnemonics indicating SFP degradation.
- **Security violations** - DHCP snooping denies, ARP inspection failures.
- **CRC storms** - accumulated input errors suggesting physical layer problems.

Each event is enriched with a plain-language explanation from the knowledge base. The goal is that someone who is not a network engineer can read the output and understand what happened.

**CiscoInterfaceParser** (`app/parsers/cisco_interface_parser.py` - 874 lines)

Processes raw `show interface <name>` output. Extracts every counter the IOS/IOS-XE interface display provides - MTU, bandwidth, duplex, media type, utilization, error counters, resets, lost carrier, runts.

Computes:

- **Health score** (0-100) based on error presence and severity.
- **Peak utilization** from five-minute rate relative to bandwidth.
- **Findings** - plain-language descriptions of detected issues.
- **Recommendations** - actionable next steps tied to each finding.

### Correlation engine

`app/modules/cisco_analyzer/correlation_engine.py` - 263 lines

Cross-references signals from the log parser and the interface parser to produce evidence-backed diagnoses.

Four rules:

| Rule | Trigger | Diagnosis | Confidence |
|---|---|---|---|
| 1 | Burst flap + physical errors | Physical Layer Fault | High |
| 2 | Sustained flap + physical errors | Marginal Connection | Medium |
| 3 | Optic alarms present | Optical Signal Issues | Varies |
| 4 | High CRC + no flapping | Duplex or Cable Mismatch | Medium |

Each correlated incident includes: what happened, why it matters, evidence from both sources, confidence level and severity. This is the output that Deep Analysis mode generates - the merged view of log events and interface counters producing a single diagnosis.

### Historical intelligence

`app/modules/cisco_analyzer/historical_intelligence.py`

Analyzes collections of past analysis records per device to surface patterns that a single-run view cannot show:

- **Trend detection** - tracks critical event counts over time, classifying direction as improving, stable or degrading.
- **Anomaly detection** - identifies spikes or recoveries using a 2.5x change threshold against the recent baseline.
- **Recurring interfaces** - surfaces interfaces that appear across multiple runs, separating chronic problem ports from one-off events.
- **Health and CRC trend history** - longitudinal view of interface health scores and error counter growth.
- **Global statistics** - total runs, devices analyzed, events by severity, top recurring issues across the whole fleet.

Used by the dashboard, history page and reports page to populate intelligence summary cards. The output feeds into the final assessment so that per-run diagnostics are contextualized by fleet-wide history.

### Playbooks

`app/modules/cisco_analyzer/playbooks.py`

Pre-defined multi-step diagnostic processes:

- **Link Health Check** - collects interface stats + logging, correlates.
- **Physical Layer Diagnostics** - interface + show controllers + logging.
- **Connectivity Snapshot** - interface brief + CDP neighbors + version (no specific interface needed).

Each step specifies a command template, parser type and purpose. The playbook engine executes steps sequentially over Telnet and correlates results when the playbook definition requests it. The resulting structured output flows into the same downstream intelligence layers as a single-shot analysis.

### GPON tools

`app/modules/gpon_tools/`

Handles SI3000 and G16 OLT/ONU configurations:

- **Device store** - persistent inventory with encrypted credentials, per-model JSON files.
- **Command engine** - executes command sequences with safety validation, retry logic for database lock errors and delete guardrails requiring structural anchors.
- **Transport** - raw socket Telnet with IAC negotiation and pager handling, maintaining persistent connections for multi-command sessions.
- **Command builders** - predefined CLI command templates for ONU assignment, bridge creation and related operations.

The GPON tool uses WebSockets to maintain persistent Telnet sessions that survive browser tab switches.

---

## Data flow

### Analysis flow

```
User submits form (device + action)
         |
         |
cisco_analyzer route selects action
         |
    -----------------------------
    |          |       |        |
collect_   collect_ run_     manual_
logging  interface playbook  paste
    |          |       |        |
    |          |       |        |
 Telnet     Telnet  Telnet   Raw text
 client     client  client   from form
    |          |       |        |
    |          |       |        |
CiscoLog   CiscoInt  Per-step  Parser
Parser     Parser    parsers   (auto)
    |          |       |        |
    -----------------------------
         |
         |
    [Deep Analysis?]
         |--- yes ---> CorrelationEngine.correlate()
         |                    |
         |                    |
    HistoricalIntelligence    Correlated incidents
    (device history context)          |
         |                            |
         ------------------------------
                        |
                        |
                  Assessment output
                        |
                        |
                  HistoryService.add_item()
                        |
                        |
                  Template response
                  (results rendered inline)
```

### Credential lifecycle

```
User enters password in form
         |
         |
DeviceService encrypts with Fernet
         |
         |
devices.json stores "enc:v1:..." ciphertext
         |
    [On device use]
         |
         |
DeviceService decrypts at execution time
         |
         |
TelnetCiscoClient receives plaintext
         |
    [Connection closed]
         |
         |
Plaintext discarded (not cached)
```

### Export flow

```
User selects report type + format + filters
         |
         |
reports route queries HistoryService
         |
         |
ExportService transforms records
         |
    ----------------
    |    |    |    |
   CSV  JSON XLSX  PDF
    |    |    |    |
    |    |    |    |
  StreamingResponse (download)
```

---

## Security model

**Credentials at rest** - Fernet symmetric encryption (AES-128-CBC + HMAC-SHA256). The key is loaded from the `NETOPS_CRED_KEY` environment variable (recommended) or auto-generated at `app/storage/.cred_key` (single-machine fallback). Encrypted values carry an `enc:v1:` prefix so the encrypt function is idempotent - already-encrypted values pass through unchanged.

**Command safety** - only `show` commands (read-only diagnostics) are allowed. The `CiscoCommandService` blocks `config`, `configure`, `shutdown`, `reload`, `copy`, `delete`, `erase` and other state-changing commands by pattern matching before any command reaches the Telnet client.

**Credential display** - the device edit form always blanks password and enable_secret fields. Empty form submission means "keep existing value." The credential is never rendered in HTML.

**GPON safeguards** - `onu clear` is unconditionally blocked. Bridge delete commands require a `1-1-` structural anchor. Database lock retries use exponential backoff.

**Deployment assumption** - the application assumes internal network deployment. There is no authentication layer or multi-tenancy. All analysis history is visible to any user who can reach the server.

---

## Storage

JSON files - no database server dependency.

| File | Contents | Growth |
|---|---|---|
| `devices.json` | Device profiles with encrypted credentials | Small (tens of records) |
| `history.json` | Every analysis run with full structured results | Unbounded (20MB+ observed) |
| `gpon/si300_devices.json` | SI3000 device inventory | Small |
| `gpon/g16_devices.json` | G16 device inventory | Small |
| `.cred_key` | Fernet encryption key (gitignored) | Static |

All writes are atomic - the service reads the entire file, modifies in memory, writes the entire file. This prevents partial writes at the cost of not scaling to very large datasets.

---

## Frontend architecture

Server-rendered HTML with Jinja2 templates extending a shared `base.html` layout. No SPA framework - forms submit to FastAPI endpoints and receive full page responses.

**Template hierarchy:**

```
base.html (navbar, sidebar, content block, footer)
  |-- dashboard.html        KPI cards, recent runs
  |-- cisco_analyzer.html   Analysis form, inline results
  |-- device_profiles.html  CRUD table with modals
  |-- history.html          Filtered run list, intelligence cards
  |-- reports.html          Export filters and download controls
  |-- gpon.html             GPON tool, WebSocket integration
```

**Styling** uses Bootstrap 5 grid for layout with a custom stylesheet (`app.css`) defining design tokens (color palette, shadows), component classes and responsive adjustments. The visual style uses gradient backgrounds and subtle glassmorphism.

**JavaScript** is minimal. The GPON tool uses native WebSocket API for persistent Telnet sessions. Other pages rely on standard form submissions.

---

## Configuration

All configuration flows through `app/config.py` which calls `python-dotenv` to load `.env` before any module reads `os.environ`.

| Variable | Purpose | Required |
|---|---|---|
| `NETOPS_CRED_KEY` | Fernet key for credential encryption | No - auto-generated if absent |

Path constants (`TEMPLATES_DIR`, `STATIC_DIR`, `STORAGE_DIR`, `DEVICES_FILE`, `HISTORY_FILE`, `REPORTS_DIR`) are derived from `BASE_DIR` in `config.py`.

---

## Extension points

| What | Where | Interface |
|---|---|---|
| New parser | `app/parsers/` | `analyze(raw_text) -> dict` with `summary`, `severity_counts`, `signals` |
| New playbook | `PLAYBOOKS` list in `playbooks.py` | Dict with `name`, `steps[]`, `correlate` flag |
| New route | `app/routes/` + register in `main.py` | `fastapi.APIRouter` instance |
| New device type | `app/modules/` | Follow `gpon_tools/` structure |
| New export format | `ExportService` class | Add method returning bytes or streaming response |
| New correlation rule | `CorrelationEngine` | Add rule method checking signal combinations |
| New intelligence metric | `HistoricalIntelligenceEngine` | Add method operating on analysis record collections |
