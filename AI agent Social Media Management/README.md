# 🤖 AI Agent Social Media Management

An n8n workflow that **fully automates AI news discovery, content generation, and social media publishing**. It scrapes top AI industry RSS feeds daily, uses a Gemini-powered AI agent to select a fresh, non-duplicate story, generates a custom AI image, and simultaneously publishes platform-tailored posts to both **Facebook** and **Instagram** — with email alerts for success or failure.

---

## 🔄 Workflow Overview

```
Schedule Trigger
    ├── VentureBeat RSS
    ├── TechCrunch RSS     → Merge (4-in) → Clean/Normalize → AI Agent (Gemini)
    ├── Ars Technica RSS                                           │
    └── The Verge RSS                                             │
                                                           ┌──────┴──────┐
                                               Google Sheets Tools (Read + Append)
                                               Structured Output Parser
                                                                  │
                                                        Generate Image (Gemini 2.5)
                                                                  │
                                                        Extract Image → Convert to File
                                                           ┌───────┴────────┐
                                                    Facebook post      Instagram post
                                                    ┌──┴──┐             ┌──┴──┐
                                                   OK   Error         OK   Error
                                                    │     │             │     │
                                                   Merge  Gmail       Merge  Gmail
                                                    └──────┬───────────┘
                                                    Send success email
```

---

## 🧩 Workflow Steps

### 1. Schedule Trigger
Fires on a configurable recurring interval to kick off the entire pipeline automatically. Adjust the trigger's interval to control posting frequency (e.g., once a day, twice a day).

### 2. RSS Feed Collection (Parallel)
Four RSS reader nodes run in parallel, each pulling the latest articles from a dedicated AI industry source:

| Node | Feed URL |
|---|---|
| `venturebeat` | `https://venturebeat.com/category/ai/feed/` |
| `techcrunch` | `https://techcrunch.com/category/artificial-intelligence/feed/` |
| `arstechnica` | `https://arstechnica.com/ai/feed/` |
| `theverge` | `https://www.theverge.com/rss/index.xml` |

### 3. Aggregate Result (Merge Node — 4 inputs)
Waits for all four RSS readers to complete and merges their output into a single unified batch of articles.

### 4. Cleaning (Code Node)
Normalises the merged articles, stripping each item down to only the fields the AI agent needs:

```js
{ title, creator, link, contentSnippet, pubDate }
```

This keeps the AI agent's input concise and reduces token usage.

### 5. AI Agent (Google Vertex / Gemini 3.5 Flash)
The core intelligence of the workflow. It receives the full list of cleaned articles and performs a multi-step reasoning process:

1. **Reads the tracking sheet** via the `Get row(s) in sheet` tool to retrieve all previously covered articles (`article_url`, `published_date`, `ai_topic`, `facebook_copy`, `instagram_copy`).
2. **Ranks all candidates** by newsworthiness for an AI-industry audience.
3. **Filters for freshness**: rejects any article whose URL was already covered, or whose topic semantically matches a sheet row published within the last **7 days**.
4. **Selects the best qualifying article** and generates:
   - `ai_topic` — a concise topic label for deduplication.
   - `facebook_caption` — 2–4 sentences, long-form, article-focused, ending with the source link.
   - `instagram_caption` — 1–2 hook-styled lines + hashtag block, no bare URL (references "link in bio" instead).
   - `image_prompt` — a visual description for image generation, distinct from the caption text.
5. **Writes the selection** to the tracking sheet via the `Append row in sheet` tool so it won't be reposted within 7 days.
6. Returns a **structured JSON object** validated by the Structured Output Parser.

**Model:** `gemini-3.5-flash` via Google Vertex AI

**Output schema:**
```json
{
  "status": "success | no_match | error",
  "data": {
    "source_title": "string",
    "source_link": "string",
    "ai_topic": "string",
    "image_prompt": "string",
    "facebook_caption": "string",
    "instagram_caption": "string"
  },
  "err": null,
  "meta": { "tool_used": "Append row in sheet in Google Sheets | null" }
}
```

> **Edge cases handled by the agent:**
> - Missing required fields on a candidate → that candidate is skipped (not the whole batch).
> - All candidates are duplicates → returns `{ "status": "no_match" }`.
> - Empty article list → returns `{ "status": "error", "reason": "missing_field:candidate_articles" }`.
> - Tool call failure → returns error immediately without retrying.

### 6. Generate Image (HTTP Request → Gemini 2.5 Flash Image)
Sends the `image_prompt` from the agent to the **Gemini 2.5 Flash Image** generation endpoint:

```
POST https://aiplatform.googleapis.com/v1/publishers/google/models/gemini-2.5-flash-image:generateContent
```

Authenticated via an HTTP Query Auth credential (Google API key). Retries automatically on failure with a **5-second delay** between attempts.

### 7. Extract Image (Set Node)
Pulls the base64-encoded image data from the API response. Checks both part index 0 and part index 1 to handle any variation in the response structure:

```
candidates[0].content.parts[0].inlineData.data
  OR
candidates[0].content.parts[1].inlineData.data
```

### 8. Convert to File
Converts the raw base64 image string into a binary file object that can be attached to social media upload requests.

### 9. Parallel Publishing
The binary image and AI-generated captions are sent to both platforms simultaneously:

#### Facebook Post
- **Platform:** Facebook Page (ID: `1232254806641325`)
- **Content:** `source_title` + `facebook_caption` as the post body; `source_link` in the first comment; AI topic as alt-text.
- **Timezone:** `Jordan/Amman`
- **On error:** Continues to the error branch (does not halt Instagram publishing).

#### Instagram Post
- **Platform:** Instagram (linked account)
- **Content:** `source_title` + `instagram_caption` as the post body; `source_link` in the first comment; AI topic as alt-text.
- **Timezone:** `Jordan/Amman`
- **On error:** Continues to the error branch (does not halt Facebook publishing).

### 10. Merge Node
Waits for **both** the Facebook and Instagram branches to complete (success paths only) before proceeding to the success email. Uses `chooseBranch` + `empty` output mode to act as a synchronization gate without duplicating data.

### 11. Email Notifications (Gmail)

| Node | Trigger | Recipient | Content |
|---|---|---|---|
| `Send success message` | Both posts succeed | `zaidradaideh.dev@gmail.com` | HTML email with live links to the Facebook and Instagram posts |
| `Facebook error` | Facebook post fails | `zaidradaideh.dev@gmail.com` | HTML alert email asking to check workflow logs |
| `Instagram error` | Instagram post fails | `zaidradaideh.dev@gmail.com` | HTML alert email asking to check workflow logs |

---

## 🔑 Required Credentials

| Credential | Used By |
|---|---|
| **Google Service Account** (Vertex AI) | AI Agent — Gemini chat model |
| **HTTP Query Auth** (Google API key) | Generate Image — Gemini 2.5 Flash Image endpoint |
| **Google Sheets OAuth2** | `Get row(s) in sheet` and `Append row in sheet` tools |
| **Upload Post API** | Facebook Post and Instagram Post nodes |
| **Gmail OAuth2** | Success email, Facebook error email, Instagram error email |

---

## 📊 Google Sheets — Post Tracking Schema

The AI Agent reads from and writes to a Google Sheet named **"Auto posting"** (`Sheet1`) to track previously covered topics and prevent duplicate posts within a 7-day window.

| Column | Type | Description |
|---|---|---|
| `article_url` | string | The full URL of the published source article |
| `published_date` | string | The article's original `pubDate` from the RSS feed |
| `ai_topic` | string | A concise label describing the specific topic/story |
| `facebook_copy` | string | The full Facebook caption that was posted |
| `instagram_copy` | string | The full Instagram caption that was posted |

> ⚠️ **Do not delete or rename columns in this sheet.** The AI agent relies on exact column names for duplicate detection. Altering the sheet structure will break the deduplication logic.

---

## ⚙️ Setup Instructions

1. **Google Sheets** — Create a spreadsheet with the five columns above. Update the Sheet ID (`1Ue-ifw064ECZthdLUiLnY-lR8u9CHcGPLETXLLK8_SM`) in both the `Get row(s) in sheet` and `Append row in sheet` tool nodes to point to your own sheet.

2. **Gemini / Vertex AI** — Create a Google Cloud project and enable the Vertex AI API. Add a Google Service Account credential in n8n and link it to the `Google Vertex Chat Model` node.

3. **Gemini Image Generation** — Obtain a Google API key with access to the Gemini image generation endpoint. Add it as an **HTTP Query Auth** credential in n8n and link it to the `Generate image` node.

4. **Social Media (Upload Post)** — Configure the `Upload Post` credential with your Facebook Page and Instagram account tokens. Update the `facebookPageId` in the `Facebook post` node to your Page ID.

5. **Gmail** — Authenticate a Gmail OAuth2 credential and update the `sendTo` address in the three email nodes (`Send success message`, `Facebook error`, `Instagram error`) to your desired notification email.

6. **Schedule** — Open the `Schedule Trigger` node and set your preferred posting interval (e.g., once per day at 9 AM).

7. **Activate the workflow** — Toggle the workflow to Active in n8n. It will run automatically on your schedule.

---

## ⚠️ Notes

- **Fully automated, no human-in-the-loop.** Unlike approval-gate workflows, this pipeline publishes directly. Monitor the success/error emails to stay informed.
- **7-day deduplication window.** The AI agent will never cover the same specific story or topic twice within a rolling 7-day period. The Google Sheet is the agent's long-term memory.
- **No-match handling.** If all candidate articles from today's feed are duplicates, the agent returns a `no_match` status and the workflow ends gracefully without publishing anything.
- **Platform-tailored captions.** Facebook gets a long-form, article-anchored caption with a direct URL. Instagram gets a short, trend-styled hook with no bare URL (readers are directed to "link in bio").
- **Image generation retry.** The `Generate image` node retries automatically on failure, with a 5-second pause between attempts, to handle transient API rate-limit errors.
- **Error isolation.** Facebook and Instagram posts run independently. A failure on one platform sends a targeted error email but does **not** cancel the other platform's post — both attempts always run.
- **Success email dependency.** The `Send success message` node references `$('Facebook post')` and `$('Instagram post')` by node name. If you rename either publishing node, update the success email's HTML expressions to match.
- **RSS source scope.** The agent is explicitly constrained to evaluate articles only from the four configured AI industry RSS feeds. It will not source content from anywhere else.
- **Google Sheet memory.** Do not delete the tracking sheet or alter its column names. The AI agent relies on this sheet to "remember" past posts and avoid duplicates.
