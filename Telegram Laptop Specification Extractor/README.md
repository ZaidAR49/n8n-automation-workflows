# 📊 Telegram Laptop Specification Extractor
This workflow automates the extraction of laptop specifications from user-submitted data via Telegram, processing images or product URLs through various nodes and ultimately storing the extracted data in a Google Sheet.

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
The workflow is triggered by messages in Telegram that either contain images of laptops or Amazon product URLs. It utilizes various AI and Google Sheets integrations to extract, analyze, and store laptop specifications based on user inputs.

---

## How It Works
```
Telegram Trigger → Switch → URL Extractor → URL AI Agent
Switch → Download from Telegram → Image AI Agent
Image AI Agent → Append to Sheet-image → Send Success
URL AI Agent → Append to Sheet-url → Send Success
Comparison Agent → Send Comparison Result
```
1. The process begins with a `Telegram Trigger` that captures user inputs.
2. Based on the input type, a `Switch` node directs the flow to either the `URL Extractor` for URLs or the `Download from Telegram` for images.
3. The image analysis occurs in the `Image AI Agent`, while URLs are handled by the `URL AI Agent` for extraction and validation.
4. Extracted data is then appended to the Google Sheet, followed by a success message sent back through Telegram.
5. The `Comparison Agent` analyzes the specifications and sends back results on the best laptop based on user-defined criteria.

---

## Node Architecture
| Node Name                    | Type                                      | Role                                                   |
|------------------------------|-------------------------------------------|--------------------------------------------------------|
| Telegram Trigger             | n8n-nodes-base.telegramTrigger           | Trigger the workflow based on Telegram messages.       |
| Switch                       | n8n-nodes-base.switch                    | Route workflow based on message type.                  |
| URL Extractor                | n8n-nodes-base.code                      | Extract URL data from Telegram messages.               |
| URL AI Agent                 | @n8n/n8n-nodes-langchain.agent           | Extract and analyze data from Amazon URLs.             |
| Download from Telegram        | n8n-nodes-base.telegram                  | Download images from Telegram messages.                |
| Image AI Agent               | @n8n/n8n-nodes-langchain.agent           | Analyze images for laptop specifications.               |
| Append to Sheet-image        | n8n-nodes-base.googleSheetsTool          | Insert image-derived specifications into Google Sheets. |
| Append to Sheet-url          | n8n-nodes-base.googleSheetsTool          | Insert URL-derived specifications into Google Sheets.   |
| Comparison Agent             | @n8n/n8n-nodes-langchain.agent           | Compare laptops and send back the best option.         |
| Send Success                 | n8n-nodes-base.telegram                  | Notify successful data processing via Telegram.        |
| Send Comparison Result        | n8n-nodes-base.telegram                  | Send the best laptop comparison result to the user.    |
| Send Error                   | n8n-nodes-base.telegram                  | Notify users of errors during processing.               |

---

## Tech Stack
| Component    | Technology     |
|--------------|----------------|
| Trigger      | n8n            |
| Messaging    | Telegram API   |
| Data Storage | Google Sheets  |
| AI           | LangChain      |
| Code         | n8n            |

---

## Prerequisites
- n8n instance v1.0+
- Telegram account API key required
- Google Sheets account API key required
- [Any credential inferred from node `credentials` keys]

---

## Configuration
Users need to authenticate with valid Telegram and Google Sheets credentials within the n8n instance prior to running this workflow.

---

## Output
The output of the workflow includes successfully extracted laptop specifications which are stored in a designated Google Sheet. Users are notified via Telegram about each successful insertion.

---

## Edge Cases & Fallbacks
| Scenario                                 | Behavior                                                |
|------------------------------------------|--------------------------------------------------------|
| Missing required fields in Telegram      | Send error message about input requirement.            |
| Malformed URL or unreadable image        | Notify user of processing failure.                      |
| Google Sheets tool failure                | Log the error and notify user.                         |
| No laptops found in the tracker          | Inform user to add laptops before making comparisons.  |