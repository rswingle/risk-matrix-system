# Implementation Plan — Risk Matrix System

## Overview
This plan outlines the step-by-step implementation of the Risk Matrix System, an enterprise vulnerability risk scoring and ticket generation tool. The plan is based on the requirements, architecture, and data flows defined in the project's markdown documentation (`PROMPT.md`, `ARCHITECTURE.md`, `DATA_FLOW.md`, `INTEGRATIONS.md`, and `README.md`).

## Phase 1: Foundation & Data Modeling
**Objective:** Define the core data schemas and set up the project structure.

1. **Initialize Project:**
   - Run `npm init` and configure `package.json`.
   - Set up the directory structure as defined in `ARCHITECTURE.md` (`src/builder`, `src/storage`, `src/scorer`, `src/ticketing`, `config`, `data`, `scripts`, `lambda`).
   - Create `config/settings.example.json` and `config/settings.json`.

2. **Design Data Models:**
   - **Risk Matrix Schema:** Define the structure for likelihood levels, impact levels, severity grid, and SLA policy.
   - **Vulnerability Input Schema:** Define the input format for CVEs (e.g., `cve_id`, `description`, `affected_hosts`, `cvss_score`, `cvss_vector`, `severity_hint`, `references`).
   - **Ticket Schema:** Define the payload structure for ticketing (e.g., `TicketPayload` and `TicketRecord`).

3. **Implement Storage Layer (`src/storage/`):**
   - Create the abstract interface `src/storage/interface.js`.
   - Implement local filesystem storage `src/storage/local.js` to handle reading/writing JSON files to the `/data/` directory.

## Phase 2: CLI Risk Matrix Builder
**Objective:** Build the interactive console application to define the corporate risk matrix.

1. **Interactive Prompts (`src/builder/prompts.js` & `index.js`):**
   - Integrate an interactive CLI tool (e.g., `inquirer`).
   - Create prompts for Likelihood levels, Impact levels, Severity grid mapping, and SLA policy.

2. **Matrix Construction & Validation (`src/builder/matrix.js`):**
   - Validate that all grid cells are assigned, SLAs are positive integers, and there are no duplicate labels.

3. **Export Functionality (`src/builder/exporter.js`):**
   - Implement JSON export to `data/matrix.json`.
   - Integrate `xlsx` package to export a color-coded spreadsheet to `data/matrix.xlsx`.

## Phase 3: AI Scoring Script Generator
**Objective:** Generate a standalone script that scores vulnerabilities using the saved risk matrix and an AI model.

1. **AI Classifier Integration (`src/scorer/classifier.js`):**
   - Implement an OpenAI-compatible API client to handle requests to local (Ollama) or cloud (Groq, OpenAI) endpoints.
   - Design the prompt to classify vulnerabilities into likelihood and impact indices based on the matrix.
   - Handle API errors, retries, timeouts, and malformed JSON responses.

2. **Matrix Application Logic (`src/scorer/mapper.js`):**
   - Map AI-classified indices (likelihood × impact) to the severity label.
   - Compute the SLA due date based on the severity policy.

3. **Script Generator (`src/scorer/generator.js`):**
   - Write a generator that bundles the matrix definition and scoring logic into a standalone script (`scripts/generated-scorer.js`).
   - Ensure the generated script supports both CLI execution and AWS Lambda handler execution.

## Phase 4: Ticketing Integration Layer
**Objective:** Automate Jira ticket creation for scored vulnerabilities.

1. **Ticketing Abstraction (`src/ticketing/interface.js` & `schema.js`):**
   - Define the `TicketPayload` schema and markdown rendering logic.
   - Create the abstract ticketing adapter interface.

2. **Jira MCP Adapter (`src/ticketing/adapters/jira.js`):**
   - Integrate with the Jira MCP server.
   - Map calculated severities to Jira priorities.
   - Implement the `createTicket` function to send payloads to Jira and retrieve ticket IDs.

3. **Ticketing Pipeline (`src/ticketing/index.js`):**
   - Read scored findings from the storage layer.
   - Iterate through findings, call the Jira adapter, and write the resulting `TicketRecord`s back to storage.
   - Implement fallback mechanisms (e.g., saving pending tickets if MCP is unavailable).

## Phase 5: End-to-End Validation
**Objective:** Verify the complete system flow.

1. Run the builder to generate `data/matrix.json` and `data/matrix.xlsx`.
2. Generate the scoring script (`node src/scorer/generator.js`).
3. Prepare a sample findings file (`data/findings/scan-001.json`).
4. Execute the scoring script and verify `data/scored/scan-001.json` is created with correct severities and SLAs.
5. Run the ticketing integration and verify Jira tickets are created and recorded in `data/tickets/scan-001.json`.

## Phase 6: Optional Enhancements (Time Permitting)
1. Add batch processing logic for handling large vulnerability scans.
2. Implement additional ticketing adapters (e.g., Linear, GitHub Issues).
3. Automate the AWS Lambda packaging and deployment scripts.