# 📡 Google Tech News Pipeline

An automated n8n workflow that polls the **Google Blog RSS feed** every hour, enriches each article with AI-generated Arabic editorial content via **Gemini 3.7 Flash** on Google Vertex AI, and broadcasts formatted posts simultaneously to a **Discord** channel and a **Telegram** channel.

---

## 📋 Overview

The **Google Tech News Pipeline** keeps your community up to date with the latest Google tech news — fully automated, bilingual, and editorially enriched:

1. **Poll** — An RSS trigger checks `blog.google/rss` every hour for new articles.
2. **Parse** — A JavaScript code node cleans the raw RSS data, extracts image URLs, and normalises categories.
3. **Download** — An HTTP Request node fetches the article's cover image as a binary file.
4. **Enrich** — A Gemini LLM chain rewrites each article into Arabic with a headline, summary, importance score, alert label, and hashtags.
5. **Broadcast** — The enriched article is posted in parallel to Discord (as a rich embed) and Telegram (as a photo message with a Markdown caption).

---

## 🏗️ Architecture

```
Poll Google RSS (every hour)
        │
        ▼
Clean & Parse Feed  ─── extract imageUrl, url, categories, publishedAt, snippet
        │
        ▼
Download Cover Image  ─── binary file fetched via HTTP Request
        │
        ▼
Enrich with Gemini  ◄── Google Vertex Chat Model (gemini-3.7-flash)
   (LLM Chain)      ◄── Structured Output Parser
        │
        ├──────────────────────────────────────┐
        ▼                                      ▼
Send a message (Discord embed)     Send a photo message (Telegram)
```

---

## ⚙️ Workflow Nodes

### 1. `Poll Google RSS`
- **Type:** RSS Feed Read Trigger
- **Feed URL:** `https://blog.google/rss`
- **Poll Interval:** Every hour
- **Purpose:** Entry point. Detects new articles published on the Google Blog and triggers the pipeline for each one.

---

### 2. `Clean & Parse Feed`
- **Type:** Code (JavaScript)
- **Purpose:** Normalises the raw RSS item into a clean, structured object for downstream nodes.

| Step | Detail |
|---|---|
| URL extraction | Reads `link`, `guid`, or `id` fields to get the canonical article URL |
| Image extraction | Regex-scans the raw HTML `content` field for the first `<img src="…">` |
| Snippet cleaning | Strips all HTML tags except `<a>` links from `contentSnippet` or `description` |
| Category normalisation | Flattens array or string `categories` into a comma-separated string |

**Output fields:** `title`, `imageUrl`, `snippet`, `url`, `categories`, `publishedAt`

---

### 3. `Download Cover Image`
- **Type:** HTTP Request
- **URL:** `{{ $json.imageUrl }}`
- **Response format:** File (binary)
- **Purpose:** Downloads the article cover image as binary data so it can be attached directly to the Telegram photo message.

---

### 4. `Enrich with Gemini` *(LLM Chain)*
- **Type:** Basic LLM Chain (`@n8n/n8n-nodes-langchain.chainLlm`)
- **Model:** Google Vertex Chat Model — `gemini-3.7-flash`
- **Output Parser:** Structured Output Parser (JSON schema)
- **Purpose:** The core AI enrichment step. The **Tech News Enrichment Agent** system prompt instructs Gemini to:
  - Rewrite the headline in Arabic with an optional emoji prefix.
  - Generate a 2–4 sentence Arabic summary grounded only in the article snippet.
  - Write a 1–2 sentence Arabic "why it matters" explanation.
  - Assign a fixed category from the allowed list.
  - Score importance (1–10) with a short Arabic rationale.
  - Derive the alert label mechanically from the importance score.
  - Generate 3–6 English hashtags from named entities in the article.
  - Pass through all original fields unchanged.

**Alert label logic:**

| Importance Score | Alert Label |
|---|---|
| 9–10 | 🚨 BREAKING |
| 6–8 | 🔥 IMPORTANT |
| 1–5 | 📰 NEWS |

**Allowed categories:** `AI`, `Software`, `Hardware`, `Cybersecurity`, `Cloud`, `Programming`, `Startups`, `Business`, `Science`

---

