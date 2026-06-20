# 🔍 Scam Detection Workflow
This workflow automatically analyzes links received via Telegram messages to determine their legitimacy and detect potential scams.

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
The workflow is triggered by incoming messages on Telegram. It processes the message text to extract URLs and then analyzes these URLs for scams and fraud indicators, delivering insights directly back to the Telegram chat.

---

## How It Works
[Connection data unavailable — diagram cannot be generated]
1. The workflow initiates with a Telegram Trigger that captures incoming messages.
2. A JavaScript function extracts the first URL in the message text.
3. The workflow sends this URL to various agents that conduct different analyses: domain checks, search engine signals, product and pricing patterns, and content analysis.
4. Results from these analyses are aggregated and evaluated to form a final verdict.
5. The conclusion is sent back as a Telegram message to the user.

---

## Node Architecture
| Node Name                     | Type                                             | Role                                      |
|-------------------------------|--------------------------------------------------|-------------------------------------------|
| Telegram Trigger              | n8n-nodes-base.telegramTrigger                  | Captures Telegram messages                |
| Code in JavaScript            | n8n-nodes-base.code                            | Extracts URLs and classifies them         |
| Merge1                        | n8n-nodes-base.merge                           | Aggregates results from analyses          |
| Aggregate1                    | n8n-nodes-base.aggregate                        | Combines outputs from various analyses    |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Analyzes URLs with OpenAI model          |
| SerpAPI                       | @n8n/n8n-nodes-langchain.toolSerpApi           | Fetches data for domain analysis          |
| Search Engine Signals         | @n8n/n8n-nodes-langchain.agent                  | Checks search engine results for scams    |
| Product & Pricing Patterns    | @n8n/n8n-nodes-langchain.agent                  | Evaluates pricing patterns                |
| Content Analysis              | @n8n/n8n-nodes-langchain.agent                  | Analyzes website content                  |
| Evaluator                     | @n8n/n8n-nodes-langchain.agent                  | Produces final verdict                    |
| Text analysis                 | @n8n/n8n-nodes-langchain.agent                  | Analyzes promotional text                  |
| Send a text message           | n8n-nodes-base.telegram                        | Sends the analysis result back to Telegram |

---

## Tech Stack
| Component                     | Technology                                      |
|-------------------------------|--------------------------------------------------|
| Telegram Integration           | n8n-nodes-base.telegramTrigger                  |
| JavaScript Processing          | n8n-nodes-base.code                             |
| Data Aggregation              | n8n-nodes-base.merge                            |
| Data Analysis                 | n8n-nodes-base.aggregate                         |
| OpenAI Integration            | @n8n/n8n-nodes-langchain.lmChatOpenAi          |
| SERP API Integration          | @n8n/n8n-nodes-langchain.toolSerpApi           |

---

## Prerequisites
- n8n instance v1.0+
- Telegram API key required
- OpenAI API key required
- SerpAPI account required

---

## Configuration
_[Insufficient data — review manually]_ 

---

## Output
The workflow produces an analysis message detailing the legitimacy of the URL assessed and is sent back to the originating Telegram chat.

---

## Edge Cases & Fallbacks
| Scenario                         | Behavior                                        |
|----------------------------------|------------------------------------------------|
| Missing required input fields    | Error message will indicate which field is missing |
| Malformed workflow JSON input    | Error message will indicate input format issue       |