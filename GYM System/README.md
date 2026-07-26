# 🏋️ GYM Subscription Management System
This workflow is a fully conversational AI-powered gym CRM bot operating over Telegram. It enables authenticated gym administrators to register new subscriptions, cancel existing ones, and generate analytical reports — all through natural Arabic-language Telegram commands, with no manual spreadsheet editing required.

---

## Table of Contents
- Overview
- How It Works
- Pipeline Flow
- Node Architecture
- Agent Hierarchy
- Google Sheets Schema
- Available Commands
- Tech Stack
- Prerequisites
- Configuration
- Output
- Edge Cases & Fallbacks

---

## Overview

The system is built around a **hierarchical multi-agent architecture**: a Supervisor Agent acts as the sole entry point, authenticates the admin, and routes every incoming Telegram message to the correct specialized sub-agent. Each sub-agent operates as a single-turn state machine with its own isolated session memory, its own Gemini language model instance, and its own set of Google Sheets tools.

All interaction is in Arabic. All data is persisted in a Google Sheets document titled **GYM Subscriptions Management**.

---

## How It Works

1. The admin sends a message to the Telegram bot.
2. An authentication gate checks the sender's chat ID against a whitelist — unauthorized users are immediately rejected.
3. The **Supervisor Agent** reads the message and routes it to the correct sub-agent based on the command or the current active route in session memory.
4. The matched sub-agent performs exactly one action per invocation (state machine pattern), collects data across turns, and writes to or reads from Google Sheets.
5. For the `/analysis` command, a nested `html report` AgentTool generates a full HTML report when requested.
6. A JavaScript code node detects whether the final output is HTML or plain text.
7. The response is sent back to the admin via Telegram — either as a text message or as an HTML document.

---

## Pipeline Flow

```
Telegram Trigger
      │
      ▼
Authentication Gate (IF: chat.id whitelist)
      │ Authorized                    │ Unauthorized
      ▼                               ▼
Supervisor Agent                 ⛔️ Send rejection message
  ├── Memory: sup_<chat_id>
  ├── LLM: Google Vertex (Gemini)
  │
  ├──[/new_subscription]──────────▶ AgentTool: new subscription
  │                                   ├── Memory: add_<chat_id>
  │                                   ├── LLM: Google Vertex3
  │                                   ├── Tool: get user       (read all rows)
  │                                   ├── Tool: get dates      (JS date calculator)
  │                                   └── Tool: Append row     (write new row)
  │
  ├──[/cancel_subscription]───────▶ AgentTool: cancel subscription
  │                                   ├── Memory: add_<chat_id>
  │                                   ├── LLM: Google Vertex1
  │                                   └── Tool: update         (update Status column)
  │
  └──[/analysis | gym Q&A]────────▶ AgentTool: analysis
                                      ├── Memory: Simple Memory3
                                      ├── LLM: Google Vertex2
                                      ├── Tool: get rows
                                      ├── Tool: Calculator
                                      ├── Tool: Date & Time
                                      └── Tool: html report    (nested AgentTool)
                                                ├── LLM: gemini-2.5-pro
                                                ├── Tool: get rows
                                                ├── Tool: Calculator
                                                └── Tool: Date & Time
      │
      ▼
Code in JavaScript (detect HTML vs plain text)
      │
   Is HTML?
   ├── Yes ──▶ Send HTML Report → Telegram (document)
   └── No  ──▶ Response to user → Telegram (text message)
```

---

## Node Architecture