### 5. `Send a message` *(Discord)*
- **Type:** Discord (`n8n-nodes-base.discord`) — OAuth2
- **Guild:** `1531264583166333078`
- **Channel:** `google-news` (`1538230524852568155`)
- **Purpose:** Posts the enriched article as a Discord embed with:
  - **Title:** Arabic headline (linked to article URL)
  - **Description:** Alert label, category, headline, summary, importance score, and tags
  - **Image:** Article cover image (via URL)
  - **Timestamp:** `publishedAt` date
  - **Color:** `#427BD0`
  - **Author:** Zaid Radaideh

---

### 6. `Send a photo message` *(Telegram)*
- **Type:** Telegram (`n8n-nodes-base.telegram`) — sendPhoto
- **Chat ID:** `-1004361710069`
- **Purpose:** Sends the downloaded cover image as a Telegram photo with a Markdown caption containing:
  - Alert label and category header
  - Arabic headline and summary
  - "Why it matters" section
  - Importance score and hashtags
  - Published date and a deep link to the full article

---

## 🔗 Dependencies & Credentials

| Service | Credential Name | Purpose |
|---|---|---|
| Google Vertex AI | `Google Service Account account` | Powers the Gemini 3.7 Flash LLM chain |
| Discord | `Discord account` (OAuth2) | Posts rich embeds to the `google-news` channel |
| Telegram | `Telegram account 4` | Sends photo messages with Markdown captions |

---

## 🚀 Setup

1. **Import** `workflow.json` into your n8n instance.
2. **Configure credentials:**
   - Add a **Google Service Account** with Vertex AI access under `Google Service Account account`. Ensure the `Vertex AI API` is enabled on your GCP project (`n8n-gemini-app`).
   - Connect a **Discord OAuth2** application with `bot` scope and `Send Messages` permission under `Discord account`.
   - Add your **Telegram Bot token** under `Telegram account 4`.
3. **Update target IDs** (if needed):
   - Discord Guild ID and Channel ID in the `Send a message` node.
   - Telegram Chat ID in the `Send a photo message` node.
4. **Activate** the workflow — the RSS poller starts immediately on the next hour.

---

## 📤 Output Format

### Discord Embed

```
🔥 IMPORTANT • AI

🤖 جوجل تطلق Gemini 3.7 Flash بأداء أقوى للبرمجة والوكلاء الذكية

أعلنت جوجل عن إطلاق Gemini 3.7 Flash …

💡 الأهمية:
يعزز هذا الإصدار قدرات جوجل التنافسية …

📊 التقييم: `8/10 — تحديث تقني مهم`
#Gemini #Google #AI #LLM
```

### Telegram Photo Caption *(Markdown)*

```
🔥 IMPORTANT • *AI*

*🤖 جوجل تطلق Gemini 3.7 Flash …*

أعلنت جوجل عن إطلاق Gemini 3.7 Flash …

💡 *الأهمية:*
يعزز هذا الإصدار …

📊 *التقييم:* `8/10 — تحديث تقني مهم`
🏷 *الوسوم:* #Gemini #Google #AI #LLM
📅 *تاريخ النشر:* 2026-08-13

[🔗 قراءة المقال كاملاً](https://blog.google/…)
```

---

## ⚠️ Notes & Limitations

- **Hourly polling** — The workflow fires once per hour. Articles published between polls are picked up on the next run. Duplicate suppression relies on n8n's built-in RSS deduplication (based on `guid`/`link`).
- **Image requirement** — If the RSS item contains no `<img>` tag in its content, `imageUrl` will be empty. The `Download Cover Image` node will fail and skip that item; you may want to add an error branch or a fallback image URL.
- **Gemini grounding** — The LLM is instructed to use **only** the article `title` and `snippet` as factual sources. It will not hallucinate facts from outside knowledge. If the snippet is missing, the summary is prefixed with `بيانات محدودة من المصدر:`.
- **Language** — All editorial fields (`headline`, `summary`, `why_it_matters`, `importance_score`) are generated in **Modern Standard Arabic**. Category, tags, and alert labels remain in English.
- **Telegram file size** — The cover image is downloaded and re-uploaded via the Telegram Bot API. Images larger than 10 MB may be rejected; the bot API limit for photos is 10 MB.
- **Discord rate limits** — Posting many articles in quick succession may trigger Discord's rate limiter. Consider adding a `Wait` node between iterations if you process batched backfills.
- **GCP project** — The workflow is pre-configured for GCP project `n8n-gemini-app`. Update the `projectId` in the `Google Vertex Chat Model` node if you use a different project.
