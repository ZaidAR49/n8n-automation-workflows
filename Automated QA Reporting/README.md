# 📊 Automated QA Reporting
This workflow automates the extraction of test results from a Cypress execution log and appends them into a Google Sheet, providing a structured overview of testing outcomes for easy analysis.

---

## Table of Contents
- Overview
- How It Works
- Node Architecture
- Tech Stack
- Prerequisites
- Configuration
- Output
- Edge Cases & Fallbacks

---

## Overview
Triggered by a Cypress test execution log, this workflow extracts detailed results and uploads them into Google Sheets. It ensures that the QA reports are delivered in a structured format that improves accessibility and accuracy in test result reporting.

---

## How It Works
[Connection data unavailable — diagram cannot be generated]
1. The workflow is triggered upon receiving a Cypress execution log.
2. The QA Agent processes the log to extract relevant test details.
3. Rows are appended to a Google Sheets document for reporting.

---

## Node Architecture
| Node Name                                | Type                                    | Role                                         |
|------------------------------------------|-----------------------------------------|----------------------------------------------|
| AI Agent                                 | @n8n/n8n-nodes-langchain.agent          | Parses log and extracts test results         |
| Append row in sheet in Google Sheets     | n8n-nodes-base.googleSheetsTool         | Appends extracted data to Google Sheets      |
| Send a message                           | n8n-nodes-base.slack                   | Sends notifications via Slack                 |
| If                                       | n8n-nodes-base.if                       | Conditional logic for error handling          |
| Send a message1                          | n8n-nodes-base.gmail                    | Sends email notifications if errors occur     |
| OpenAI Chat Model                        | @n8n/n8n-nodes-langchain.lmChatOpenAi | Interacts with OpenAI for responses           |
| GitHub actions                           | n8n-nodes-base.webhook                  | Webhook for GitHub actions                   |
| QA message                               | n8n-nodes-base.code                     | Handles messaging output formatting            |

---

## Tech Stack
| Component                       | Technology                                      |
|--------------------------------|-------------------------------------------------|
| Workflow Automation            | n8n                                            |
| Cloud Storage                  | Google Sheets                                   |
| Messaging                      | Slack, Gmail                                   |
| AI Integration                 | OpenAI                                         |

---

## Prerequisites
- n8n instance v1.0+
- Google Sheets OAuth2 API key required
- Slack account required
- Gmail OAuth2 API required
- [credential: openAiApi — confirm requirement]

---

## Configuration
Users must ensure that the Google Sheets document is correctly set up and accessible. Additionally, configuration of messaging services (Slack and Gmail) is required to send updates on the QA reporting process.

---

## Output
Upon success, the workflow outputs rows of test results into the designated Google Sheet, alongside a summary of the overall test execution status.

---

## Edge Cases & Fallbacks
| Scenario                                        | Behavior                                     |
|------------------------------------------------|----------------------------------------------|
| Missing required log data                       | Workflow fails and no output is generated    |
| Malformed Google Sheets document                | Workflow aborts with a relevant error message |
| API call failure during data appending          | Workflow stops without completing all operations |
