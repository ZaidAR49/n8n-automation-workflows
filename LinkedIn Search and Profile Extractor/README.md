# 🔍 LinkedIn Search & Profile Extractor

An n8n automation workflow that accepts a search request via **Telegram**, scrapes LinkedIn for matching profiles using **Apify**, scores and ranks results with **GPT-5 Mini**, and delivers a beautifully formatted HTML report back to the user — all without touching LinkedIn's UI.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Diagram](#workflow-diagram)
- [Prerequisites](#prerequisites)
- [Setup & Configuration](#setup--configuration)
- [How to Use](#how-to-use)
- [Node Reference](#node-reference)
- [AI Scoring System](#ai-scoring-system)
- [Output Format](#output-format)
- [Edge Cases & Error Handling](#edge-cases--error-handling)
- [Tips & Troubleshooting](#tips--troubleshooting)

---

## Overview

| Property        | Details                                       |
|-----------------|-----------------------------------------------|
| **Trigger**     | Telegram Bot Message                          |
| **Scraping**    | Apify (LinkedIn Profile Search + Full Scrape) |
| **AI Model**    | GPT-5 Mini (OpenAI)                           |
| **Output**      | Telegram HTML formatted message               |
| **Max Profiles**| 10 searched → top 5 displayed                |

**Use cases:**
- Quickly find a specific person on LinkedIn by name + contextual notes
- Verify someone's professional background before a meeting
- Research candidates, leads, or contacts without manual LinkedIn browsing

---

## Workflow Diagram

```
Telegram Trigger
      │
      ▼
Extract Name & Notes (Code)
      │
      ▼
Is Valid Format? (IF)
  ├── ❌ No  ──► Request Correct Format (Telegram)
  │
  └── ✅ Yes ──► Apify: Search Profiles
                      │
                      ▼
               Apify: Fetch Full Profile
                      │
                      ▼
               AI: Select Best Match (GPT-5 Mini)
                      │
                      ▼
               Is Match? (IF)
                  └── ✅ ──► Send Final Report (Telegram)
```

---

## Prerequisites

Before importing and activating this workflow, you need:

1. **n8n** instance (self-hosted or cloud)
2. **Telegram Bot** — Create one via [@BotFather](https://t.me/BotFather)
3. **Apify Account** — [apify.com](https://apify.com) (free tier available)
   - Actor: `apimaestro/linkedin-profile-search-scraper` (ID: `pIyH7237rHZBxoO7q`)
   - Actor: `harvestapi/linkedin-profile-scraper` (ID: `LpVuK3Zozwuipa5bp`)
4. **OpenAI API Key** — [platform.openai.com](https://platform.openai.com) with access to `gpt-5-mini`

---

## Setup & Configuration

### 1. Import the Workflow

In your n8n instance:
1. Go to **Workflows** → **Import**
2. Upload `workflow.json`

### 2. Configure Credentials

Set up the following credentials in n8n (**Settings → Credentials**):

| Credential Name    | Type       | Where Used                          |
|--------------------|------------|-------------------------------------|
| `Telegram account` | Telegram API | Telegram Trigger, Send nodes      |
| `Apify account`    | Apify API  | Both Apify scraper nodes            |
| `OpenAI API`       | OpenAI API | AI: Select Best Match               |

### 3. Configure the Telegram Chat ID

In both the **Send Final Report** and **Request Correct Format** nodes, update the `chatId` field to your own Telegram user ID or group ID.

> **Tip:** Send a message to [@userinfobot](https://t.me/userinfobot) on Telegram to get your Chat ID.

### 4. Activate the Workflow

Toggle the workflow to **Active** in n8n. The Telegram webhook will begin listening for messages.

---

## How to Use

Send a message to your Telegram bot using **exactly** this format:

```
Name: Target Name
Notes: Write any details, context, or language here
```

**Example:**

```
Name: Jane Doe
Notes: Senior software engineer specializing in machine learning, based in San Francisco, previously at Google
```

The bot will:
1. ✅ Confirm the format is valid and start searching
2. 🔍 Scrape up to 10 LinkedIn profiles matching the name
3. 📄 Fetch full profile details for each result
4. 🤖 Score and rank all profiles against your notes
5. 📨 Send back a formatted report with the best matches

---

## Node Reference

### 1. Telegram Trigger
- **Type:** `n8n-nodes-base.telegramTrigger`
- **Listens for:** New messages sent to the bot
- **Outputs:** Raw Telegram message object

### 2. Extract Name & Notes
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Logic:** Uses regex to parse the message for `Name:` and `Notes:` fields
- **Outputs:**
  - `status` — `true` if both fields found, `false` otherwise
  - `firstName`, `lastName` — split from the name
  - `searchNotes` — raw notes text

### 3. Is Valid Format?
- **Type:** `n8n-nodes-base.if`
- **Condition:** Checks `status === true`
- **Branch True (output 1):** Proceeds to Apify scraping
- **Branch False (output 2):** Sends format correction message to user

### 4. Apify: Search Profiles
- **Type:** `@apify/n8n-nodes-apify.apify`
- **Actor:** `apimaestro/linkedin-profile-search-scraper`
- **Input:** `firstName`, `lastName`, `max_profiles: 10`
- **Note:** Retry on fail is enabled with a 2-second delay to handle proxy timeouts

### 5. Apify: Fetch Full Profile
- **Type:** `@apify/n8n-nodes-apify.apify`
- **Actor:** `harvestapi/linkedin-profile-scraper`
- **Mode:** `Profile details no email ($4 per 1k)`
- **Input:** Profile URLs from the search step

### 6. AI: Select Best Match
- **Type:** `@n8n/n8n-nodes-langchain.openAi`
- **Model:** `gpt-5-mini`
- **Task:** Scores all profiles against the search notes and returns a Telegram-formatted HTML report (top 5 profiles max)

### 7. Is Match?
- **Type:** `n8n-nodes-base.if`
- **Condition:** AI output is not the "no profiles found" fallback message
- **Branch True:** Forwards to Send Final Report
- **Branch False:** (no further action — the AI output already contains the warning)

### 8. Send Final Report
- **Type:** `n8n-nodes-base.telegram`
- **Format:** Sends HTML-formatted message to the configured chat ID

### 9. Request Correct Format
- **Type:** `n8n-nodes-base.telegram`
- **Triggered when:** Message did not match the `Name: / Notes:` format
- **Action:** Sends a copyable template back to the user

---

## AI Scoring System

The AI node uses a deterministic scoring algorithm to rank profiles:

| Score | Label  | Icon |
|-------|--------|------|
| 8–10  | High   | ✅   |
| 5–7   | Medium | ⚠️   |
| 1–4   | Low    | ❌   |

**Scoring Formula:**

```
base_score = 5
+ 1 per matched keyword      (capped at +5)
- 2 per contradicted keyword (no cap)
Clamped to range: 1–10
```

**Fields used for scoring:** headline, about, current company, top skills, location, experience (title, company, description), education (school, degree, field), certifications, languages.

**Fields never used for scoring or display:** email, follower/connection counts, profile pictures, internal IDs, timestamps.

---

## Output Format

Each profile block in the Telegram report follows this exact structure:

```
👤 Full Name

  💼 Job Title / Headline
  🏢 Company: Company Name
  📍 Location: City, Country

🎯 Confidence: 9/10 ✅ (High)
📝 Matched keywords: Software Engineer; React; San Francisco; Google

🔗 View LinkedIn Profile

━━━━━━━━━━━━━━━━━━━━━━
```

- Up to **5 profiles** are shown per search, ordered by descending confidence score
- Missing fields display `Not listed` rather than being omitted
- If no profiles are found: `⚠️ No LinkedIn profiles were found for this search. Try again with different details.`

---

## Edge Cases & Error Handling

| Situation | Behavior |
|-----------|----------|
| Message does not match `Name: / Notes:` format | Bot replies with a copyable format template |
| Apify returns no profiles | AI returns a "no profiles found" warning message |
| No keywords extractable from notes | Every profile assigned score 5; most complete profile shown |
| All profiles score 1–4 (Low) | Only top-scoring profile shown with a low-confidence warning |
| Profile has no `profile_url` | Excluded entirely from scoring and output |
| Two profiles tie for top score | Both shown (subject to 5-profile cap) |
| Apify proxy timeout | Node retries automatically (retry on fail enabled, 2s delay) |

---

## Tips & Troubleshooting

**🔧 Apify actor not running?**
- Confirm your Apify API key is valid and has sufficient credits
- `max_profiles` is set to `10` — increase for broader searches or decrease to save credits

**🤖 AI output looks wrong or inconsistent?**
- Ensure the OpenAI model is `gpt-5-mini` and temperature is `0` for deterministic output
- Do not modify the AI system prompt structure — it is carefully engineered for Telegram HTML output

**📨 Telegram message not sending?**
- Confirm the `chatId` in both Telegram send nodes matches your actual user/group ID
- Telegram's HTML parser is strict — do not modify the AI prompt's HTML tag structure

**💸 Reducing Apify costs?**
- Lower `max_profiles` from `10` to `5` for quicker, cheaper searches
- The profile scraper is billed at `$4 per 1,000 profiles` — light usage stays within free tier limits

**📝 Notes tips for best results:**
- Be specific: include job title, company, location, skills, or university
- Avoid vague notes like "software engineer" alone — add company or location for better scoring
- Multi-word phrases like "quality assurance" or "machine learning" are treated as single keywords
