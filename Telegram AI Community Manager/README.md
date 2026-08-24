# 🤖 Telegram AI Community Manager

An intelligent n8n automation workflow that acts as a fully autonomous community manager for Telegram groups. It handles member moderation, AI-powered Q&A, and automated new member onboarding — all in one workflow.

---

## ✨ Features

| Feature | Description |
|---|---|
| 🛡️ **AI Moderation** | Automatically detects and removes inappropriate messages reported by members |
| 🧠 **AI Q&A Assistant** | Answers member questions via bot mention using Google Gemini |
| 👋 **Welcome Messages** | Greets new human members with instructions on how to use the bot |
| 🌐 **Bilingual Support** | Supports both Arabic (RTL) and English responses |
| 🤖 **Bot Filtering** | Skips welcome messages for bot accounts joining the group |
| 📢 **Transparent Moderation** | Notifies the group publicly when a message is deleted and warns the offender privately |

---

## 🔄 Workflow Overview

```
Telegram Trigger
    └── Is group message? (filter)
            └── Extract Message Context (JS)
                    └── Set User Preferences
                            └── Route by Action Type
                                    ├── [Report]     → Text Classifier (AI)
                                    │                       ├── [Inappropriate] → Delete Message → Public Notice → Warn Offender (DM)
                                    │                       └── [Safe]          → Notify Reporter (DM)
                                    │
                                    ├── [Question]   → AI Agent (Gemini) → Send AI Response
                                    │
                                    └── [New Member] → Is Bot? → Send Welcome Message
```

---

## 🧩 Workflow Nodes

### 1. `Telegram Trigger`
- Listens for incoming **messages** in the configured Telegram group.
- Entry point for all events.

### 2. `Is group message`
- Filters out non-group messages (currently configured for a group named **"Test Group"**).
- Prevents the bot from processing direct messages or unrelated chats.

### 3. `Extract Message Context` *(JavaScript Code)*
- Core parsing node. Extracts all relevant data from the Telegram message payload:
  - **Reply/Quote detection** — captures the text of the message being replied to
  - **Mention & command detection** — identifies if the bot is mentioned (`@zaidinfo_bot`) and if `/report` command is used
  - **Report detection** — `isReport = isBotMentioned AND isReply AND /report`
  - **Question detection** — `isQuestion = isBotMentioned AND NOT isReport`
  - **New member detection** — checks for `new_chat_members` events
  - **Offender & reporter info** — captures names, IDs, and usernames

### 4. `Set User Preferences`
- Sets global configuration variables used throughout the workflow:
  - `lan` — Language code (`ar` for Arabic, or `en` for English)
  - `admin_username` — Admin's Telegram handle for contact in warnings

### 5. `Route by Action Type` *(Switch)*
- Routes the flow into one of three branches based on the message context:
  - **Report** → moderation branch
  - **Question** → AI Q&A branch
  - **New Member** → onboarding branch

---

### 🛡️ Moderation Branch

#### `Text Classifier`
- Uses **Google Gemini** to classify the reported message as either:
  - `Inappropriate` — bad words, hate speech, spam, scams, political arguments, gambling/drugs/weapons
  - `Safe` — normal conversation or a false report

#### `Delete Inappropriate Message`
- Calls the Telegram API to **delete** the offending message from the group.
- Configured to continue on error (e.g., if the message was already deleted).

#### `Send Public Deletion Notice`
- Posts a **public message in the group** announcing the deletion, tagging the offender by name.

#### `Send Warning to Offender`
- Sends a **private DM** to the offender explaining:
  - Which message was removed (quoted)
  - Why it was removed (community guidelines)
  - How to appeal (contact the admin)
- Fully bilingual (Arabic/English).

#### `Notify Reporter (Safe)`
- If the reported message is classified as **Safe**, sends a **private DM to the reporter** thanking them but clarifying the message didn't violate rules.

---

### 🧠 AI Q&A Branch

#### `AI Agent`
- A LangChain AI Agent powered by **Google Gemini** (`gemini-3.7-flash`).
- Receives the cleaned question text (bot mention stripped out).
- Uses **Simple Memory** (window buffer) for per-chat conversation history.

