# 🚀 Submit Workflow to GitHub
This workflow automates the submission of n8n workflows to a GitHub repository and generates a README file documenting the workflow.

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
Triggered by a user’s form submission containing a JSON representation of an n8n workflow, it validates, processes, and publishes the workflow to a specified GitHub repository.

---

## How It Works
[ASCII flow diagram]
On form submission → Code in JavaScript → If (Check success) → Message a model → Workflow JSON → readme → success

1. The workflow starts when a user submits a form with their email and workflow JSON.
2. It validates the JSON format and checks for success.
3. If the workflow JSON is valid, an AI-generated README is created.
4. The workflow JSON and README file are then committed to the user's GitHub repository.
5. Finally, confirmation emails are sent to notify the user of the outcome.

---

## Node Architecture
| Node Name           | Type                                         | Role                                                |
|---------------------|----------------------------------------------|----------------------------------------------------|
| On form submission   | n8n-nodes-base.formTrigger                   | Trigger the workflow based on form submission      |
| Code in JavaScript   | n8n-nodes-base.code                          | Validate the workflow JSON                          |
| If                  | n8n-nodes-base.if                            | Check if the workflow JSON validation was successful|
| Message a model     | @n8n/n8n-nodes-langchain.openAi              | Generate README documentation using AI              |
| Workflow JSON       | n8n-nodes-base.github                        | Commit the workflow JSON to GitHub                 |
| readme              | n8n-nodes-base.github                        | Commit the generated README to GitHub              |
| success             | n8n-nodes-base.gmail                         | Notify user of successful publication                |
| Fali                | n8n-nodes-base.gmail                         | Notify user of failure in workflow processing        |

---

## Tech Stack
| Component          | Technology                       |
|--------------------|----------------------------------|
| Trigger Node       | n8n form submission               |
| Code Validation     | JavaScript                       |
| AI Generation      | OpenAI                           |
| GitHub Interaction | GitHub API                       |
| Email Notifications | Gmail API                        |

---

## Prerequisites
- n8n instance v1.0+
- OpenAI API key required
- GitHub OAuth2 credentials
- Gmail OAuth2 credentials

---

## Configuration
Ensure to set the correct GitHub repository name and owner before running the workflow.

---

## Output
Upon successful execution, this workflow generates two files in the specified GitHub repository: the workflow JSON and a README.md file documenting the workflow.

---

## Edge Cases & Fallbacks
| Scenario                                     | Behavior                                               |
|----------------------------------------------|--------------------------------------------------------|
| Missing required form fields                 | User receives an email notification about the error   |
| Malformed workflow JSON input                | User receives an email notification about the error   |
| Workflow JSON cannot be parsed               | User receives an email notification about the error   |
