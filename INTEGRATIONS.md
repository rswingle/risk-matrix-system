# Integrations — Risk Matrix System

## Overview

The Risk Matrix System integrates with four external surfaces: the local
filesystem for persistence, an AI reasoning model for CVE classification,
Jira via MCP for ticket creation, and AWS Lambda for serverless scoring deployments.
All integrations are optional except the filesystem — the core pipeline degrades
gracefully when any integration is unavailable.

---

## Integration Inventory

| Integration | Type | Component | Required |
|---|---|---|---|
| Local filesystem | Native I/O | All | Yes |
| AI model (OpenAI-compatible) | HTTP API | Scorer | Yes (for scoring) |
| Jira MCP | MCP server | Ticketing | Yes (for tickets) |
| AWS Lambda | Deployment target | Scorer | No (optional deploy) |
| Excel export | npm (`xlsx`) | Builder | Yes (for .xlsx output) |

---

## 1. Local Filesystem

**Type:** Native Node.js `fs`
**Used by:** All components via the Storage Layer abstraction

The filesystem is the only strictly required integration. All data is stored
under `/data/` as JSON files. The storage layer reads and writes through the
abstract interface — switching to a different store (S3, DynamoDB) requires
only a new adapter, not changes to any other component.

**Initialization:**
```bash
mkdir -p data/findings data/scored data/tickets
```

**Key conventions:** See `DESIGN.md → Storage Layer → Storage Key Conventions`.

---

## 2. AI Reasoning Model

**Type:** HTTP API (OpenAI-compatible `/v1/chat/completions`)
**Used by:** `src/scorer/classifier.js`
**Required:** Yes, for the scoring workflow

The scorer is model-agnostic — any endpoint implementing the OpenAI chat
completions API is supported. Recommended options ranked by cost and
infrastructure requirement:

| Model | Cost | Requires | Notes |
|---|---|---|---|
| Ollama (local) | Free | Local GPU/CPU | Best for air-gapped environments |
| Groq (API) | Free tier | API key | Fast inference, good free quota |
| OpenAI GPT-4o-mini | ~$0.001/CVE | API key | Highest classification quality |
| AWS Bedrock Claude | Pay-per-use | AWS account | Use when running as Lambda |

**Configuration (`config/settings.json`):**

```json
{
  "ai": {
    "endpoint": "http://localhost:11434/v1",
    "model": "llama3.1",
    "api_key": "",
    "timeout_ms": 30000,
    "max_retries": 2
  }
}
```

For cloud models, set `endpoint` to the provider's base URL and provide an
`api_key`. The classifier uses the same request structure regardless of provider.

**Failure modes:**

| Failure | Behavior |
|---|---|
| Model unreachable | Log error, mark finding as `severity: "Unscored"`, continue |
| Malformed JSON response | Retry up to `max_retries`, then mark as Unscored |
| Index out of range | Log, mark Unscored |
| Timeout | Abort after `timeout_ms`, mark Unscored |

All Unscored findings are written to `data/scored/{scan-id}.json` with a
`needs_review: true` flag and printed to console at the end of the run.

---

## 3. Jira MCP Integration

