# 🔍 LinkedIn DeepSight Pipeline

A two-stage AI-powered n8n workflow that scrapes a LinkedIn profile or company page, generates an interactive HTML analytics dashboard, and lets the user ask unlimited follow-up questions about the data — all via **Telegram**.

---

## 📋 Overview

The **LinkedIn DeepSight Pipeline** turns a single LinkedIn URL sent through Telegram into a full intelligence report:

1. **Scrape** — Apify pulls every public post (text, engagement metrics, comments, reactions).
2. **Analyze** — A first AI agent (DeepSight Post Analyzer) reads every post and produces:
   - A self-contained, interactive **HTML dashboard** (charts, summaries, KPIs).
   - A structured **per-post analysis array** (`vertex`) for downstream use.
3. **Deliver** — The HTML file is sent directly to the Telegram chat.
4. **Q&A** — A second AI agent (DeepSight Q&A Agent) answers any follow-up question about the profile using the `vertex` data and conversation memory, without ever needing to re-scrape.

---

## 🏗️ Architecture

```
Telegram Trigger
       │
       ▼
Check URL & Determine Route  ──►  Switch
                                    │
             ┌──────────────────────┼──────────────────────┐
             ▼                      ▼                      ▼
  ASK_FOR_URL branch         RUN_APIFY branch       ANSWER_QUESTION branch
  (Send Invalid URL           (Scrape LinkedIn)      (AI: Answer Follow-up
   Warning)                         │                      Question)
                                     ▼                      │
                             Bundle Posts into               ▼
                             Single Array            Send Answer to User
                                     │
                                     ▼
                          AI: Analyze Posts &
                           Build Dashboard
                                 │       │
                         Success │       │ Error
                                 ▼       ▼
                              To HTML   Send "Analysis Error" Alert
                                 │
                                 ▼
                      Send HTML Dashboard to User
                                 │
                                 ▼
                       Extract Saved Post Data
                                 │
                                 ▼
                      AI: Answer Follow-up Question
```

---

## ⚙️ Workflow Nodes

### 1. `Telegram Trigger`
- **Type:** Telegram Trigger
- **Purpose:** Entry point. Listens for any incoming `message` event from the connected Telegram bot.

---

### 2. `Check URL & Determine Route`
- **Type:** Code (JavaScript)
- **Purpose:** The routing brain of the pipeline. Uses persistent **workflow static data** keyed by Telegram `chat.id` to track each user's state independently.
- **Logic:**
  | Condition | Route |
  |---|---|
  | User sends `/reset` | `ASK_FOR_URL` — resets user state |
  | Message contains a `linkedin.com/...` URL | `RUN_APIFY` — saves URL, updates state to `DATA_READY` |
  | User's state is `DATA_READY` (follow-up question) | `ANSWER_QUESTION` |
  | No URL, state is `AWAITING_URL` | `ASK_FOR_URL` |
- **Output fields:** `chatId`, `text`, `route`, `url`

---

### 3. `Switch`
- **Type:** Switch
- **Purpose:** Routes execution to one of three branches based on the `route` field set by the previous node.
- **Outputs:** `ASK_FOR_URL` → `RUN_APIFY` → `ANSWER_QUESTION`

---

### Branch A — `ASK_FOR_URL`

#### 4. `Send "Invalid URL" Warning`
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that a valid LinkedIn URL is required, with an example format.

---

### Branch B — `RUN_APIFY`

#### 5. `Send "Scraping Started" Notice`
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Sends an acknowledgment to the user (in Arabic) that the LinkedIn URL was received and scraping has begun.

#### 6. `Scrape LinkedIn (Apify)`
- **Type:** Apify (`Wpp1BZ6yGWjySadk3` actor)
- **Purpose:** Runs the LinkedIn scraper actor against the submitted URL.
- **Configuration:**
  - `deepScrape: true`
  - `numComments: 10`, `numLikes: 5`
  - `limitPerSource: 100` posts
  - `rawData: false`
- **Error Handling:** On failure, routes to `Apify Error` node.

#### 7. `Apify Error`
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that the scrape failed, and advises them to check the URL visibility and try again.

#### 8. `Bundle Posts into Single Array`
- **Type:** Code (JavaScript)
- **Purpose:** Collects all Apify output items into a single `{ posts: [...] }` object. This prevents the downstream AI Agent from looping over items individually.

#### 9. `Send "AI Analysis Started" Notice`
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that data collection is complete and AI analysis is now in progress.

