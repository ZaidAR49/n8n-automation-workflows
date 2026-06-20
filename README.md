# 🤖 n8n Automation Workflows

A curated collection of production-ready n8n automation workflows covering AI agents, QA reporting, fraud detection, and more — built with OpenAI, Google Sheets, Slack, Telegram, GitHub Actions, and other integrations.

---

## 📋 Table of Contents

- [About This Repository](#about-this-repository)
- [Repository Structure](#repository-structure)
- [Workflow Index](#workflow-index)
- [Getting Started](#getting-started)
- [Prerequisites](#prerequisites)
- [How to Import a Workflow](#how-to-import-a-workflow)
- [Credentials & Integrations](#credentials--integrations)
- [Contributing](#contributing)

---

## About This Repository

This repository is a centralized library of n8n workflow automations built for real-world use cases. Each workflow is self-contained in its own folder with:

- The **workflow JSON file** — ready to import directly into n8n
- A **README.md** — documenting how it works, how to set it up, and what it connects to

Workflows are organized by category and range from AI-powered agents to CI/CD integrations, notification pipelines, and data reporting tools.

**Author:** Zaid Ashraf Radaideh
**Stack:** n8n · OpenAI · Google Sheets · Slack · Telegram · Gmail · GitHub Actions · SerpAPI

---

## Repository Structure

```
n8n-automation-workflows/
│
├── README.md                          ← You are here — master index
│
├── ai-agents/
│   └── scam-detection/
│       ├── Scam_detection.json
│       └── README.md
│
├── qa-reporting/
│   └── cypress-qa-report-agent/
│       ├── QA_Agent_Reports.json
│       └── README.md
│
├── notifications/
│   └── ...
│
├── data-pipelines/
│   └── ...
│
└── crm-integrations/
    └── ...
```

Each workflow lives in a subfolder under its category. New workflows are added automatically via the **Workflow Publisher** pipeline (see [Contributing](#contributing)).

---

## Workflow Index

| # | Workflow | Category | Trigger | Tools Used | Description |
|---|---|---|---|---|---|
| 01 | [Scam Detection](./ai-agents/scam-detection/README.md) | AI Agents | Telegram Message | Telegram · OpenAI · SerpAPI | Multi-agent pipeline that analyzes suspicious URLs and promotional messages, scores them 1–10 for fraud risk, and returns an Arabic report via Telegram |
| 02 | [Cypress QA Report Agent](./qa-reporting/cypress-qa-report-agent/README.md) | QA Reporting | GitHub Actions Webhook | OpenAI · Google Sheets · Slack · Gmail | Parses Cypress terminal logs, extracts per-spec test results, writes them to Google Sheets row-by-row, and routes pass/fail notifications to Slack and Gmail |

> More workflows are added regularly. Each new entry is automatically published to this repository.

---

## Getting Started

### Prerequisites

- **n8n instance** — self-hosted or [n8n Cloud](https://n8n.io/cloud/). All workflows are built on n8n v1.0+.
- **API credentials** for the services used by each workflow (see each workflow's own README for specifics).
- A **GitHub account** if you plan to fork and extend these workflows.

### How to Import a Workflow

1. Open your n8n instance.
2. Navigate to **Workflows → Import from File**.
3. Select the `.json` file from the workflow's folder.
4. Open each node and assign the correct credentials.
5. Update any hardcoded IDs (channel IDs, sheet IDs, email addresses) to match your environment.
6. Activate the workflow.

> Each workflow's README contains a detailed step-by-step setup section.

---

## Credentials & Integrations

The following services are used across workflows in this repository. Set them up once in n8n under **Settings → Credentials** and reuse them across workflows.

| Service | Used By | Credential Type |
|---|---|---|
| OpenAI | All AI Agent workflows | OpenAI API Key |
| Telegram | Scam Detection | Telegram Bot Token |
| SerpAPI | Scam Detection | SerpAPI Key |
| Google Sheets | QA Report Agent | Google Sheets OAuth2 |
| Slack | QA Report Agent | Slack Bot OAuth Token |
| Gmail | QA Report Agent | Gmail OAuth2 |
| GitHub | Workflow Publisher *(coming soon)* | GitHub Personal Access Token |

---

## Contributing

New workflows are published to this repository automatically using the **n8n Workflow Publisher** pipeline *(coming soon)*. The publisher takes a workflow JSON, generates a README using an AI Agent, and commits both files to the correct category folder in this repository — no manual steps required.

To add a workflow manually in the meantime:

1. Create a new folder under the appropriate category: `category-name/workflow-name/`
2. Add the workflow JSON file.
3. Add a `README.md` documenting the workflow (see existing READMEs as a template).
4. Add a row to the [Workflow Index](#workflow-index) table above.

---

## License

MIT — free to use, fork, and adapt for your own automation projects.
