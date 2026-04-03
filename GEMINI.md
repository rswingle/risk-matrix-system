# Risk Matrix System

## Project Overview
The Risk Matrix System is a CLI-first, modular security engineering tool designed for enterprise vulnerability risk scoring and automated ticket generation. It allows security teams to:
1. Define a corporate risk matrix (likelihood/impact levels and SLA policies).
2. Score vulnerability findings (CVEs) using an AI classifier against the defined matrix.
3. Automatically create Jira tickets pre-populated with context, calculated severity, and SLA due dates.
4. Deploy the scoring logic as a standalone script or an AWS Lambda function.

The system is composed of four loosely-coupled components:
- **CLI Risk Matrix Builder (`src/builder/`)**: Interactive console for defining the matrix and exporting to JSON/Excel.
- **Storage Layer (`src/storage/`)**: Abstract document store interface (currently local filesystem at `/data/`).
- **AI Scoring Script Generator (`src/scorer/`)**: Generates a standalone, Lambda-ready script that classifies CVEs using OpenAI-compatible APIs (e.g., Ollama, Groq, GPT-4o-mini).
- **Ticketing Integration Layer (`src/ticketing/`)**: Abstraction layer for creating tickets via MCP (currently Jira).

## Building and Running

### Prerequisites
- Node.js 18+
- An AI model endpoint (local Ollama, Groq, or OpenAI-compatible API)
- Jira access + API token (for ticket creation)

### Setup
```bash
npm install
mkdir -p data/findings data/scored data/tickets
cp config/settings.example.json config/settings.json
# Edit config/settings.json with your AI endpoint + Jira credentials
```

### Standard Workflow
1. **Build the Risk Matrix**:
   ```bash
   node src/builder/index.js
   ```
2. **Generate the Scoring Script**:
   ```bash
   node src/scorer/generator.js
   ```
3. **Score Vulnerabilities**:
   Prepare a findings file at `data/findings/scan-001.json`, then run:
   ```bash
   node scripts/generated-scorer.js --input data/findings/scan-001.json
   ```
4. **Create Jira Tickets**:
   ```bash
   node src/ticketing/index.js --scan scan-001
   ```

### Lambda Deployment (Optional)
The generated scoring script can be deployed to AWS Lambda:
```bash
node src/scorer/generator.js
cd scripts
zip -r ../lambda/scorer-lambda.zip generated-scorer.js ../../node_modules/
# Use AWS CLI to create/update the function using scorer-lambda.zip
```

## Development Conventions & Design Principles

- **CLI-Only**: All workflows are terminal-driven. There is no graphical user interface.
- **Modularity & Loose Coupling**: Each component has a single responsibility.
- **Local-First**: The system works entirely offline where possible (e.g., using local Ollama for AI). Integrations are additive.
- **Abstract Early**: Storage and ticketing utilize generic interfaces, allowing easy swapping of backends (e.g., S3 for storage, Linear/ServiceNow for ticketing).
- **Generation over Hardcoding**: The scoring script is generated dynamically based on the saved matrix rather than hardcoding policy rules.
- **Iterative Delivery**: Each component is designed to be built, tested, and used independently.
