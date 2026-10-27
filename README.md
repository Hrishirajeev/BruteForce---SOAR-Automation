# BruteForce SOAR Automation

Automated detection, enrichment, and case creation for brute-force login attacks — built with Splunk, N8n, TheHive, VirusTotal, and AbuseIPDB.

> MITRE ATT&CK: **T1110.001 — Password Guessing**

---

## What this is

A lab-built SOAR (Security Orchestration, Automation and Response) workflow that:

- Detects brute-force login attempts from Splunk logs
- Filters out false positives (e.g. a user who forgot their password)
- Enriches the attacker IP with VirusTotal and AbuseIPDB
- Calculates a composite risk score
- Creates or updates a case in TheHive — Critical or Medium severity
- Avoids duplicate cases for the same attacker
- Emails the SOC team a summary

Automation handles investigation and documentation. **It does not auto-block IPs** — that decision is left to a human analyst, by design (see [Why no auto-blocking](#why-no-auto-blocking)).

---

## Architecture

```
Kali (attacker) ──attacks──▶ Windows VM (target)
                                   │
                          Splunk Forwarder
                                   │
                          Splunk Enterprise (alert)
                                   │  webhook
                                   ▼
                                  N8n
                   ┌───────────────┼───────────────┐
                   │               │               │
             VirusTotal       AbuseIPDB         TheHive
                   │               │               │
                   └───────┬───────┘               │
                     Risk Scoring                  │
                           │                        │
                     Severity Check ─────▶ Create/Update Case
                           │                        │
                           └────────▶ Email to SOC Team
```

| Tool | Role |
|---|---|
| **Splunk** | Detects failed logins (EventCode 4625), fires a real-time alert |
| **N8n** | Workflow engine — orchestrates everything below |
| **VirusTotal** | IP reputation check (malicious detection count) |
| **AbuseIPDB** | Abuse confidence score (0–100) |
| **TheHive** | Case management — stores and tracks incidents |

---

## How it decides

**1. Is this a real attack or a mistake?**
More than 10 failed attempts → investigate. Fewer → log as a likely false positive, no enrichment needed.

**2. How dangerous is the IP?**
```
risk_score = (AbuseIPDB_score × 0.6) + (min(VirusTotal_detections / 10, 1) × 40)
```
Score ≥ 80 → **High severity**. Below 80 → **Medium severity**.

**3. Have we seen this IP before?**
TheHive is searched first. If a case already exists, it's updated instead of duplicated.

**4. Notify.**
Every path ends with an email containing the IP, attempt count, both enrichment scores, the risk score, and a recommended action.

---

## Why no auto-blocking

Threat intelligence scores are not 100% accurate. Automatically blocking an IP risks locking out a legitimate user, a VPN, or a partner network — with no human in the loop to catch the mistake.

This workflow **recommends** action and documents everything an analyst needs to decide fast. The decision to block stays human.

---

## Repo contents

```
├── workflow.json         # Importable N8n workflow
├── README.md
└── screenshots/
    ├── 01-workflow-canvas.png
    ├── 02-execution-high-path.png
    ├── 03-thehive-case.png
    ├── 04-alert-email.png
    ├── 05-risk-scoring-code.png
    └── 06-false-positive-run.png
```

---

## Proof it runs

| Severity | Attacker IP | VirusTotal | AbuseIPDB | Risk Score | Result |
|---|---|---|---|---|---|
| **High** | `49.206.27.37` | 6 detections | 100 | **84/100** | Case created, email sent |
| **Low** | `8.8.8.8` | 0–1 detections | ~0 | **~1/100** | Case created, email sent |

Screenshots of each run are in [`screenshots/`](./screenshots).

---

## Setup (lab)

1. Deploy Splunk Universal Forwarder on the target machine, forward `Security` logs to Splunk.
2. Create a real-time Splunk alert on `EventCode=4625`, triggered when count > 5, with a **Webhook action** pointing to N8n's **Production** webhook URL.
3. Import `workflow.json` into N8n.
4. Add your own credentials: VirusTotal API key, AbuseIPDB API key, TheHive API credential, SMTP email credential.
5. Publish the workflow.

> API keys and credentials are **not** included in `workflow.json`. Add your own before running.

---

## Lessons learned

- Test webhook URLs ≠ production webhook URLs — N8n needs the Production URL for live alerts to work.
- A workflow must be **Published**, not just saved, to receive traffic.
- Always verify real field names from upstream data instead of assuming them.
- Deduplication must run *before* case creation, not after.
- Severity scoring and blocking logic deserve a second pass — test with real malicious IPs, not just arbitrary ones, to confirm thresholds actually trigger.

---

## Disclaimer

Built in an isolated lab environment for learning purposes. Not intended as a production-ready security tool.