**Type:** MCP (Model Context Protocol) server
**Used by:** `src/ticketing/adapters/jira.js`
**MCP server:** `jira` (Claude Code's Jira MCP integration)

The Jira adapter creates one ticket per scored finding using the Jira MCP server.
It maps the internal `TicketPayload` schema to Jira's issue structure.

**Environment variables:**

```bash
JIRA_BASE_URL=https://your-org.atlassian.net
JIRA_PROJECT_KEY=SEC
JIRA_ISSUE_TYPE=Bug                    # or "Task", "Security Vulnerability"
JIRA_API_TOKEN=your-api-token          # Atlassian API token
JIRA_USER_EMAIL=you@company.com
```

**Severity → Jira Priority mapping:**

| Risk Matrix Severity | Jira Priority |
|---|---|
| Critical | Highest |
| High | High |
| Medium | Medium |
| Low | Low |
| Informational | Lowest |

**Jira custom fields used:**

| Field | Internal mapping |
|---|---|
| `summary` | `"CVE-XXXX: {short description}"` |
| `description` | Full Markdown ticket body |
| `priority.name` | Mapped from severity |
| `duedate` | ISO8601 SLA due date |
| `labels` | `["security", "vulnerability", severity.toLowerCase()]` |
| `customfield_cve` | `cve_id` (if custom field exists in project) |

**MCP tools called:**

```javascript
// Create issue
await mcp.call("jira_create_issue", {
  project: process.env.JIRA_PROJECT_KEY,
  issuetype: process.env.JIRA_ISSUE_TYPE,
  summary: payload.title,
  description: payload.description,
  priority: { name: jiraPriority },
  duedate: payload.due_date,
  labels: payload.labels
});
```

**Failure handling:** If Jira MCP is unavailable, the adapter writes the full
`TicketPayload` to `data/tickets/pending/{scan-id}.json` so tickets can be
created manually or retried later.

---

## 4. AWS Lambda Deployment (Optional)

**Type:** Serverless deployment target
**Used by:** `src/scorer/generator.js`

The generator produces `scripts/generated-scorer.js` as a valid Lambda handler.
The same file runs as a CLI script or as a Lambda function — mode is detected
at runtime.

**Handler signature:**
```javascript
// Lambda invocation
exports.handler = async (event) => {
  const findings = event.findings;   // VulnerabilityFinding[]
  const scored = await scoreAll(findings);
  return { statusCode: 200, body: JSON.stringify(scored) };
};

// CLI invocation
if (require.main === module) {
  const findings = JSON.parse(fs.readFileSync(process.argv[3]));
  scoreAll(findings).then(console.log);
}
```

**Packaging:**
```bash
node src/scorer/generator.js          # Regenerate from current matrix
cd scripts
zip -r ../lambda/scorer-lambda.zip generated-scorer.js ../../node_modules/
```

**Lambda environment variables:**
```
AI_ENDPOINT=https://api.groq.com/openai/v1
AI_MODEL=llama-3.1-8b-instant
AI_API_KEY=gsk_...
```

The Lambda version reads the matrix from its own bundled copy (embedded at
generation time) rather than from the local filesystem.

---

## 5. Excel Export (`xlsx` npm package)

**Type:** npm library (`xlsx` / `SheetJS`)
**Used by:** `src/builder/exporter.js`

```bash
npm install xlsx
```

The exporter generates a two-sheet workbook:
- **Sheet 1:** Color-coded risk matrix grid
- **Sheet 2:** SLA policy table

Cell background colors are applied using SheetJS cell styles. The workbook is
written to `data/matrix.xlsx`.

---

## Adding Future Ticketing Adapters

All ticketing adapters implement the same interface from `src/ticketing/interface.js`.
Adding a new system requires only a new adapter file:

```bash
# Create adapter
touch src/ticketing/adapters/linear.js
# Implement: createTicket(), updateTicket(), getTicket()
# Register in src/ticketing/index.js adapter factory
```

Planned adapters:

| System | MCP / API | Notes |
|---|---|---|
| Linear | Linear MCP | Good fit for eng-led security teams |
| GitHub Issues | GitHub MCP | Use for open-source or GH-native teams |
| ServiceNow | REST API | Enterprise ITSM integration |
| PagerDuty | REST API | For critical severity auto-escalation |

---

## Integration Dependency Map

```
src/builder/
  └── uses → xlsx (npm)             [Excel export]
  └── writes → data/matrix.json     [filesystem]
  └── writes → data/matrix.xlsx     [filesystem]

src/scorer/classifier.js
  └── calls → AI model HTTP API     [HTTP/OpenAI-compatible]

src/scorer/mapper.js
  └── reads → data/matrix.json      [filesystem]

src/scorer/generator.js
  └── writes → scripts/generated-scorer.js  [filesystem]
  └── writes → lambda/scorer-lambda.zip     [filesystem]

src/ticketing/adapters/jira.js
  └── calls → Jira MCP server       [MCP]

src/storage/local.js
  └── reads/writes → data/          [filesystem]
```
