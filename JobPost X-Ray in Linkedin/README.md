# 💼 JobPost X-Ray in LinkedIn

An AI-powered n8n automation workflow that transforms LinkedIn job post URLs into comprehensive intelligence reports — delivered directly to Telegram. It scrapes the post, the recruiter's profile, and the company's data, then uses Gemini AI to generate a structured analysis with actionable insights, and optionally drafts a personalized Gmail application email.

---

## ✨ Features

| Feature | Description |
|---|---|
| 🔗 **LinkedIn Post Scraping** | Extracts full job post content from any LinkedIn post or job URL |
| 👤 **HR/Recruiter Profiling** | Fetches the poster's LinkedIn profile — name, headline, education, experience, and bio |
| 🏢 **Company Intelligence** | Scrapes the employer's company page — industry, size, locations, website, and description |
| 🤖 **AI-Powered Analysis** | Uses Google Gemini (with Anthropic Claude fallback) to generate a structured Markdown report |
| 📩 **Auto Email Drafting** | Detects application emails in the post and offers to create a Gmail draft with your CV attached |
| 🌐 **Bilingual Support** | Full Arabic (RTL) and English support — auto-detects language from the post content |
| 🔐 **User Authentication** | Restricts bot access to a configurable whitelist of Telegram user IDs |
| 🔗 **Short URL Expansion** | Automatically resolves `lnkd.in` shortened URLs before scraping |
| ⚡ **Fail-Fast Architecture** | Halts immediately on scrape failure to save Apify credits and AI tokens |
| 💡 **Cross-Check Insights** | Flags domain mismatches, role/industry inconsistencies, and missing critical fields |
| 🛡️ **Zero-Hallucination Email** | AI extracts emails only when explicitly designated as the application channel in the original post |

---

## 🔄 Workflow Overview

```
Telegram Trigger
    └── Parse User Input (JS)
            └── Set User Preferences
                    └── Switch (Route by Action)
                            │
                            ├── [Auth Error]      → Send Unauthorized Message
                            │
                            ├── [Send Email]      → Download CV (Google Drive)
                            │                          └── Create Gmail Draft
                            │                               └── Telegram: Draft Ready Notice
                            │
                            ├── [Generate Report] → Is Short URL? (lnkd.in)
                            │                          ├── [Yes] → HTTP: Expand URL ─┐
                            │                          └── [No] ─────────────────────┘
                            │                                └── Send Process Status (⏳)
                            │                                     └── Apify: Fetch Post Data
                            │                                           ├── [Empty] → Send Post Error
                            │                                           └── [OK]    → Is Author a Company?
                            │                                                  ├── [Company] → Format Company ID ──┐
                            │                                                  └── [Person]  → Apify: Fetch Author ┘
                            │                                                          └── Format Company ID / URL
                            │                                                               └── Apify: Fetch Company Data
                            │                                                                    └── Update Status (✅)
                            │                                                                         └── Extract Key Data (JS)
                            │                                                                              └── AI: Generate Report
                            │                                                                                   ├── [Error] → Send AI Error
                            │                                                                                   └── [OK]    → Send Report
                            │                                                                                        └── Is Owner?
                            │                                                                                             └── Check HR Email
                            │                                                                                                  ├── [Found]   → Ask to Draft Email (Inline KB)
                            │                                                                                                  └── [Missing] → Notify No Email
                            │
                            └── [Invalid URL]     → Send Invalid URL Error
```

---

## 🧩 Workflow Nodes

### 1. `Telegram Trigger`
- Listens for incoming **messages** and **callback queries** (inline button responses).
- Entry point for both new LinkedIn URLs and email draft confirmations.

### 2. `Parse User Input` *(JavaScript Code)*
- Core input parsing node. Handles two distinct input types:
  - **LinkedIn URL** — validates against a regex for `/posts/`, `/feed/update/`, `/jobs/view/`, and `lnkd.in` short links.
  - **Callback Query** — detects `"true"/"false"` from the inline keyboard for email drafting.
- Extracts hidden metadata (role, company, HR name, email) encoded in the callback message URL for downstream use.

### 3. `Set User Preferences`
- Sets global configuration variables:
  - `lan` — Language code (`ar` or `en`)
  - `user_name` — Your full name (used in email drafts)
  - `CV url` — Google Drive link to your CV/resume
  - `chat_ids` — Array of authorized Telegram user IDs

### 4. `Switch` *(Smart Router)*
- Routes traffic into four branches based on input context:
  - **Authentication Error** → user's Telegram ID not in `chat_ids`
  - **Sending Email** → callback query with `data === "true"`
  - **Generate Report** → valid LinkedIn URL detected
  - **Fallback** → invalid URL or unrecognized input

---

### 🔗 URL Resolution Branch

