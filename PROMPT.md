Build a CLI-first application for managing corporate vulnerability risk scoring and ticket generation.

Context:

- This is a security engineering tool used by enterprises
- Focus on simplicity first (CLI, local execution), extensible later
- Prefer modular structure over monolith

High-Level Architecture:

1. CLI Risk Matrix Builder
2. Storage Layer (local + document store)
3. AI Scoring Script Generator
4. Ticketing Integration Layer (start with Jira)

---

Requirements

1. Risk Matrix Builder (CLI)

- Interactive console flow to define:
  - Likelihood levels
  - Impact levels
  - Resulting risk/severity mapping
- Allow definition of corporate policy:
  - SLA / due dates per severity
- Save output as:
  - Local `.xlsx`
  - JSON (for reuse by scoring script)

2. Storage

- Persist matrix locally
- Abstract document store interface (simple implementation first, extensible later)

3. Scoring Script Generator

- Generate a standalone script that can run:
  - Locally (CLI)
  - As a Lambda function
- Script input:
  - Vulnerability findings (CVE, description, affected hosts, severity signals)
- Script behavior:
  - Use a lightweight/free AI reasoning model to classify risk
  - Apply the defined risk matrix to compute final severity

4. Ticket Automation

- For each CVE:
  - Create one ticket
- Populate:
  - CVE details
  - Remediation steps
  - Affected hosts
  - Calculated severity
  - SLA/due date based on policy
- Start with Jira integration via MCP
- Design abstraction layer to support other systems later

---

Process

1. Design data models:
   - Risk matrix schema
   - Vulnerability input schema
   - Ticket schema

2. Implement CLI builder and persistence

3. Generate scoring script using saved matrix

4. Integrate ticket creation (use MCP for Jira)

5. Validate end-to-end:
   - Input findings → AI scoring → matrix mapping → ticket creation

---

Constraints

- Keep implementation simple and iterative
- Avoid over-engineering (no UI, CLI only)
- Make components loosely coupled for future extension

---

Verification Criteria

- User can build and export a valid risk matrix (.xlsx)
- Generated script can process vulnerability input and assign severity
- Tickets are created with correct severity + SLA mapping
- System works end-to-end with minimal configuration

---

Optional Enhancements (only if time permits)

- Support additional ticketing systems
- Add batch processing for large scans
