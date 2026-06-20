# 🤖 Chat Message Handler
This workflow listens for chat messages and responds with information regarding the Family and Childhood Protection Society, utilizing an AI agent to provide accurate answers based on the user's queries. It aims to facilitate user interaction and streamline information retrieval about the organization.

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
The workflow is triggered when a chat message is received. It then processes the message through an AI agent that fetches relevant information, enhancing user experience and ensuring prompt responses.

---

## How It Works
[Connection data unavailable — diagram cannot be generated]
Step 1 → Step 2 → Step 3 ...

1. When a chat message is received, the workflow triggers the AI agent.
2. The AI agent processes the chat input using predefined rules to fetch information about the Family and Childhood Protection Society.
3. Responses or any errors are sent back through the email notification system to the administrators.

---

## Node Architecture
| Node Name                     | Type                                         | Role                                          |
|-------------------------------|----------------------------------------------|-----------------------------------------------|
| When chat message received     | @n8n/n8n-nodes-langchain.chatTrigger        | Triggers workflow on receiving chat messages  |
| AI Agent                      | @n8n/n8n-nodes-langchain.agent              | Processes input and manages responses          |
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains context of conversation               |
| organization data             | n8n-nodes-base.supabaseTool                 | Fetches organization details                    |
| If                            | n8n-nodes-base.if                           | Handles conditional logic in workflow           |
| Send a message                | n8n-nodes-base.gmail                        | Sends email notifications                       |
| get admin email               | n8n-nodes-base.supabase                     | Retrieves admin email addresses                 |
| If1                           | n8n-nodes-base.if                           | Conditional forwarding based on admin email    |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Utilizes OpenAI for chat management            |
| Error Trigger                 | n8n-nodes-base.errorTrigger                 | Triggers on workflow errors                     |
| Send a message1               | n8n-nodes-base.gmail                        | Sends error notifications                       |
| Sticky Note                   | n8n-nodes-base.stickyNote                   | Provides notes or explanations                  |

---

## Tech Stack
| Component          | Technology                       |
|--------------------|----------------------------------|
| Workflow Trigger    | n8n-nodes-base.chatTrigger      |
| AI Processing       | @n8n/n8n-nodes-langchain.agent   |
| State Management    | @n8n/n8n-nodes-langchain.memoryBufferWindow |
| Data Fetching       | n8n-nodes-base.supabaseTool      |
| Conditional Logic   | n8n-nodes-base.if                |
| Email Notification  | n8n-nodes-base.gmail             |
| Admin Management    | n8n-nodes-base.supabase          |
| Workflow Error Handling | n8n-nodes-base.errorTrigger   |
| OpenAI Integration  | @n8n/n8n-nodes-langchain.lmChatOpenAi |

---

## Prerequisites
- n8n instance v1.0+
- Gmail OAuth2 API key required
- Supabase account required
- OpenAI API key required

---

## Configuration
Make sure to set the required credentials for Supabase, Gmail API, and OpenAI in your n8n instance settings.

---

## Output
The workflow produces responses to chat inquiries regarding the Family and Childhood Protection Society based on user input, alongside notifications to administrators in case of errors.

---

## Edge Cases & Fallbacks
| Scenario                                 | Behavior                                                              |
|------------------------------------------|-----------------------------------------------------------------------|
| Missing required fields in chat message  | The AI will prompt the user for clarification on required information. |
| Malformed workflow JSON input            | Error message will be returned: Workflow cannot be processed.        |
| Failure to fetch organization data       | User will receive a generic error response indicating data unavailability. |
| Email sending failure                     | Administrators will be alerted about email notification failure.       |