#### `If` (Short URL Check)
- Checks if the URL contains `lnkd.in` — LinkedIn's URL shortener.

#### `HTTP: Expand URL`
- Makes an HTTP request with redirects disabled to capture the `Location` header — resolving the full LinkedIn URL.

---

### 📊 Data Scraping Pipeline

#### `Send Process Status`
- Sends the user a loading message: *"⏳ Link received! I am currently fetching the post data..."*
- This message is later **edited in-place** to show progress updates.

#### `Apify: Fetch Post Data`
- Calls the Apify LinkedIn Post Scraper actor (`d0DhjXPjkkwm4W5xK`).
- Extracts the full post text, author info, and post metadata.
- Routes to error output if the scrape fails.

#### `Is Data Empty?`
- Validates that post data was successfully returned before proceeding.
- Halts with an error message if the post is private, deleted, or inaccessible.

#### `Is Author a Company?`
- Checks if the post author has a `company_id` — indicating the post was made by a company page rather than an individual.
- **Company** → skips author scraping, goes directly to company data fetch.
- **Person** → proceeds to fetch the individual author's profile.

#### `Apify: Fetch Author Data`
- Calls the Apify LinkedIn Profile Scraper (`LpVuK3Zozwuipa5bp`).
- Fetches: name, headline, location, education, experience history, and about section.

#### `Is Data Empty?1`
- Validates author data. If empty, sends a notice but continues the flow.

#### `Format Company ID / URL`
- Resolves the company LinkedIn URL from the author's current position or the post's `company_id`.
- Ensures the correct URL is passed to the company scraper.

#### `Apify: Fetch Company Data`
- Calls the Apify LinkedIn Company Scraper (`UwSdACBp7ymaGUJjS`).
- Fetches: name, industry, employee count range, geographic presence, website, and description.

#### `Is Data Empty?2`
- Validates company data. If empty, sends a notice but continues the flow.

#### `Change process Status`
- **Edits** the original loading message to: *"✅ Data collected successfully! Now analyzing..."*

---

### 🤖 AI Analysis Engine

#### `Extract Key Data` *(JavaScript Code)*
- Consolidates data from all three Apify scrapers into a single clean JSON object:
  - `post` — job description text and URL
  - `hr` — name, headline, location, degree, approximate experience years, about
  - `company` — name, industry, locations, size, website, description
  - `language` — user's configured language preference

#### `AI: Generate Report` *(LangChain LLM Chain)*
- Uses a comprehensive system prompt (the **LinkedIn Opportunity Report Agent**) to:
  - Parse the unstructured job description into structured sections
  - Generate a formatted Markdown report with: role summary, responsibilities, requirements, application method, HR profile, company overview, and analytical flags
  - Extract `opportunity_metadata` (role, company, HR name, email) for downstream automation
- **Primary Model:** Google Gemini (`gemini-3.7-flash`) via Vertex AI
- **Fallback Model:** Anthropic Claude Sonnet 5
- Output is parsed via a **Structured Output Parser** into `telegram_report` + `metadata`.

#### `Send Report`
- Delivers the full AI-generated Markdown report to the Telegram user.

#### `Send AI Error`
- If the AI fails, sends: *"❌ The AI encountered an issue while analyzing the data. Please try again."*

---

### 📩 Email Drafting Branch

#### `Is User the Owner`
- Checks if the requesting user matches the configured owner ID.
- Restricts email drafting to the bot owner only.

#### `Check for HR Email`
- Checks if the AI extracted a valid application email from the post metadata.
- Routes to email prompt or "no email found" notice.

#### `Ask to Draft Email`
- Sends an **inline keyboard** to Telegram: *"Do you want us to send you an email?"* with **Yes/No** buttons.
- The hidden metadata (role, company, HR name, email) is encoded in an invisible link within the message for stateless retrieval on callback.

#### `Notify No Contact Email`
- If no application email was found: *"ℹ️ The author did not include an email address for direct application."*

#### `Download file` *(Google Drive)*
- Downloads the user's CV/resume from the configured Google Drive URL.

#### `Create Gmail Draft`
- Creates a personalized Gmail draft with:
  - **Subject:** `Application for {Role} - {Your Name}`
  - **Body:** Professional cover letter addressing the HR/recruiter by name (falls back to `{Company} Team` or `Hiring Team`)
  - **Attachment:** Your CV from Google Drive
  - **To:** The extracted application email

#### `Telegram: Draft Ready Notice`
- Notifies the user: *"✅ Email draft is ready!"* with a direct **link to open the draft in Gmail**.

---

## ⚙️ Configuration

Before activating the workflow, update the following settings:

