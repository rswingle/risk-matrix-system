# Data Flow — Risk Matrix System

## Overview

This document traces data through the four system components across two primary
workflows: (1) matrix definition and export, and (2) end-to-end vulnerability
scoring and ticket creation.

---

## Workflow A: Risk Matrix Definition

```
User runs: node src/builder/index.js
        │
        ▼
┌───────────────────────────────────────────┐
│         BUILDER: Input Collection         │
│                                           │
│  Prompt: likelihood levels                │
│  Prompt: impact levels                    │
│  Prompt: severity grid (L × I cells)     │
│  Prompt: SLA policy per severity level   │
│                                           │
│  Output: raw_matrix (in-memory object)   │
└──────────────────┬────────────────────────┘
                   │ raw_matrix
                   ▼
┌───────────────────────────────────────────┐
│         BUILDER: Validation               │
│                                           │
│  Verify: all L × I cells assigned        │
│  Verify: SLA days are positive integers  │
│  Verify: no duplicate level labels       │
│                                           │
│  Output: validated RiskMatrix object     │
└──────────────────┬────────────────────────┘
                   │ RiskMatrix
         ┌─────────┴──────────┐
         ▼                    ▼
┌─────────────────┐  ┌─────────────────────┐
│  Storage Layer  │  │  Excel Exporter     │
│                 │  │                     │
│  write(         │  │  Generate color-    │
│    "matrix/     │  │  coded grid +       │
│    current",    │  │  SLA sheet          │
│    matrix       │  │                     │
│  )              │  │  Output:            │
│                 │  │  /data/matrix.xlsx  │
│  /data/         │  └─────────────────────┘
│  matrix.json    │
└─────────────────┘
```

---

## Workflow B: End-to-End Vulnerability Scoring + Ticket Creation

```
User provides: data/findings/scan-001.json
        │
        ▼
┌────────────────────────────────────────────────────┐
│                SCORER: Startup                      │
│                                                     │
│  Read matrix from storage:                          │
│    read("matrix/current") → RiskMatrix             │
│                                                     │
│  Read findings from storage:                        │
│    read("findings/scan-001") → Finding[]           │
│                                                     │
│  Load AI model config from config/settings.json    │
└──────────────────────┬─────────────────────────────┘
                       │ {matrix, findings, config}
                       ▼
┌────────────────────────────────────────────────────┐
│           SCORER: Per-Finding Loop                  │
│                                                     │
│  For each VulnerabilityFinding:                     │
│                                                     │
│  1. Build classification prompt                     │
│     (inject CVE, description, CVSS, matrix labels) │
│                                                     │
│  2. Call AI model (classifier.js)                  │
│     → {likelihood_index, impact_index, reasoning}  │
│                                                     │
│  3. Apply matrix (mapper.js)                       │
│     likelihood_index × impact_index                │
│     → severity label ("High", "Critical", etc.)   │
│                                                     │
│  4. Compute SLA due date                            │
│     today + sla_policy[severity] days              │
│                                                     │
│  5. Produce ScoredFinding object                   │
└──────────────────────┬─────────────────────────────┘
                       │ ScoredFinding[]
                       ▼
┌────────────────────────────────────────────────────┐
│              STORAGE: Persist Scored Results        │
│                                                     │
│  write("scored/scan-001", ScoredFinding[])         │
│  → /data/scored/scan-001.json                      │
└──────────────────────┬─────────────────────────────┘
                       │ ScoredFinding[]
                       ▼
┌────────────────────────────────────────────────────┐
│           TICKETING: Per-Finding Loop               │
│                                                     │
│  For each ScoredFinding:                            │
│                                                     │
│  1. Build TicketPayload                            │
│     - title: "CVE-XXXX: {short description}"      │
│     - description: rendered Markdown body          │
│     - severity + priority mapped from score        │
│     - due_date from SLA computation                │
│     - labels: ["security", "vuln", severity]       │
│                                                     │
│  2. Call Jira adapter                              │
│     jira.createTicket(payload)                    │
│     → {ticket_id, url, status}                    │
│                                                     │
│  3. Record TicketRecord                            │
│     {cve_id, ticket_id, url, created_at}          │
└──────────────────────┬─────────────────────────────┘
                       │ TicketRecord[]
                       ▼
┌────────────────────────────────────────────────────┐
│         STORAGE: Persist Ticket Records             │
│                                                     │
│  write("tickets/scan-001", TicketRecord[])         │
│  → /data/tickets/scan-001.json                     │
│                                                     │
│  Console output:                                    │
│  ✔  CVE-2024-1234 → JIRA-1042 (Critical, due 24h) │
│  ✔  CVE-2024-5678 → JIRA-1043 (High, due 7d)      │
└────────────────────────────────────────────────────┘
```

