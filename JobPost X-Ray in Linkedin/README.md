# 💼 JobPost X-Ray in LinkedIn

> An intelligent n8n automation that deep-dives into any LinkedIn job post and delivers a rich, structured intelligence report straight to your Telegram chat — powered by **Apify** scrapers and **Google Gemini AI**.

---

## 📋 Overview

Send a LinkedIn job or post URL to a Telegram bot, and this workflow automatically:

1. **Validates** the URL format
2. **Scrapes** the post content, the author/HR profile, and the hiring company data
3. **Analyzes** all collected data with a Gemini AI model
4. **Delivers** a polished Markdown report back to your Telegram chat

The report covers three dimensions: the **Opportunity** (role, requirements, how to apply), the **HR / Poster** profile (background, experience), and the **Company** (industry, size, locations, website).

---

## 🔄 Workflow Diagram

```
Telegram Trigger
       │
       ▼
Validate LinkedIn URL ──(invalid)──► Send Invalid URL Error
       │
       ▼
Set User Preferences (language: ar/en)
       │
       ▼
Is Valid URL? ──(no)──► Send Invalid URL Error
       │ yes
       ▼
Send Process Status ("⏳ Fetching...")
       │
       ▼
Apify: Fetch Post Data ──(error/empty)──► Send Fetch Post Error
       │
       ▼
Is Data Empty? ──(empty)──► Send Fetch Post Error
       │ has data
       ▼
Is Author a Company?
    │                │
   yes (company)    no (individual)
    │                │
    │         Apify: Fetch Author Data ──(error)──► Send Fetch Author Error
    │                │
    └────────────────┘
                     │
                     ▼
            Is Data Empty?1 ──(empty)──► Send Fetch Author Error
                     │ has data
                     ▼
            Format Company ID / URL
                     │
                     ▼
            Apify: Fetch Company Data ──(error)──► Send Fetch company Error
                     │
                     ▼
            Is Data Empty?2 ──(empty)──► Send Fetch company Error
                     │ has data
                     ▼
            Change Process Status ("✅ Analyzing...")
                     │
                     ▼
            Extract Key Data (Code node)
                     │
                     ▼
            AI: Generate Report (Gemini via LLM Chain)
                     │              │
                   success        error
                     │              │
                     ▼              ▼
               Send Report    Send AI Error
```

---

## ✨ Features

| Feature | Details |
|---|---|
| 🔗 **URL Validation** | Regex-based check for `linkedin.com/posts`, `/feed/update`, and `/jobs/view` URLs |
| 🤖 **Smart Author Detection** | Detects whether the post was made by an individual or a company page and routes scraping accordingly |
| 🏢 **3-Layer Scraping** | Fetches post content, HR/recruiter profile, and company data via separate Apify actors |
| 🧠 **AI Analysis** | Gemini 3.7 Flash generates a structured intelligence report from the collected data |
| 🌐 **Bilingual Support** | Full Arabic (`ar`) and English (`en`) support for all bot messages and the final report |
| ⚡ **Fail-Fast Design** | Every scrape step has an empty-data and error check to stop the flow early, saving Apify credits and AI tokens |
| 📲 **Live Status Updates** | Sends an initial "⏳ Fetching..." message, then edits it in-place to "✅ Analyzing..." as the workflow progresses |
| 🛡️ **Error Handling** | Distinct, user-friendly error messages for each failure point (invalid URL, private post, scrape failure, AI error) |

---

## 🧩 Nodes Breakdown

### Trigger & Input
| Node | Type | Purpose |
|---|---|---|
| **Telegram Trigger** | Telegram Trigger | Listens for incoming messages containing a LinkedIn URL |
| **Validate LinkedIn URL** | Code (JS) | Regex-extracts the URL from the message; sets `isValid` and `url` |
| **Set User Preferences** | Set | Initializes the `lan` variable (currently hardcoded to `ar`; change to `en` if needed) |
| **Is Valid URL?** | IF | Branches: valid URL → proceed; invalid → error message |

### Scraping Layer (Apify)
| Node | Apify Actor ID | Purpose |
|---|---|---|
| **Apify: Fetch Post Data** | `d0DhjXPjkkwm4W5xK` | Scrapes the LinkedIn post text, author name, and author `company_id` |
| **Apify: Fetch Author Data** | `LpVuK3Zozwuipa5bp` | Scrapes the individual HR/recruiter's LinkedIn profile |
| **Apify: Fetch Company Data** | `UwSdACBp7ymaGUJjS` | Scrapes the hiring company's LinkedIn page |

### Routing & Data Prep
| Node | Type | Purpose |
|---|---|---|
| **Is Author a Company?** | IF | Checks if `author.company_id` is set; skips author scrape if posting as a company |
| **Is Data Empty? (×3)** | IF | Validates each Apify response is non-empty before continuing |
| **Format Company ID / URL** | Set | Resolves the correct company LinkedIn URL from either author data or `company_id` |
| **Extract Key Data** | Code (JS) | Distills raw Apify JSON into a compact, token-efficient object for the AI |
| **Change Process Status** | Telegram | Edits the earlier "⏳ Fetching..." message to "✅ Analyzing..." |