### 🔧 `Set User Preferences` Node
| Variable | Default | Description |
|---|---|---|
| `lan` | `en` | Response language — `ar` (Arabic) or `en` (English) |
| `user_name` | `Zaid Alradaideh` | Your full name — used in email draft subject and signature |
| `CV url` | *(Google Drive link)* | Direct link to your CV/resume on Google Drive |
| `chat_ids` | `[2076665816, 2068276244]` | Array of Telegram user IDs authorized to use the bot |

### 🔧 `Is User the Owner` Node
- Update the owner check condition to match **your** Telegram user ID:
```
2076665816  →  <your Telegram user ID>
```

### 🔧 `Parse User Input` Node
- The bot username detection is handled via regex. No changes needed unless you modify the LinkedIn URL pattern.

---

## 🔑 Required Credentials

| Credential | Node(s) | Description |
|---|---|---|
| **Telegram Bot API** | All Telegram nodes | API token from [@BotFather](https://t.me/BotFather) |
| **Apify API** | Fetch Post, Fetch Author, Fetch Company | [Apify](https://apify.com/) account API token |
| **Google Service Account** | Google Vertex Chat Model | GCP service account with Vertex AI / Gemini API access |
| **Anthropic API** | Anthropic Chat Model | API key for Claude (fallback LLM) |
| **Gmail OAuth2** | Create Gmail Draft | Google OAuth2 credentials with Gmail scope |
| **Google Drive OAuth2** | Download file | Google OAuth2 credentials with Drive read scope |

> **Tip:** The Apify actors used are paid per run. The fail-fast architecture ensures you're not charged for downstream scrapes if an earlier step fails.

---

## 🚀 Setup Instructions

1. **Import** `workflow.json` into your n8n instance.
2. **Configure credentials:**
   - Add your Telegram Bot API token.
   - Add your Apify API token.
   - Add your Google Cloud Service Account (Vertex AI / Gemini enabled).
   - Add your Anthropic API key (optional fallback).
   - Configure Gmail and Google Drive OAuth2.
3. **Update configuration** in the nodes listed in the [Configuration](#️-configuration) section above.
4. **Upload your CV** to Google Drive and update the `CV url` in `Set User Preferences`.
5. **Activate** the workflow.

---

## 🤖 Bot Usage

| Action | How to Trigger |
|---|---|
| Analyze a LinkedIn post | Send the bot a LinkedIn post URL |
| Analyze a LinkedIn job | Send the bot a LinkedIn job URL (`/jobs/view/...`) |
| Analyze a shortened link | Send the bot an `lnkd.in` short URL |
| Draft an application email | Tap **"Yes"** on the inline keyboard after the report |

### Supported URL Formats
```
https://www.linkedin.com/posts/johndoe_hiring-software-engineer-1234567890-aB-C/
https://www.linkedin.com/jobs/view/1234567890/
https://www.linkedin.com/feed/update/urn:li:activity:1234567890/
https://lnkd.in/abc123
```

---

## 🌐 Language Support

The workflow supports **bilingual** output. Set `lan` in the `Set User Preferences` node:

- `en` → All bot messages and AI reports in **English**
- `ar` → All bot messages and AI reports in **Arabic** (RTL enabled)

The AI report agent also performs **automatic language detection** from the post content as a fallback when the language setting is ambiguous.

---

## 🛠️ Tech Stack

- **n8n** — Workflow automation platform
- **Telegram Bot API** — User interaction, inline keyboards, and message effects
- **Apify** — LinkedIn scraping (Post, Profile, Company actors)
- **Google Gemini** (`gemini-3.7-flash`) via **Vertex AI** — Primary AI analysis engine
- **Anthropic Claude** (`claude-sonnet-5`) — Fallback AI model
- **LangChain (n8n nodes)** — LLM Chain, Structured Output Parser
- **Gmail API** — Automated email draft creation with CV attachment
- **Google Drive API** — CV/resume file download
- **JavaScript** — Custom input parsing, data extraction, and URL metadata encoding

---

## 📂 File Structure

```
JobPost X-Ray in Linkedin/
├── workflow.json   # n8n workflow definition
└── README.md       # This file
```

---

## ⚠️ Notes & Limitations

- **Apify costs** — Each scrape (Post, Author, Company) is a paid Apify actor run. The fail-fast design minimizes wasted credits.
- **Private posts** — LinkedIn posts with restricted visibility cannot be scraped and will return an error.
- **Email safety** — The AI is strictly prohibited from fabricating or guessing email addresses. Only emails explicitly marked as application channels in the post text are extracted.
- **Owner-only email drafting** — The Gmail draft feature is restricted to the configured owner's Telegram ID for security.
- **Rate limits** — Heavy usage may hit Apify or Telegram Bot API rate limits.
- **Stateless callback handling** — Metadata for the email draft flow is encoded in a hidden URL within the Telegram message, making it fully stateless with no database required.
- **Dual LLM fallback** — If the primary Gemini model fails, the Anthropic Claude model is available as a secondary option (requires separate credentials).