---

## Data Entities: Transformation Chain

```
VulnerabilityFinding (raw input)
         │
         │  + AI classification (likelihood, impact, reasoning)
         │  + matrix lookup → severity label
         │  + SLA policy → due_date
         ▼
ScoredFinding (enriched)
         │
         │  + rendered Markdown description
         │  + Jira priority mapping
         │  + label generation
         ▼
TicketPayload (ready for adapter)
         │
         │  + Jira ticket ID
         │  + URL
         │  + creation timestamp
         ▼
TicketRecord (persisted reference)
```

---

## File Read/Write Matrix

| File | Builder | Scorer | Ticketing | Storage |
|---|---|---|---|---|
| `data/matrix.json` | W | R | — | R/W |
| `data/matrix.xlsx` | W | — | — | — |
| `data/findings/*.json` | — | R | — | R/W |
| `data/scored/*.json` | — | W | R | R/W |
| `data/tickets/*.json` | — | — | W | R/W |
| `config/settings.json` | R | R | R | — |
| `scripts/generated-scorer.js` | — | W | — | — |

*R = Read, W = Write*

---

## AI Model Call Contract

The classifier sends one HTTP request per finding. The interface is intentionally
model-agnostic via an OpenAI-compatible API format.

**Request:**
```json
POST {AI_ENDPOINT}/v1/chat/completions
{
  "model": "{configured_model}",
  "messages": [
    { "role": "system", "content": "You are a security risk classifier..." },
    { "role": "user", "content": "CVE: ...\nClassify as JSON." }
  ],
  "response_format": { "type": "json_object" }
}
```

**Expected response shape:**
```json
{
  "likelihood_index": 4,
  "impact_index": 5,
  "reasoning": "Remotely exploitable with no authentication on internet-facing hosts."
}
```

**Error handling:** If the model returns malformed JSON or an index out of range,
the scorer logs the error, assigns `severity: "Unscored"`, and continues to the
next finding. Unscored findings are flagged in the output for manual review.

---

## End-to-End Verification Trace

```
Input:  findings/scan-001.json          (3 CVEs)
          │
          ▼
Scorer: CVE-2024-1001
  AI → likelihood: 4 (Likely), impact: 5 (Catastrophic)
  Matrix lookup → severity: Critical
  SLA → due: 2025-04-02 (1 day)

Scorer: CVE-2024-2002
  AI → likelihood: 3 (Possible), impact: 3 (Moderate)
  Matrix lookup → severity: Medium
  SLA → due: 2025-05-01 (30 days)

Scorer: CVE-2024-3003
  AI → likelihood: 2 (Unlikely), impact: 4 (Major)
  Matrix lookup → severity: High
  SLA → due: 2025-04-08 (7 days)

Ticketing:
  CVE-2024-1001 → JIRA-1042 [Critical] ✔
  CVE-2024-2002 → JIRA-1043 [Medium]   ✔
  CVE-2024-3003 → JIRA-1044 [High]     ✔

Stored:
  data/scored/scan-001.json             ✔
  data/tickets/scan-001.json            ✔
```