| Node Name | Type | Role |
|---|---|---|
| Telegram Trigger | n8n-nodes-base.telegramTrigger | Captures all incoming Telegram messages |
| Authentication | n8n-nodes-base.if | Whitelists admin by chat ID — rejects all others |
| Send a text message | n8n-nodes-base.telegram | Sends unauthorized access rejection message |
| Supervisor agent | @n8n/n8n-nodes-langchain.agent | Main router — reads commands and delegates to sub-agents |
| Google Vertex | @n8n/n8n-nodes-langchain.lmChatGoogleVertex | LLM powering the Supervisor agent |
| Simple Memory1 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Session memory for Supervisor (keyed: `sup_<chat_id>`) |
| new subscription | @n8n/n8n-nodes-langchain.agentTool | 7-branch state machine: collects name, phone, duration, amount → writes to Sheets |
| Google Vertex3 | @n8n/n8n-nodes-langchain.lmChatGoogleVertex | LLM powering the new subscription agent |
| Simple Memory2 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Session memory for new subscription (keyed: `add_<chat_id>`) |
| get user | n8n-nodes-base.googleSheetsTool | Reads all rows to check for duplicate active subscribers and fetch last ID |
| get dates | @n8n/n8n-nodes-langchain.toolCode | Custom JS tool: computes start/end dates from duration in months |
| Append row | n8n-nodes-base.googleSheetsTool | Appends a new subscriber row to Google Sheets |
| cancel subscription | @n8n/n8n-nodes-langchain.agentTool | 3-branch state machine: collects name → updates Status to `ملغي` |
| Google Vertex1 | @n8n/n8n-nodes-langchain.lmChatGoogleVertex | LLM powering the cancel subscription agent |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Session memory for cancel subscription (keyed: `add_<chat_id>`) |
| update | n8n-nodes-base.googleSheetsTool | Updates the Status column for a matched subscriber row |
| analysis | @n8n/n8n-nodes-langchain.agentTool | Answers gym-related questions and triggers HTML report generation |
| Google Vertex2 | @n8n/n8n-nodes-langchain.lmChatGoogleVertex | LLM powering the analysis agent |
| Simple Memory3 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Session memory for analysis agent |
| get rows | n8n-nodes-base.googleSheetsTool | Reads all subscription rows for analysis and reporting |
| Calculator | @n8n/n8n-nodes-langchain.toolCalculator | Performs arithmetic for statistics and aggregations |
| Date & Time | @n8n/n8n-nodes-langchain.toolDatetime | Handles date calculations for expiry analysis |
| html report | @n8n/n8n-nodes-langchain.agentTool | Nested agent: generates a full HTML subscription report |
| gemini-2.5-pro | @n8n/n8n-nodes-langchain.lmChatGoogleVertex | High-capability LLM powering the HTML report agent |
| Code in JavaScript | n8n-nodes-base.code | Detects whether the Supervisor output is HTML or plain text |
| Is HTML | n8n-nodes-base.if | Routes to HTML sender or text sender based on output type |
| Response to user | n8n-nodes-base.telegram | Sends plain Arabic text reply to the admin |
| Send HTML report | n8n-nodes-base.telegram | Sends the HTML report file as a Telegram document |

---

## Agent Hierarchy

```
Level 1 — Supervisor Agent
  └── Level 2 — new subscription      (AgentTool)
  └── Level 2 — cancel subscription   (AgentTool)
  └── Level 2 — analysis              (AgentTool)
        └── Level 3 — html report     (AgentTool nested inside analysis)
```

