# 🥈 Silver Market Intelligence Brief
Sends a daily silver market brief to Telegram every weekday at 8AM (Amman time) by fetching silver prices and relevant news. It triggers an alert if the price changes significantly, helping users stay informed effectively.

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
This workflow is triggered daily at 8 AM, fetching silver market data and news. It combines multiple information sources and reports results via Telegram, delivering timely updates to users.

---

## How It Works
[Connection data unavailable — diagram cannot be generated]
1. The workflow starts at 8 AM every weekday.
2. It fetches the current silver price from Yahoo Finance.
3. It retrieves silver market news from Yahoo Finance and Investing.com RSS feeds.
4. The data is merged and processed to build a market analysis prompt.
5. An AI agent analyzes the market data and generates a report.
6. If the silver price changes significantly, a separate alert is sent.
7. Finally, a daily report is sent to Telegram.

---

## Node Architecture
| Node Name                           | Type                                               | Role                                      |
|-------------------------------------|----------------------------------------------------|-------------------------------------------|
| Daily 8AM Mon-Fri1                 | n8n-nodes-base.scheduleTrigger                    | Trigger for executing workflow daily     |
| Get Silver Price1                  | n8n-nodes-base.httpRequest                        | Fetch current silver price                |
| Yahoo Finance Silver RSS            | n8n-nodes-base.httpRequest                        | Fetch silver market news from Yahoo      |
| Investing.com Commodities RSS       | n8n-nodes-base.httpRequest                        | Fetch commodities news from Investing.com |
| Merge                               | n8n-nodes-base.merge                              | Merge input data from multiple sources    |
| Build AI Prompt                     | n8n-nodes-base.code                               | Create AI prompt based on merged data    |
| OpenAI GPT-4o1                     | @n8n/n8n-nodes-langchain.lmChatOpenAi            | Process AI language model for analysis    |
| Basic LLM Chain                     | @n8n/n8n-nodes-langchain.chainLlm                 | Handle structured AI language tasks       |
| If                                  | n8n-nodes-base.if                                 | Conditional branching for alerts          |
| alert                                | n8n-nodes-base.telegram                           | Send alerts via Telegram                  |
| Send Daily Report                   | n8n-nodes-base.telegram                           | Send daily market report to Telegram      |
| Sticky Note                         | n8n-nodes-base.stickyNote                         | Description note for workflow context      |

---

## Tech Stack
| Component                          | Technology                                      |
|------------------------------------|--------------------------------------------------|
| Trigger                             | n8n-nodes-base.scheduleTrigger                   |
| Data Fetching                      | n8n-nodes-base.httpRequest                       |
| Data Merging                       | n8n-nodes-base.merge                             |
| AI Processing                      | @n8n/n8n-nodes-langchain.lmChatOpenAi          |
| Reporting                          | n8n-nodes-base.telegram                          |

---

## Prerequisites
- n8n instance v1.0+
- OpenAI API key required
- Telegram bot token required

---

## Configuration
Users need to update their OpenAI API key and Telegram bot token. Additionally, replace the chat ID in Telegram nodes with the user’s own chat ID.

---

## Output
The workflow generates a daily summary of the silver market with price details, changes, and relevant news sent to the designated Telegram chat.

---

## Edge Cases & Fallbacks
| Scenario                                            | Behavior                                           |
|----------------------------------------------------|---------------------------------------------------|
| Missing required node fields                        | Workflow will not execute correctly.               |
| Error fetching data from external APIs              | Workflow returns an error message.                |
| Data processing issues                               | Output may indicate missing or unavailable data.   |