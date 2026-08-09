# 📦 Workflow to GDrive Extractor

An n8n automation workflow that accepts a **workflow `.json` file via Telegram**, extracts all embedded code snippets and AI system prompts from it, uploads every extracted file to **Google Drive**, makes them publicly accessible, and sends back the shareable links — all from a single Telegram message.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Diagram](#workflow-diagram)
- [Prerequisites](#prerequisites)
- [Setup & Configuration](#setup--configuration)
- [How to Use](#how-to-use)
- [Node Reference](#node-reference)
- [Extraction Logic](#extraction-logic)
- [Output Format](#output-format)
- [Edge Cases & Error Handling](#edge-cases--error-handling)
- [Tips & Troubleshooting](#tips--troubleshooting)

---

## Overview

| Property         | Details                                               |
|------------------|-------------------------------------------------------|
| **Trigger**      | Telegram Bot — document (`.json` file) message        |
| **Auth**         | Allowlist by Telegram Chat ID                         |
| **Processing**   | JavaScript Code node (custom extractor)               |
| **Storage**      | Google Drive (OAuth2)                                 |
| **Output**       | Telegram rich-text message with public file links     |

**Use cases:**
- Instantly extract and review the code & prompts embedded inside any n8n workflow export
- Archive workflow assets to Google Drive for sharing or version control
- Audit AI system prompts from workflows without opening n8n

---

## Workflow Diagram

```
Telegram Trigger (receives .json file)
      │
      ▼
Check Authorized User (IF — Chat ID allowlist)
  ├── ❌ Unauthorized ──► Send Unauthorized Alert (Telegram)
  │
  └── ✅ Authorized ──► Get a file (Telegram — download binary)
                              │
                              ▼
                        Extract from File (JSON → object)
                              │
                        ┌─────┴─────┐
                        ▼           ▼
                  Create folder   Extract Prompts & Code (Code node)
                  (Google Drive)        │
                        │         Split Binary Files
                        │               │
                        └───────► Upload file (Google Drive)
                                        │
                                        ▼
                                  Make File Public (Google Drive)
                                        │
                                        ▼
                                  Send File Links (Telegram)
```

---

## Prerequisites

Before importing and activating this workflow, you need:

1. **n8n** instance (self-hosted or cloud)
2. **Telegram Bot** — Create one via [@BotFather](https://t.me/BotFather)
3. **Google Account** with Google Drive enabled — [drive.google.com](https://drive.google.com)
   - A pre-existing target folder in your Drive (the parent folder where per-workflow subfolders will be created)
4. Your **Telegram Chat ID** — the numeric ID of the user(s) authorized to use the bot

---

## Setup & Configuration

### 1. Import the Workflow

In your n8n instance:
1. Go to **Workflows** → **Import**
2. Upload `workflow.json`

### 2. Configure Credentials

Set up the following credentials in n8n (**Settings → Credentials**):

| Credential Name         | Type                   | Where Used                          |
|-------------------------|------------------------|-------------------------------------|
| `Telegram account 2`    | Telegram API           | Telegram Trigger, Get a file, Send nodes |
| `Google Drive account`  | Google Drive (OAuth2)  | Create folder, Upload file, Make File Public |

### 3. Set the Authorized User

In the **Check Authorized User** node, update the `rightValue` of the condition to your own Telegram Chat ID (currently hardcoded as `2076665816`).

> **Tip:** Send a message to [@userinfobot](https://t.me/userinfobot) on Telegram to get your Chat ID.

### 4. Set the Target Google Drive Folder

In the **Create folder** node, update the `folderId` field with the ID of the Google Drive folder where per-workflow subfolders should be created.

> **Tip:** Open the target folder in Google Drive. The folder ID is the last segment of the URL:
> `https://drive.google.com/drive/folders/`**`<FOLDER_ID>`**

### 5. Activate the Workflow

Toggle the workflow to **Active** in n8n. The Telegram webhook will begin listening for incoming documents.

---

## How to Use

1. Open your Telegram bot
2. Attach and send any n8n **workflow `.json` file** (exported from n8n) as a document message
3. The bot will automatically:
   - Download the file from Telegram
   - Parse the workflow JSON
   - Extract all code nodes and AI system prompts
   - Create a named subfolder in your Google Drive
   - Upload every extracted file (`.js`, `.py`, `.md`, `.json`) to that folder
   - Make the folder publicly readable
   - Reply with a clickable link for each uploaded file

If you are not in the authorized user list, the bot replies:
`غير مصرح لك باستعمال هذا البوت` *(You are not authorized to use this bot)*

---

## Node Reference

### 1. Telegram Trigger
- **Type:** `n8n-nodes-base.telegramTrigger`
- **Listens for:** Any incoming `message` update (including document attachments)
- **Outputs:** Raw Telegram message object containing `message.document.file_id`

### 2. Check Authorized User
- **Type:** `n8n-nodes-base.if`
- **Condition:** `message.chat.id` equals the hardcoded authorized Chat ID
- **Branch True (output 1):** Proceeds to file download
- **Branch False (output 2):** Sends an unauthorized alert and stops

### 3. Send Unauthorized Alert
- **Type:** `n8n-nodes-base.telegram`
- **Triggered when:** The sender is not in the authorized allowlist
- **Action:** Replies to the user's chat with an Arabic-language rejection message

### 4. Get a file
- **Type:** `n8n-nodes-base.telegram` (resource: `file`)
- **Input:** `message.document.file_id` from the Telegram message
- **Output:** Binary file data of the uploaded `.json`

### 5. Extract from File
- **Type:** `n8n-nodes-base.extractFromFile` (operation: `fromJson`)
- **Input:** Binary data from the previous node
- **Output:** Parsed JavaScript object — the full n8n workflow structure

### 6. Create folder
- **Type:** `n8n-nodes-base.googleDrive` (resource: `folder`)
- **Folder name:** `{{ $json.data.name }}` — uses the workflow's own name
- **Parent:** The configured target folder in your Google Drive
- **Output:** The new folder's `id`, referenced by all subsequent upload steps

### 7. Extract Prompts & Code
- **Type:** `n8n-nodes-base.code` (JavaScript, run-once mode)
- **Input:** Reads all items from the **Extract from File** node
- **Output:** One binary item per extracted file — see [Extraction Logic](#extraction-logic) below

### 8. Split Binary Files
- **Type:** `n8n-nodes-base.splitOut`
- **Purpose:** Splits the array of binary file items so each file flows through the pipeline independently as a separate execution item

### 9. Upload file
- **Type:** `n8n-nodes-base.googleDrive`
- **Drive:** My Drive
- **Folder:** The folder created in step 6 (`Create folder` node ID)
- **File name:** Taken directly from each binary item's `fileName` metadata

### 10. Make File Public
- **Type:** `n8n-nodes-base.googleDrive` (operation: `share`)
- **Permission:** `reader` / `anyone` — makes the file publicly viewable without sign-in
- **Input:** The parent folder ID from the uploaded file's `parents[0]` field

### 11. Send File Links
- **Type:** `n8n-nodes-base.telegram` (operation: `sendRichMessage`)
- **Format:** HTML — sends the file name in bold and its Google Drive `webViewLink` in a code block for easy copying

---

## Extraction Logic

The **Extract Prompts & Code** node contains a custom JavaScript extractor that recursively processes every node in the workflow:

### What gets extracted

| Content Type        | Source Field(s)                                                        | Output Extension |
|---------------------|------------------------------------------------------------------------|-----------------|
| JavaScript code     | `jsCode`, `functionCode`, `code`                                       | `.js`           |
| Python code         | `pythonCode`, `code` (with `language: python`)                         | `.py`           |
| AI system prompts   | `systemMessage`, `systemPrompt`, `system`, or any `role: "system"` entry in arrays | `.md` |
| Workflow JSON       | The entire workflow object itself                                       | `.json`         |

### Naming rules

- **Filename = node name** as it appears on the canvas (special characters replaced with `_`)
- **Collision avoidance:** If two nodes share the same name, the second file is suffixed `(2)`, the third `(3)`, etc.
- The full workflow JSON is always included as `<workflow name>.json`

### Items emitted

Each output item has:
- `json: {}` — intentionally empty; the item carries no JSON data
- `binary.data` — the extracted file with its filename, MIME type, and content embedded

---

## Output Format

For each uploaded file, the bot sends a Telegram message in this format:

```
<b>filename.js</b>
<code>https://drive.google.com/file/d/.../view?usp=drivesdk</code>
```

- The filename is **bold** for quick scanning
- The link is in a `<code>` block so it can be tapped/copied easily on mobile
- One message is sent per file

---

## Edge Cases & Error Handling

| Situation | Behavior |
|-----------|----------|
| Sender's Chat ID is not in the allowlist | Bot replies with an unauthorized message; workflow stops |
| Workflow JSON has no Code nodes or AI nodes | Only the raw `.json` file of the workflow itself is uploaded |
| Two nodes have the same canvas name | Both files are uploaded; the second is renamed with a `(2)` suffix |
| A Code node's `jsCode`/`pythonCode` field is empty | That node is skipped; no empty file is created |
| A system prompt field is duplicated across parameters | `Set` deduplication ensures only unique prompts create files |
| Google Drive folder creation fails | Upload node will fail — check Drive credentials and parent folder ID |
| Telegram file download fails | Workflow stops at **Get a file** — verify the bot token has file-download permissions |

---

## Tips & Troubleshooting

**📁 Files not appearing in the right folder?**
- Double-check the `folderId` in the **Create folder** node matches your intended parent folder
- Confirm the Google Drive credential has write access to that folder

**🔒 "Unauthorized" message even for yourself?**
- Find your exact numeric Chat ID using [@userinfobot](https://t.me/userinfobot)
- Update the `rightValue` in the **Check Authorized User** node to match that number exactly

**📎 Bot not responding to file uploads?**
- Ensure the file is sent as a **Document** (not as a photo or compressed file)
- The Telegram Trigger must be set to listen for `message` updates

**🔗 Public link not working?**
- Confirm the **Make File Public** node is using `role: reader` and `type: anyone`
- Sharing permissions may take a few seconds to propagate

**⚙️ Adding more authorized users?**
- Replace the single **IF** condition with a **Switch** node or add multiple `OR` conditions in the **Check Authorized User** node