Each agent at every level has its own:
- **LLM instance** (separate Google Vertex / Gemini model node)
- **Session memory buffer** (isolated per agent, keyed by chat ID)
- **Tool set** (only the tools relevant to that agent's scope)

---

## Google Sheets Schema

**Sheet name:** `Sheet1` | **Document:** GYM Subscriptions Management

| Column | Type | Description |
|---|---|---|
| ID | String | Auto-incremented integer (`last_id + 1`) |
| Full Name | String | Subscriber's full three-part name (match key) |
| Phone Number | String | Digits only, stripped of non-digit characters |
| Duration(month) | String | Subscription duration in months |
| Start Date | String | Auto-computed start date (`YYYY-MM-DD`) |
| End Date | String | Auto-computed end date (`YYYY-MM-DD`) |
| Amount Paid | String | Amount paid in Jordanian Dinars |
| Status | String | `نشط` (active) or `ملغي` (cancelled) |

---

## Available Commands

| Command | Sub-Agent | Description |
|---|---|---|
| `/new_subscription` | new subscription | Starts a multi-turn flow to register a new subscriber |
| `/cancel_subscription` | cancel subscription | Cancels an existing subscription by full name |
| `/analysis` | analysis | Generates an HTML report or answers gym-related questions |
| _(any gym-related text)_ | analysis | Conversational Q&A about subscriptions and members |

---

## Tech Stack

| Component | Technology |
|---|---|
| Messaging | Telegram Bot API |
| AI Orchestration | n8n LangChain Agent / AgentTool |
| Language Models | Google Vertex AI (Gemini) — 4 separate instances |
| Session State | n8n Memory Buffer Window — 4 isolated buffers |
| Date Computation | Custom JavaScript Tool Code node |
| Data Storage | Google Sheets API (read / append / update) |
| Report Generation | Gemini 2.5 Pro (nested HTML report agent) |

---

## Prerequisites

- n8n instance v1.0+
- **Telegram Bot Token** — required for Telegram Trigger and send nodes
- **Google Service Account** — with Vertex AI API enabled (for all Gemini model nodes)
- **Google Sheets OAuth2** — for all Google Sheets tool nodes
- A **Google Cloud project** with Vertex AI API enabled (project ID: configure in each `lmChatGoogleVertex` node)

---

## Configuration

### 1. Authentication
- Open the `Authentication` IF node.
- Replace the hardcoded `rightValue` (chat ID) with your own Telegram chat ID.
- To find your chat ID: message `@userinfobot` on Telegram.

### 2. Google Credentials
- In n8n Settings → Credentials, add a **Google Service Account** credential.
- Link it to all four `lmChatGoogleVertex` nodes (`Google Vertex`, `Google Vertex1`, `Google Vertex2`, `Google Vertex3`, `gemini-2.5-pro`).
- Update the `projectId` field in each model node to your own Google Cloud project ID.

### 3. Google Sheets Credentials
- Add a **Google Sheets OAuth2** credential in n8n.
- Link it to all Google Sheets tool nodes: `get user`, `get rows`, `Append row`, `update`.
- Ensure the target spreadsheet (`GYM Subscriptions Management`) is shared with the service account email.

### 4. Telegram Credentials
- Add a **Telegram API** credential (bot token) in n8n.
- Link it to `Telegram Trigger`, `Response to user`, `Send HTML report`, and `Send a text message`.

---

## Output

| Trigger | Output |
|---|---|
| `/new_subscription` completed | Arabic confirmation message with full subscriber details |
| `/cancel_subscription` completed | Arabic confirmation with subscriber name and new status |
| `/analysis` (text query) | Arabic conversational answer about the subscription database |
| `/analysis` (report request) | HTML document sent as a Telegram file attachment |
| Unauthorized access attempt | Arabic rejection message: _"وصول غير مصرح به"_ |
| Unknown command | Arabic error listing all available commands |

---

## Edge Cases & Fallbacks

| Scenario | Behavior |
|---|---|
| Unauthorized chat ID | Immediately rejects with Arabic message — no further processing |
| Empty or null message text | Returns: _"لم أتمكن من قراءة رسالتك. الرجاء المحاولة مرة أخرى."_ |
| Name not exactly 3 words | Returns validation error and re-prompts — state preserved |
| Phone number fewer than 5 digits | Returns validation error and re-prompts — state preserved |
| Invalid duration (non-integer or ≤ 0) | Returns validation error and re-prompts — state preserved |
| Duplicate active subscriber detected | Returns full subscriber details with warning — registration blocked |
| `get dates` tool failure | Returns error message, resets state to `AWAITING_DURATION` |
| `Append row` failure | Returns error message, resets state to `AWAITING_NAME` |
| Subscriber not found during cancellation | Returns not-found message and prompts for re-entry |
| `update` tool failure | Returns error message, resets state to `AWAITING_CANCEL_NAME` |
| Unknown `/command` | Returns Arabic error with list of all valid commands — active route unchanged |
| Sub-agent returns empty or error | Supervisor returns: _"حدث خطأ أثناء تشغيل الأمر. الرجاء المحاولة مرة أخرى لاحقاً."_ |
