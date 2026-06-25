# Arabic Story Storytelling Pipeline

An automated n8n workflow that generates a complete Arabic short story every day, illustrates each scene with AI-generated images, and delivers the full story to a Telegram channel — hands-free.

---

## Overview

Every evening at 9:00 PM, the pipeline wakes up, invents a brand-new 5-scene Arabic story, generates a matching illustration for each scene using Google's Gemini image model, and posts everything to a Telegram group — one scene at a time, with a 10-second pause between scenes to create a storytelling rhythm.

---

## Pipeline Flow

```
Schedule Trigger (9 PM)
    │
    ▼
AI Agent  ←─── Google Vertex Chat Model (Gemini)
    │           └─── Structured Output Parser
    │
    ▼
Split Out (scenes array → individual items)
    │
    ▼
Loop Over Items ──────────────────────────────────────┐
    │                                                  │
    ▼                                                  │
Vertex AI HTTP Request (Gemini image generation)       │
    │                                                  │
    ▼                                                  │
Edit Fields (extract base64 image)                     │
    │                                                  │
    ▼                                                  │
Convert to File                                        │
    │                                                  │
    ▼                                                  │
Send Photo → Telegram                                  │
    │                                                  │
    ▼                                                  │
Send Text → Telegram                                   │
    │                                                  │
    ▼                                                  │
Wait (10 seconds)                                      │
    │                                                  │
    ▼                                                  │
If (more scenes?) ── Yes ──────────────────────────────┘
    │
    No
    ▼
Send "النهاية" end message → Telegram
```

---

## Nodes

| Node | Type | Purpose |
|---|---|---|
| **Schedule Trigger** | Trigger | Fires daily at 9:00 PM |
| **AI Agent** | LangChain Agent | Generates the full 5-scene story |
| **Google Vertex Chat Model** | LLM | Powers the AI Agent (Gemini via Vertex AI) |
| **Structured Output Parser** | Output Parser | Enforces JSON schema on the agent's output |
| **Split Out** | Data | Splits `scenes[]` array into individual scene items |
| **Loop Over Items** | Control Flow | Iterates through each of the 5 scenes one by one |
| **Vertex AI HTTP Request** | HTTP | Calls Gemini image generation API per scene |
| **Edit Fields** | Data | Extracts the base64-encoded image from the API response |
| **Convert to File** | Data | Converts base64 string to a binary image file |
| **Send a photo message** | Telegram | Posts the scene illustration to the Telegram channel |
| **Send a text message** | Telegram | Posts the Arabic scene text to the Telegram channel |
| **Wait** | Control Flow | Pauses 10 seconds between scenes |
| **If** | Control Flow | Checks if all scenes have been sent |
| **the end message** | Telegram | Posts a `-------النهاية-------` closing message |

---

## Story Generation

The AI Agent operates as a **Story Scene Generator** with a detailed system prompt that enforces:

- A fully autonomous story invented from scratch on every run (no repeated themes or settings)
- Exactly **5 scenes** following a classic narrative arc:
  - Scene 1 — World & protagonist established
  - Scene 2 — Conflict introduced
  - Scene 3 — Tension escalated
  - Scene 4 — Climax / turning point
  - Scene 5 — Resolution & emotional close
- Each scene produces two fields:
  - `scene_text` — 3–5 sentences of literary Arabic prose
  - `image_prompt` — a 40–80 word English visual specification (lighting, palette, camera angle, mood, subject) fed directly to the image model

Output is a validated JSON object:

```json
{
  "status": "success | partial | error",
  "scenes": [
    {
      "scene_index": 1,
      "scene_text": "النص العربي للمشهد...",
      "image_prompt": "English visual prompt for image generation..."
    }
  ]
}
```

---

## Image Generation

Each scene's `image_prompt` is sent to the **Gemini 2.5 Flash Image** model via the Vertex AI REST API. The returned image is base64-encoded inline data, extracted by the **Edit Fields** node and converted to a named binary file (`image1` through `image5`) before being sent to Telegram.

The HTTP request node is configured with **retry on fail** (5-second wait between retries) to handle transient API errors gracefully.

---

## Prerequisites

Before importing this workflow, ensure you have:

1. **n8n** instance (self-hosted or cloud)
2. **Google Cloud project** with Vertex AI API enabled
   - A Google Service Account with Vertex AI permissions
   - A Vertex AI API key for the Gemini image endpoint
3. **Telegram Bot** with a bot token
   - The target channel/group chat ID (currently set to `-5326491438`)

---

## Setup & Configuration

### 1. Google Credentials

- In n8n, add a **Google Service Account** credential and link it to the `Google Vertex Chat Model` node.
- In the `Vertex AI HTTP Request` node, replace the `key` query parameter value with your own Vertex AI API key.
- Update the `projectId` in the `Google Vertex Chat Model` node to your own Google Cloud project ID (currently `n8n-gemini-app-500118`).

### 2. Telegram Credentials

- Add a **Telegram API** credential (bot token) in n8n.
- Update the `chatId` field in the three Telegram nodes (`Send a photo message`, `Send a text message`, `the end message`) to your target channel or group chat ID.

### 3. Schedule

The workflow triggers at **21:00 (9 PM)** server time daily. To change this, edit the `Schedule Trigger` node.

---

## Error Handling

The workflow includes three layers of resilience:

- **Agent-level fallbacks** — if a scene cannot be generated, the agent returns a `partial` status with null values for that scene rather than failing entirely.
- **HTTP retry** — the Vertex AI image request retries automatically on failure with a 5-second delay.
- **Agent retry** — the AI Agent node itself is also set to retry on fail.

---

## Notes

- The pipeline is currently **active** (`"active": true` in the workflow settings).
- Binary data is handled in **separate** mode (`"binaryMode": "separate"`).
- The workflow is **not** exposed via MCP (`"availableInMCP": false`).
- Scene images are named `image1` through `image5` corresponding to their `scene_index`.
- The API key embedded in the JSON should be rotated before sharing this workflow file publicly.