#### `Simple Memory`
- Window buffer memory keyed by `chatId` — maintains short-term conversation context per group.

#### `Send AI Response`
- Sends the AI Agent's output back to the group chat as a Markdown-formatted message.

---

### 👋 New Member Onboarding Branch

#### `Is bot?`
- Checks if the new member joining is a **bot** (`is_bot` flag).
- Only proceeds if the new member is a **human**.

#### `Send Welcome Message`
- Sends a friendly, formatted welcome message to the group, tagging the new member.
- Includes instructions on:
  - How to ask the AI Assistant (`@zaidinfo_bot <question>`)
  - How to report inappropriate messages (`/report` as a reply)
- Bilingual: Arabic or English based on `lan` preference.

---

## ⚙️ Configuration

Before activating the workflow, update the following settings:

### 🔧 `Is group message` Node
Change the group name filter to match **your** Telegram group title:
```
"Test Group"  →  "Your Group Name"
```

### 🔧 `Extract Message Context` Node
Update the bot username constant:
```js
const BOT_USERNAME = 'zaidinfo_bot'.toLowerCase();
// Change to your bot's username
```

### 🔧 `Set User Preferences` Node
| Variable | Default | Description |
|---|---|---|
| `lan` | `ar` | Response language — `ar` (Arabic) or `en` (English) |
| `admin_username` | `@zaidrad49` | Telegram handle shown in warning messages |

### 🔧 `Send Welcome Message` & `Send Warning to Offender` Nodes
Update the hardcoded bot mentions and admin contact inside the message templates to reflect your bot and admin usernames.

---

## 🔑 Required Credentials

| Credential | Node(s) | Description |
|---|---|---|
| **Telegram Bot API** | All Telegram nodes | API token from [@BotFather](https://t.me/BotFather) |
| **Google Service Account** | Google Vertex Chat Model | GCP service account with Vertex AI access |

> **Tip:** Make sure your Telegram bot is added to the group as an **administrator** with permission to delete messages.

---

## 🚀 Setup Instructions

1. **Import** `workflow.json` into your n8n instance.
2. **Configure credentials:**
   - Add your Telegram Bot API token.
   - Add your Google Cloud Service Account (with Vertex AI / Gemini API enabled).
3. **Update configuration** in the nodes listed in the [Configuration](#️-configuration) section above.
4. **Add the bot to your Telegram group** and grant it admin rights (delete messages permission).
5. **Activate** the workflow.

---

## 🤖 Bot Commands & Usage

| Action | How to Trigger |
|---|---|
| Ask the AI a question | Mention the bot: `@your_bot_name What is n8n?` |
| Report a message | Reply to the message with: `@your_bot_name /report` |
| (Auto) Welcome new member | Triggered automatically when someone joins |

---

## 🌐 Language Support

The workflow supports **bilingual** output. Set `lan` in the `Set User Preferences` node:

- `ar` → All bot messages sent in **Arabic** (RTL enabled)
- `en` → All bot messages sent in **English**

---

## 🛠️ Tech Stack

- **n8n** — Workflow automation platform
- **Telegram Bot API** — Group message handling & actions
- **Google Gemini** (`gemini-3.7-flash`) via **Vertex AI** — Text classification & AI Q&A
- **LangChain (n8n nodes)** — AI Agent & memory management
- **JavaScript** — Custom message parsing logic

---

## 📂 File Structure

```
Telegram AI Community Manager/
├── workflow.json   # n8n workflow definition
└── README.md       # This file
```

---

## ⚠️ Notes & Limitations

- The **group filter** (`Is group message`) is currently hardcoded to `"Test Group"`. Update it for production use.
- The workflow processes **one message event at a time** via webhook.
- **Bot-to-bot messages** are ignored in the new member onboarding branch.
- Memory is **session-scoped per chat** — it resets when n8n restarts unless an external memory store is configured.
- Ensure the bot has **admin privileges** in the group to delete messages; otherwise, the delete step will silently fail and continue.
