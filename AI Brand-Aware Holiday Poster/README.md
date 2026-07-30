# 🎉 AI Brand-Aware Holiday Poster

An n8n workflow that **automatically detects upcoming cultural, national, and religious events** from Google Calendar and generates a brand-aligned LinkedIn post — complete with an AI-generated image — then routes it through a human approval gate before publishing.

---

## 🔄 Workflow Overview

```
Schedule Trigger → Google Calendar → AI Post Agent → Gemini Image Generation
     → Google Drive (Upload + Share) → Approval Email
          ├── ✅ Approve    → Publish to LinkedIn → Confirmation Email → Cleanup
          ├── 🔁 Regenerate → Loop back to Calendar fetch
          └── ❌ Reject     → Cleanup & Stop
```

---

## 🧩 Workflow Steps

### 1. Schedule Trigger
Fires on a configurable interval (default: every hour) to kick off the pipeline.

### 2. Get Many Events (Google Calendar)
Fetches the **next upcoming event** from a designated Google Calendar, ordered by start time. This calendar should contain national days, religious observances, and cultural occasions.

### 3. Post Agent (AI Agent — Google Vertex / Gemini)
The core intelligence of the workflow. Using the event details and the company's brand profile fetched live from Google Sheets, this agent:
- Classifies the event (national/patriotic, religious, cultural, or not applicable).
- Drafts an **Arabic LinkedIn caption** (800–1,500 chars, max 3,000) with 1–2 category-appropriate emojis and the brand's default hashtags.
- Writes an **English image-generation prompt** where the event's own symbols (flag, crescent+mosque, etc.) are the dominant subject (~80%) and the brand's visual style is a subtle accent (~20%).
- Reproduces the brand's contact `email` exactly for downstream routing.
- Returns a structured JSON object; gracefully handles missing data with error responses instead of hallucinating.

**Model:** `gemini-3.5-flash` via Google Vertex AI

### 4. Generate Image (HTTP Request → Gemini Imagen)
Sends the `image_prompt` from the agent to the **Gemini 2.5 Flash Image** generation endpoint (`gemini-2.5-flash-image:generateContent`). Retries automatically on failure (5-second delay between attempts).

### 5. Extract Image
Pulls the base64 image data from the API response (`candidates[0].content.parts[0].inlineData.data`).

### 6. Convert to File
Converts the base64 image string into a binary file for upload.

### 7. Upload File (Google Drive)
Uploads the binary image to a designated **"Event images"** folder in Google Drive.

### 8. Share File (Google Drive)
Sets the uploaded file's permissions to **"anyone with the link can view"** so it can be previewed in the approval email.

### 9. Send Message & Wait for Response (Gmail — Approval Gate)
Sends an **interactive HTML email** to the brand contact's email address containing:
- The full Arabic post text (in an RTL-styled blockquote).
- A live inline preview of the generated image (via Google Drive public link).
- A **dropdown form** with three choices:

| Choice (Arabic) | Action |
|---|---|
| `نشر` (Publish) | Proceeds to LinkedIn publishing |
| `اعادة التوليد` (Regenerate) | Loops back to fetch events and generate a new post |
| `رفض` (Reject) | Terminates the workflow and cleans up |

### 10. Switch (Router)
Routes execution based on the approval form response to one of the three branches above.

### 11. Create a Post (LinkedIn)
Publishes the post **as an organization** to LinkedIn with the generated image attached.

### 12. Result Email (Gmail)
Sends a confirmation email to the brand contact with a **direct link to the live LinkedIn post**.

### 13. Delete a File (Google Drive)
Cleans up the temporary image from Google Drive after publishing or rejection to avoid storage bloat.

---

## 🔑 Required Credentials

| Credential | Used By |
|---|---|
| **Google Service Account** (Vertex AI) | Post Agent (Gemini chat model) |
| **HTTP Query Auth** | Generate Image (Gemini Imagen API key) |
| **Google Sheets OAuth2** | Get row(s) from company brand sheet |
| **Google Calendar OAuth2** | Get many events |
| **Google Drive OAuth2** | Upload, Share, and Delete image files |
| **Gmail OAuth2** | Approval email + confirmation email |
| **LinkedIn OAuth2** | Create a post (as organization) |

---

## 📊 Google Sheets — Brand Profile Schema

The workflow reads company brand data from a Google Sheet (`Company data`, Sheet1). Each row must contain the following columns:

| Column | Required | Description |
|---|---|---|
| `company_name` | ✅ | Display name of the company |
| `logo_url` | ⬜ | Not used by this workflow |
| `brand_colors` | ✅ | Comma-separated hex codes (e.g. `#0A1628,#00C6FF`) |
| `visual_style` | ✅ | Descriptive phrase (e.g. `sleek dark-mode 3D render`) |
| `industry_niche` | ✅ | Company's domain (e.g. `cloud infrastructure`) |
| `brand_voice` | ✅ | Tone descriptor (e.g. `professional, inspiring`) |
| `default_hashtags` | ✅ | Space-separated tags appended to every post |
| `email` | ✅ | Contact email for approval routing and notifications |

---

## ⚙️ Setup Instructions

1. **Google Calendar** — Create a calendar dedicated to national/religious/cultural events and add your events there. Point the `Get many events` node to its Calendar ID.
2. **Google Sheets** — Create the `Company data` sheet with the columns above and fill in at least one brand profile row. Update the Sheet ID in the `Get row(s) in sheet` tool node.
3. **Google Drive** — Create a folder called `Event images` (or rename it) and update the Folder ID in the `Upload file` node.
4. **LinkedIn** — Ensure the LinkedIn OAuth2 credential has `w_organization_social` scope and update the `organization` ID in the `Create a post` node to your LinkedIn Organization URN.
5. **Gemini Imagen API** — Add your Google API key as an HTTP Query Auth credential and link it to the `Generate image` node.
6. **Activate the workflow** — Enable the Schedule Trigger and set your desired polling interval.

---

## ⚠️ Notes

- **Human-in-the-loop by design.** The workflow will pause indefinitely at the approval gate until the reviewer responds. n8n's execution must remain active.
- **Regeneration loop.** Choosing "Regenerate" routes back to the `Get many events` node, re-fetching the same next event and producing a fresh post and image.
- **Arabic-first content.** `post_text` is always in Modern Standard Arabic (except hashtags and emojis). The approval email is also displayed RTL.
- **Event classification.** If the AI agent cannot determine a cultural/national/religious significance from the event's name and description alone, it returns an error (`event_not_applicable`) and the workflow ends gracefully without publishing.
- **Image cleanup.** Regardless of the reviewer's decision (publish or reject), the temporary Google Drive image is deleted after the workflow branch completes.
- **LinkedIn character limit.** The agent enforces LinkedIn's 3,000-character post limit and writes the hook within the first 210 characters (desktop "see more" threshold).
