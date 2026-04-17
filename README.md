# NetOps Toolkit

NetOps Toolkit is a modular network operations web platform built to automate repetitive working process and turn raw CLI output into human-readable diagnostics.

The project combines two major areas:

- **GPON automation** for Iskratel SI3000 and G16 access devices
- **Cisco log and interface analysis** with correlation, historical intelligence and assessment summaries

> This repository is a portfolio reference only.  
> The full source code is not published because the project contains private network IP addresses, internal credentials and company-specific configuration details.

---

## Project Overview

NetOps Toolkit was designed as a local/self-hosted internal platform for daily network operations.

Its purpose is to:

- Reduce repetitive manual CLI work
- Standardize assignment and troubleshooting
- Improve operational safety through validation and restricted command handling
- Transform raw device output into actionable analysis
- Provide humanized summaries, diagnostics and guided next steps

---

## Modules

### 1. Cisco Analyzer
A troubleshooting interface for Cisco logs and interface diagnostics.

**Features**
- Analyze raw CLI output from `show logging`
- Analyze interface health from `show interface`
- Correlate findings across multiple outputs
- Generate plain-language summaries
- Detect likely incidents and recurring problems
- Provide historical trend analysis
- Support guided playbooks for diagnostic process

---

### 2. Iskratel SI3000 & G16 GPON Multifunctional Tool
Web-based ONU configuration for GPON devices.

**Features**
- SI3000 configuration
- G16 configuration
- Bridge setup for Internet / IPTV / WiFi
- SIP / VoIP configuration
- Profile management
- Full delete/add config operations with safety validation
- Command execution with retry and backoff logic
- Device search and filtering
- JSON-based device storage with region tagging
- Telnet transport with prompt detection, pager handling and negotiation

---

### 2. SI3000 ONU Assignment
ONU assignment designed for high-volume usage.

**Features**
- Assign up to 64 ONUs in one run
- Serial validation before execution
- Retry on failure
- Stop-signal support
- Progress tracking with visual 1-64 ONU status grid
- Real-time execution feedback via WebSocket

**Safety features**
- Blocked destructive command patterns
- Protected delete validation
- Command filtering against unsafe operations

---

## Diagnostic Intelligence

The Cisco Analyzer includes a rule-based correlation engine that combines multiple data sources to generate root-cause-oriented findings.

### Correlation Examples
- Burst flap + physical errors → likely physical-layer fault
- Sustained flap + physical errors → marginal connection
- Flap + resets without physical errors → remote-side instability
- Optic alarms + link down → optical signal loss
- Optic alarms + link up + CRC → degraded optical path
- Flap with clean interface → self-recovered instability
- Output drops + high utilization → congestion

---

## Humanized Output

A major design goal of the platform is to make diagnostics easier to understand.

Instead of returning only technical parser results, the system converts findings into human-friendly language such as:

- What happened
- Likely impact
- Most probable cause
- What to check next

This makes the tool useful not only for experienced engineers, but also for junior team members.

---

## Command Playbooks

Includes guided command playbooks for troubleshooting.

### Example playbooks
- **Link Health Check**
  - `show interface`
  - `show logging`
  - automatic cross-correlation

- **Physical Layer Diagnostics**
  - `show interface`
  - `show controllers`
  - `show logging`

- **Connectivity Snapshot**
  - `show ip interface brief`
  - `show cdp neighbors`
  - `show version`

Each playbook executes sequential steps, parses the returned output appropriately and presents the results in an human-friendly way.

---

## Historical Intelligence

Stores analysis history and uses it to identify operational patterns over time.

**Examples**
- Degrading vs improving device/interface health
- Recurring interface instability
- Anomaly detection
- Health score tracking
- Per-device and global trend summaries

---

## Analysis Layer

The platform also includes assisted analysis layer for troubleshooting support.

**Features**
- Root cause summary
- Likely impact
- Next recommended actions
- Escalation trigger
- Historical context
- Suggested follow-up commands

Executed asynchronously so the main application remains responsive.

---

## Technology Stack

- **Backend:** Python, FastAPI, Uvicorn
- **Frontend:** Jinja2 templates, Bootstrap 5, JavaScript
- **Real-time communication:** WebSocket
- **Network automation:** Telnet-based transport and execution layers
- **Validation / schemas:** Pydantic
- **Storage:** JSON-based local storage

---

## Screenshots

This repository includes screenshots of the interface glimpses for demonstration purposes.

Screenshot section:
- Dashboard
- Device Profiles
- Cisco Analyzer
- GPON Multifunctional Tool
- Analysis History
- Reports & Exports

---

## Why the source code is not public

The project contains:
- Private network IP addresses
- Internal credentials
- Company-specific commands and configuration patterns

For that reason this repository is shared as a portfolio reference rather than a public source distribution.

---

## Author

**Guga Gugunava**

Email: guga.gugunava@gmail.com

---
## License

**All Rights Reserved**

This repository is provided for portfolio and demonstration purposes only.  
No permission is granted to copy, modify, distribute or reuse the source, architecture, screenshots or documentation without explicit written permission from the author.