### AI & Output
| Node | Type | Purpose |
|---|---|---|
| **Google Vertex Chat Model** | LM Chat (Vertex AI) | Provides Gemini 3.7 Flash as the language model |
| **AI: Generate Report** | LLM Chain | Runs the AI prompt with the structured data to produce a Markdown report |
| **Send Report** | Telegram | Sends the final Markdown report to the user (RTL-aware) |

### Error Handling
| Node | Trigger Condition |
|---|---|
| **Send Invalid URL Error** | URL doesn't match LinkedIn patterns |
| **Send Fetch Post Error** | Apify post scrape returns an error or empty dataset |
| **Send Fetch Author Error** | Apify author scrape returns an error or empty dataset |
| **Send Fetch company Error** | Apify company scrape returns an error or empty dataset |
| **Send Fetch company Error1** | AI generation step fails |

---

## 🛠️ Prerequisites & Setup

### Required Credentials

| Service | Credential Type | Where Used |
|---|---|---|
| **Telegram Bot** | `telegramApi` | All Telegram nodes |
| **Apify** | `apifyApi` | All three Apify scraper nodes |
| **Google Cloud (Vertex AI)** | `googleApi` (Service Account) | Google Vertex Chat Model node |

### Step-by-Step Setup

1. **Telegram Bot**
   - Create a bot via [@BotFather](https://t.me/botfather) and get the token.
   - Add it as a credential named `Telegram account 5` (or update the node references).

2. **Apify Account**
   - Sign up at [apify.com](https://apify.com) and get an API token.
   - The following actors are used — ensure they are available in your Apify account:
     - `d0DhjXPjkkwm4W5xK` — LinkedIn Post Scraper
     - `LpVuK3Zozwuipa5bp` — LinkedIn Profile Scraper
     - `UwSdACBp7ymaGUJjS` — LinkedIn Company Scraper

3. **Google Vertex AI**
   - Create a Google Cloud project and enable the Vertex AI API.
   - Create a Service Account with `Vertex AI User` role and download the JSON key.
   - Add it as a `Google Service Account` credential in n8n.
   - Set the **Project ID** in the `Google Vertex Chat Model` node to match your GCP project.

4. **Language Setting**
   - Open the **Set User Preferences** node.
   - Change the `lan` value from `ar` to `en` for English output, or implement dynamic detection logic.

---

## 📨 How to Use

1. Start a chat with your Telegram bot.
2. Send a message containing a LinkedIn job post or post URL, e.g.:
   ```
   https://www.linkedin.com/posts/johndoe_were-hiring-software-engineer-activity-1234567890-aB-C/
   ```
   or
   ```
   https://www.linkedin.com/jobs/view/1234567890/
   ```
3. The bot replies with `⏳ Link received! Fetching...`
4. After scraping completes, the message updates to `✅ Data collected! Analyzing...`
5. The final report is sent as a structured Markdown message covering:
   - 🎯 The Opportunity (title, location, type, requirements, how to apply)
   - 👤 The HR / Poster profile
   - 🏢 The Company details
   - 🚩 Quick insights & flags

---

## 💡 Design Decisions

### Fail-Fast Pattern
Every Apify response is checked for emptiness *before* passing data downstream. This prevents wasted API calls and AI token costs when a post is private, deleted, or otherwise inaccessible.

### Token Optimization
The **Extract Key Data** code node distills three large Apify JSON blobs into a minimal, flat object before handing it to the AI — drastically reducing token usage.

### Company vs. Individual Routing
LinkedIn posts can be authored by either a person or a company page. The **Is Author a Company?** node checks for `company_id` in the post data to skip the individual profile scrape when unnecessary.

### In-Place Status Updates
Instead of flooding the chat with multiple messages, the workflow sends one status message and then edits it using `editMessageText` as the pipeline progresses.

---

## ⚠️ Limitations

- **LinkedIn Privacy:** Private posts, posts behind login walls (for non-connections), or deleted posts will return empty data and trigger an error message.
- **Apify Costs:** Each run consumes Apify compute units across up to 3 actor runs. Monitor your usage on the Apify dashboard.
- **Rate Limits:** Heavy usage may trigger LinkedIn's anti-scraping measures, causing actor failures.
- **Language Hardcoded:** The `lan` variable in **Set User Preferences** is currently set to `ar`. To support dynamic language switching, add per-user language detection or a preference command.

---

## 🔗 Related

- [n8n Documentation](https://docs.n8n.io)
- [Apify Platform](https://apify.com)
- [Google Vertex AI](https://cloud.google.com/vertex-ai)
- [Telegram Bot API](https://core.telegram.org/bots/api)
