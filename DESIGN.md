# Design — Risk Matrix System

## Component 1: CLI Risk Matrix Builder

### Interaction Flow

The builder runs as a guided multi-step console session. Each step is independent
so users can re-run individual steps without losing previous work.

```
$ node src/builder/index.js

╔══════════════════════════════════════════════════════╗
║          Corporate Risk Matrix Builder               ║
╠══════════════════════════════════════════════════════╣
║  Step 1 of 4 › Define Likelihood Levels              ║
╚══════════════════════════════════════════════════════╝

? How many likelihood levels? (default: 5)  ›  5
? Label for level 1 (lowest):  › Rare
? Label for level 2:           › Unlikely
? Label for level 3:           › Possible
? Label for level 4:           › Likely
? Label for level 5 (highest): › Almost Certain

Step 2 of 4 › Define Impact Levels
...

Step 3 of 4 › Map Likelihood × Impact to Severity
...

Step 4 of 4 › Set SLA Policy (days to remediate per severity)
? Critical:  ›  1
? High:      ›  7
? Medium:    ›  30
? Low:       ›  90

✔  Matrix saved to /data/matrix.json
✔  Spreadsheet saved to /data/matrix.xlsx
```

---

### Data Model: Risk Matrix

```typescript
interface RiskMatrix {
  version: string;                   // "1.0"
  created_at: string;                // ISO8601
  organization: string;              // Optional org name

  likelihood_levels: Level[];
  impact_levels: Level[];
  severity_map: SeverityCell[][];    // [likelihood_index][impact_index]
  sla_policy: SLAPolicy;
}

interface Level {
  index: number;       // 1–N, 1 = lowest
  label: string;       // "Rare", "Unlikely", etc.
  description?: string;
}

interface SeverityCell {
  likelihood_index: number;
  impact_index: number;
  severity: "Critical" | "High" | "Medium" | "Low" | "Informational";
}

interface SLAPolicy {
  Critical: number;    // Days to remediate
  High: number;
  Medium: number;
  Low: number;
  Informational: number;
}
```

---

### Excel Output Design (`matrix.xlsx`)

The exported spreadsheet contains two sheets:

**Sheet 1: Risk Matrix** — A color-coded grid

| | Negligible | Minor | Moderate | Major | Catastrophic |
|---|---|---|---|---|---|
| **Almost Certain** | Medium | High | Critical | Critical | Critical |
| **Likely** | Low | Medium | High | Critical | Critical |
| **Possible** | Low | Medium | Medium | High | Critical |
| **Unlikely** | Low | Low | Medium | Medium | High |
| **Rare** | Info | Low | Low | Medium | High |

Color coding: Critical = red, High = orange, Medium = yellow, Low = green, Info = grey.

**Sheet 2: SLA Policy** — Due dates per severity level.

---

## Component 2: Storage Layer

### Abstract Interface

```javascript
// src/storage/interface.js
class AbstractStore {
  async read(key) { throw new Error("Not implemented") }
  async write(key, value) { throw new Error("Not implemented") }
  async list(prefix) { throw new Error("Not implemented") }
  async delete(key) { throw new Error("Not implemented") }
}
```

### Local Implementation (`src/storage/local.js`)

- Keys map to file paths under `/data/`
- Values are serialized as JSON
- `list(prefix)` returns all files matching `data/{prefix}*`
- No dependencies beyond Node.js `fs`

### Storage Key Conventions

| Key | Value | Description |
|---|---|---|
| `matrix/current` | `RiskMatrix` | Active matrix definition |
| `matrix/history/{timestamp}` | `RiskMatrix` | Previous versions |
| `findings/{scan-id}` | `VulnerabilityFinding[]` | Raw scan input |
| `scored/{scan-id}` | `ScoredFinding[]` | AI-scored results |
| `tickets/{scan-id}` | `TicketRecord[]` | Created ticket references |

---

## Component 3: AI Scoring Script

### Data Model: Vulnerability Finding (Input)

```typescript
interface VulnerabilityFinding {
  cve_id: string;                  // "CVE-2024-1234"
  description: string;
  affected_hosts: string[];        // ["10.0.0.1", "app-server-01"]
  cvss_score?: number;             // Optional: 0.0 – 10.0
  cvss_vector?: string;
  severity_hint?: string;          // "HIGH" from scanner
  references?: string[];           // Links to NVD, vendor advisories
}
```

### Data Model: Scored Finding (Output)

```typescript
interface ScoredFinding extends VulnerabilityFinding {
  ai_classification: {
    likelihood_label: string;      // Matches a label from the matrix
    impact_label: string;
    reasoning: string;             // AI explanation (1–2 sentences)
  };
  computed_severity: string;       // "Critical" | "High" | "Medium" | "Low"
  sla_due_date: string;            // ISO8601 date
  matrix_version: string;         // Which matrix version was applied
  scored_at: string;               // ISO8601 timestamp
}
```

### AI Classification Prompt Pattern

The scorer sends each finding to the AI model with this structured prompt:

```
You are a security risk classifier. Given the following vulnerability, classify it
against the corporate risk matrix.

Risk Matrix Likelihood Levels:
- 1: Rare
- 2: Unlikely
- 3: Possible
- 4: Likely
- 5: Almost Certain

Impact Levels:
- 1: Negligible
- 2: Minor
- 3: Moderate
- 4: Major
- 5: Catastrophic

Vulnerability:
CVE: {cve_id}
Description: {description}
CVSS Score: {cvss_score}
Affected Hosts: {affected_hosts}

Respond ONLY with valid JSON:
{
  "likelihood_index": <1-5>,
  "impact_index": <1-5>,
  "reasoning": "<one sentence explaining your classification>"
}
```

### Lambda vs CLI Mode

The scoring script is generated by `src/scorer/generator.js` which reads
the current matrix and writes `scripts/generated-scorer.js`. This file can
run in two modes:

```bash
# CLI mode
node scripts/generated-scorer.js --input data/findings/scan-001.json

# Lambda mode (handler export present)
# Deploy via: zip lambda/scorer-lambda.zip scripts/generated-scorer.js node_modules/
```

---

## Component 4: Ticketing Integration Layer

### Data Model: Ticket Schema

```typescript
interface TicketPayload {
  title: string;                   // "CVE-2024-1234: {short description}"
  description: string;             // Markdown body
  severity: string;                // "Critical" | "High" | "Medium" | "Low"
  priority: string;                // Mapped from severity
  due_date: string;                // ISO8601, from SLA policy
  labels: string[];                // ["security", "vulnerability", severity]
  custom_fields: {
    cve_id: string;
    affected_hosts: string[];
    cvss_score?: number;
    remediation_steps?: string;
    references?: string[];
  };
}
```

### Ticket Body Template

```markdown
## CVE Summary

**CVE ID:** {cve_id}
**CVSS Score:** {cvss_score}
**Severity:** {severity}
**SLA Due Date:** {due_date}

## Description

{description}

## Affected Hosts

{affected_hosts_list}

## Remediation Steps

{remediation_steps}

## References

{references_list}

---
*Auto-generated by Risk Matrix System on {timestamp}*
```

### Adapter Interface

```javascript
// src/ticketing/interface.js
class AbstractTicketingAdapter {
  async createTicket(payload: TicketPayload): Promise<TicketRecord>
  async updateTicket(id: string, updates: Partial<TicketPayload>): Promise<void>
  async getTicket(id: string): Promise<TicketRecord>
}
```

Supported adapters:
- `adapters/jira.js` — Jira via MCP (initial implementation)
- Future: `adapters/linear.js`, `adapters/servicenow.js`, `adapters/github.js`
