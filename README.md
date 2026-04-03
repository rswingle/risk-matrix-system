# Risk Matrix System

A CLI-first security engineering tool for enterprise vulnerability risk scoring
and automated ticket generation. Define a corporate risk matrix once, then run
AI-assisted scoring on vulnerability findings to compute severity and SLA dates —
and create Jira tickets automatically.

---

## What It Does

1. **Define your risk matrix** — Set likelihood/impact levels and SLA policy via
   interactive CLI. Export to `.xlsx` and `.json`.

2. **Score vulnerability findings** — Feed CVE data through an AI classifier that
   applies your matrix to compute a final severity and due date per finding.

3. **Create tickets automatically** — For each scored CVE, create a Jira ticket
   pre-populated with CVE details, affected hosts, remediation steps, severity,
   and SLA due date.

4. **Deploy as Lambda** — The scoring script is auto-generated and deployable to
   AWS Lambda for serverless pipeline integration.

---

## Quick Start

### Prerequisites

- Node.js 18+
- An AI model endpoint (local Ollama, Groq, or OpenAI-compatible API)
- Jira access + API token (for ticket creation)

### Install

```bash
git clone https://github.com/your-org/risk-matrix-system
cd risk-matrix-system
npm install
mkdir -p data/findings data/scored data/tickets
cp config/settings.example.json config/settings.json
# Edit config/settings.json with your AI endpoint + Jira credentials
```

### Step 1: Build Your Risk Matrix

```bash
node src/builder/index.js
```

Follow the interactive prompts to define likelihood levels, impact levels, the
severity grid, and SLA policy. When done, the matrix is saved to:
- `data/matrix.json` (machine-readable, used by scorer)
- `data/matrix.xlsx` (human-readable, for stakeholders)

### Step 2: Generate the Scoring Script

```bash
node src/scorer/generator.js
```

Reads your saved matrix and writes `scripts/generated-scorer.js` — a standalone
script (and Lambda handler) baked with your current matrix.

### Step 3: Score Vulnerability Findings

Prepare a findings file at `data/findings/scan-001.json` (see format below), then:

```bash
node scripts/generated-scorer.js --input data/findings/scan-001.json
```

Scored results are saved to `data/scored/scan-001.json`.

### Step 4: Create Jira Tickets

```bash
node src/ticketing/index.js --scan scan-001
```

Creates one Jira ticket per scored finding with full context, severity, and SLA
due date populated.

---

## Input Format

Vulnerability findings are JSON arrays following this schema:

```json
[
  {
    "cve_id": "CVE-2024-1234",
    "description": "Remote code execution in libssl via crafted TLS handshake.",
    "affected_hosts": ["10.0.1.5", "api-server-prod-01"],
    "cvss_score": 9.8,
    "cvss_vector": "AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H",
    "severity_hint": "CRITICAL",
    "references": [
      "https://nvd.nist.gov/vuln/detail/CVE-2024-1234"
    ]
  }
]
```

All fields except `cve_id`, `description`, and `affected_hosts` are optional but
improve classification accuracy when provided.

---

## Configuration

**`config/settings.json`:**

```json
{
  "ai": {
    "endpoint": "http://localhost:11434/v1",
    "model": "llama3.1",
    "api_key": "",
    "timeout_ms": 30000,
    "max_retries": 2
  },
  "storage": {
    "type": "local",
    "base_path": "./data"
  }
}
```

**Environment variables (for Jira):**

```bash
export JIRA_BASE_URL=https://your-org.atlassian.net
export JIRA_PROJECT_KEY=SEC
export JIRA_ISSUE_TYPE=Bug
export JIRA_API_TOKEN=your-api-token
export JIRA_USER_EMAIL=you@company.com
```

---

## AI Model Options

The scorer works with any OpenAI-compatible endpoint. Recommended options:

| Option | Cost | Setup |
|---|---|---|
| [Ollama](https://ollama.com) (local) | Free | `ollama run llama3.1` |
| [Groq](https://groq.com) | Free tier | Set `endpoint` + `api_key` |
| OpenAI GPT-4o-mini | ~$0.001/CVE | Set `endpoint` + `api_key` |

For air-gapped environments, Ollama is the recommended approach.

---

## Lambda Deployment

The generated scorer runs as an AWS Lambda function without modification:

```bash
# Generate (embeds current matrix)
node src/scorer/generator.js

# Package
cd scripts
zip -r ../lambda/scorer-lambda.zip generated-scorer.js ../../node_modules/

# Deploy
aws lambda create-function \
  --function-name risk-matrix-scorer \
  --runtime nodejs18.x \
  --handler generated-scorer.handler \
  --zip-file fileb://../lambda/scorer-lambda.zip \
  --environment Variables="{AI_ENDPOINT=...,AI_API_KEY=...}"
```

**Lambda invocation payload:**
```json
{
  "findings": [ ... ]
}
```

**Lambda response:**
```json
{
  "statusCode": 200,
  "body": [ ... ]
}
```

---

## Project Structure

```
risk-matrix-system/
├── config/
│   └── settings.json              # AI + storage config
├── data/
│   ├── matrix.json                # Active risk matrix
│   ├── matrix.xlsx                # Exported spreadsheet
│   ├── findings/                  # Raw CVE input files
│   ├── scored/                    # AI-scored results
│   └── tickets/                   # Created ticket records
├── scripts/
│   └── generated-scorer.js        # Auto-generated scoring script
├── lambda/
│   └── scorer-lambda.zip          # Lambda deployment package
└── src/
    ├── builder/                   # Interactive matrix builder
    ├── storage/                   # Abstract storage layer
    ├── scorer/                    # AI scoring + script generator
    └── ticketing/                 # Jira + adapter framework
```

---

## Verification Checklist

Use this checklist to confirm end-to-end functionality:

- [ ] `node src/builder/index.js` completes and writes `data/matrix.json`
- [ ] `data/matrix.xlsx` opens correctly with color-coded grid and SLA sheet
- [ ] `node src/scorer/generator.js` writes `scripts/generated-scorer.js`
- [ ] `node scripts/generated-scorer.js --input data/findings/scan-001.json`
      writes scored output to `data/scored/scan-001.json`
- [ ] Each finding in scored output has `computed_severity` and `sla_due_date`
- [ ] `node src/ticketing/index.js --scan scan-001` creates one Jira ticket per CVE
- [ ] Each Jira ticket has correct severity, due date, and description populated
- [ ] Re-running builder with updated SLA → regenerate scorer → verify new due dates

---

## Extending the System

**Add a new ticketing adapter:**
1. Create `src/ticketing/adapters/{system}.js`
2. Implement `createTicket()`, `updateTicket()`, `getTicket()`
3. Register in `src/ticketing/index.js`

**Swap the storage backend:**
1. Create `src/storage/{backend}.js` implementing the abstract interface
2. Update `config/settings.json → storage.type`

**Add a new template type:**
Modify `src/ticketing/schema.js` to add conditional sections
(e.g., patch guidance, CVSS breakdown) based on finding metadata.

---

## Design Documents

| Document | Description |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System architecture and component breakdown |
| [DESIGN.md](./DESIGN.md) | Data models, CLI flows, template design |
| [DATA_FLOW.md](./DATA_FLOW.md) | End-to-end data flow diagrams |
| [INTEGRATIONS.md](./INTEGRATIONS.md) | AI, Jira, Lambda, and Excel integration reference |

---

## License

MIT
