# Architecture — Risk Matrix System

## Overview

The Risk Matrix System is a CLI-first, modular security engineering tool for
enterprise vulnerability risk scoring and ticket generation. It is composed of
four loosely-coupled components that can be used independently or as an end-to-end
pipeline.

---

## System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Security Engineer (CLI)                    │
└────────────────────────────┬─────────────────────────────────────┘
                             │
              ┌──────────────▼──────────────┐
              │     CLI Risk Matrix Builder  │  ← Define likelihood,
              │     (Interactive console)    │    impact, SLA policy
              └──────────────┬──────────────┘
                             │ matrix.json / matrix.xlsx
              ┌──────────────▼──────────────┐
              │       Storage Layer          │  ← Local files +
              │  (Abstract document store)   │    extensible store
              └──────────┬──────────┬────────┘
                         │          │
           matrix.json   │          │  vulnerability input
                         ▼          ▼
              ┌──────────────────────────────┐
              │    AI Scoring Script          │  ← Classify CVEs,
              │ (CLI or Lambda function)      │    apply matrix,
              │                              │    compute severity
              └──────────────┬───────────────┘
                             │ scored findings
              ┌──────────────▼──────────────┐
              │   Ticketing Integration      │  ← Create one ticket
              │   Layer (Jira via MCP)       │    per CVE with full
              │   [Abstraction layer]        │    context + SLA
              └─────────────────────────────┘
```

---

## Component Breakdown

### 1. CLI Risk Matrix Builder

**Responsibility:** Collect, validate, and persist the corporate risk matrix.

- Interactive console flow using `inquirer` or equivalent
- Defines likelihood levels (e.g., Rare → Certain)
- Defines impact levels (e.g., Negligible → Critical)
- Maps likelihood × impact combinations to final severity labels
- Captures SLA/due date policy per severity level
- Exports:
  - `matrix.json` — machine-readable, consumed by the scoring script
  - `matrix.xlsx` — human-readable, shareable with stakeholders

**Module:** `src/builder/`

---

### 2. Storage Layer

**Responsibility:** Persist all system artifacts with a clean abstraction so the
backing store can be swapped later.

- Default implementation: local filesystem (`/data/`)
- Interface: `read(key)`, `write(key, value)`, `list(prefix)`
- Future implementations: DynamoDB, MongoDB, S3 — all behind the same interface
- Stores: matrix definitions, run history, scored findings, ticket records

**Module:** `src/storage/`

---

### 3. AI Scoring Script Generator

**Responsibility:** Generate and execute a standalone script that scores
vulnerability findings using the saved risk matrix.

- Accepts vulnerability findings as JSON input (CVE, description, hosts, signals)
- Uses a lightweight/free AI reasoning model (e.g., local Ollama, Groq, or
  OpenAI-compatible endpoint) to classify severity signals
- Applies the corporate risk matrix to compute final severity
- Deployable as:
  - A local CLI (`node score.js --input findings.json`)
  - An AWS Lambda function (packaged via the same generator)

**Module:** `src/scorer/`

---

### 4. Ticketing Integration Layer

**Responsibility:** Create one ticket per CVE, populated with full context and SLA.

- Reads scored findings from the storage layer
- Maps each finding to a ticket schema
- Creates Jira tickets via Claude Code's Jira MCP integration
- Abstraction layer allows future adapters (Linear, ServiceNow, GitHub Issues)

**Module:** `src/ticketing/`

---

## Directory Structure

```
risk-matrix-system/
├── PROMPT.md                    # Original system spec
├── ARCHITECTURE.md              # This file
├── DESIGN.md                    # Component design + data models
├── DATA_FLOW.md                 # End-to-end data flow
├── INTEGRATIONS.md              # Integration reference
├── README.md                    # Getting started guide
│
├── src/
│   ├── builder/
│   │   ├── index.js             # CLI entry point
│   │   ├── prompts.js           # inquirer prompt definitions
│   │   ├── matrix.js            # Matrix construction logic
│   │   └── exporter.js          # .xlsx and .json export
│   │
│   ├── storage/
│   │   ├── index.js             # Storage factory
│   │   ├── interface.js         # Abstract store interface
│   │   └── local.js             # Local filesystem implementation
│   │
│   ├── scorer/
│   │   ├── index.js             # CLI scorer entry point
│   │   ├── generator.js         # Lambda script generator
│   │   ├── classifier.js        # AI model integration
│   │   └── mapper.js            # Matrix application logic
│   │
│   └── ticketing/
│       ├── index.js             # Ticket pipeline entry point
│       ├── interface.js         # Abstract ticketing interface
│       ├── adapters/
│       │   └── jira.js          # Jira MCP adapter
│       └── schema.js            # Ticket schema + validation
│
├── data/
│   ├── matrix.json              # Active risk matrix
│   ├── matrix.xlsx              # Exported spreadsheet
│   ├── findings/                # Raw vulnerability inputs
│   └── scored/                  # AI-scored findings
│
├── scripts/
│   └── generated-scorer.js     # Auto-generated scoring script
│
├── lambda/
│   └── scorer-lambda.zip       # Packaged Lambda deployment
│
└── config/
    └── settings.json            # AI model endpoint, Jira config
```

---

## Design Principles

1. **CLI-only, no UI** — Every workflow is terminal-driven
2. **Modular, not monolithic** — Each component has a single responsibility and clean interface
3. **Local-first** — Works entirely offline; integrations are additive
4. **Abstract early** — Storage and ticketing interfaces are defined generically from day one
5. **Generate, don't hardcode** — The scoring script is generated from the saved matrix, not pre-written
6. **Iterative delivery** — Each component can be built, tested, and used independently