#### 10. `AI: Analyze Posts & Build Dashboard` *(DeepSight Post Analyzer — Agent 1)*
- **Type:** LangChain AI Agent
- **LLM:** Google Vertex AI (`gemini-3.6-flash`)
- **Output Parser:** Structured Output Parser (strict JSON schema)
- **Purpose:** The core analysis engine. Reads every post in the `posts` array and returns a single JSON object with two keys:
  - **`html`** — A fully self-contained HTML report with charts (Chart.js via CDN), KPIs, engagement breakdowns, posting frequency, content-type distribution, and per-post summaries.
  - **`vertex`** — A structured array with one entry per post, each containing: `post_url`, `posted_date`, `content_type`, `engagement`, `topic`, `sentiment`, `summary`, `audience_reaction`.
- **Guarantees:**
  - Processes **every** post — no sampling or early stopping.
  - Never fabricates data; uses only values present in the input.
  - `vertex.length` must equal `posts.length` before returning.
- **Error Handling:** On failure, routes to `Send "Analysis Error" Alert`.

#### 11. `Send "Analysis Error" Alert`
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that AI analysis failed, citing possible causes (data volume, temporary server error).

#### 12. `To HTML`
- **Type:** Code (JavaScript)
- **Purpose:** Extracts the `html` string from the AI agent's output and converts it to a binary file buffer (`text/html`, `linkedin-report.html`) ready for Telegram's file-send API.

#### 13. `Send HTML Dashboard to User`
- **Type:** Telegram (sendDocument)
- **Purpose:** Delivers the `linkedin-report.html` file directly to the user's Telegram chat.

#### 14. `Extract Saved Post Data`
- **Type:** Code (JavaScript)
- **Purpose:** Extracts the `vertex` array from Agent 1's output and passes it forward as the knowledge base for Agent 2.

#### 15. `AI: Answer Follow-up Question` *(DeepSight Q&A Agent — Agent 2)*
*(Also the target of Branch C — see below)*
- **Type:** LangChain AI Agent
- **LLM:** Google Vertex AI (`gemini-3.6-flash`)
- **Memory:** Window Buffer Memory (last 10 messages, keyed by `chat.id`)
- **Purpose:** Answers the user's first follow-up question immediately after dashboard delivery. Uses only the `vertex` data — never re-scrapes. Refuses out-of-scope questions with a fixed, polite message.

---

### Branch C — `ANSWER_QUESTION`

This branch also routes to **`AI: Answer Follow-up Question`** (node 15 above). It handles any subsequent message from a user whose state is already `DATA_READY`, enabling multi-turn Q&A without re-scraping.

---

### 16. `Send Answer to User`
- **Type:** Telegram (sendMessage)
- **Purpose:** Sends Agent 2's plain-text answer back to the user's Telegram chat.

#### 17. `Send "Answer Error" Message`
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that Agent 2 failed to answer, and suggests rephrasing the question or using `/reset`.

---

## 🔗 Dependencies & Credentials

| Service | Credential Name | Purpose |
|---|---|---|
| Telegram | `Telegram account` | Bot trigger + all message sends |
| Google Vertex AI | `Google Service Account account 3` | LLM for both AI agents (`gemini-3.6-flash`) |
| Apify | `Apify account` | LinkedIn scraper actor `Wpp1BZ6yGWjySadk3` |

> **Note:** The Google Vertex project is `n8n-gemini-app-500118`.

---

## 🚀 Setup

1. **Import** `workflow.json` into your n8n instance.
2. **Configure credentials:**
   - Connect a Telegram Bot token under `Telegram account`.
   - Add a Google Cloud Service Account with Vertex AI access under `Google Service Account account 3`.
   - Add your Apify API key under `Apify account`.
3. **Activate** the workflow — the Telegram webhook registers automatically.
4. **Send** any LinkedIn profile or company page URL to your Telegram bot to begin.

---

## 💬 Usage

| User Action | Bot Response |
|---|---|
| Send a `linkedin.com/...` URL | Scrapes profile → Analyzes posts → Delivers HTML dashboard |
| Send any follow-up question | Answers from the `vertex` data (no re-scrape) |
| Send `/reset` | Resets state — user can submit a new URL |
| Send text without a URL (first time) | Asks user to provide a LinkedIn URL |

---

## ⚠️ Notes & Limitations

- **Public profiles only** — Apify can only scrape publicly accessible LinkedIn pages.
- **Per-user state** — Each Telegram `chat.id` is tracked independently via workflow static data, so multiple users can run the pipeline concurrently.
- **Post cap** — The Apify actor is configured to scrape up to **100 posts** per run (`limitPerSource: 100`).
- **`numImpressions`** is intentionally excluded from analysis (always `null` in LinkedIn's API response).
- **Language** — Status/error messages sent to the user are in **Arabic**. The AI agent responses adapt to the user's question language.
- **HTML report** — Delivered as a downloadable `.html` file. Open it in any browser for the interactive charts.
