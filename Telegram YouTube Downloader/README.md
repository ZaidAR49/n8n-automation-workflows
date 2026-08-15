# 📥 Telegram YouTube Downloader

A conversational n8n Telegram bot that accepts a YouTube link, lets the user pick a quality (1080p, 720p, 480p, or MP3 audio), downloads the media via an **Apify** actor, and delivers the file directly back to the Telegram chat.

---

## 📋 Overview

The **Telegram YouTube Downloader** turns a YouTube URL sent through Telegram into a downloadable media file:

1. **Receive** — The bot listens for incoming messages and callback button taps.
2. **Validate** — A JavaScript code node extracts and validates the YouTube link, normalising all URL formats (regular, `youtu.be`, Shorts, Live).
3. **Route** — A Switch node separates quality-selection callbacks from fresh URL messages and invalid inputs.
4. **Select Quality** — The user is presented with inline keyboard buttons: **1080p**, **720p**, **480p**, and **Audio (MP3)**.
5. **Download** — An Apify YouTube Downloader actor fetches the direct media stream at the chosen quality.
6. **Deliver** — The bot sends the downloaded file (video or audio) back to the Telegram chat.

---

## 🏗️ Architecture

```
Telegram Trigger (message / callback_query)
       │
       ▼
Code in JavaScript  ─── URL extraction & YouTube validation
       │
       ▼
Event Router (Switch)
       │
       ├── [Callback] ──► Edit a text message ("Processing…")
       │                          │
       │                          ▼
       │                    Get Video (Apify)
       │                     │         │
       │                   ✅ OK     ❌ Error
       │                     │         │
       │               HTTP Request   Send Error Message
       │                     │
       │                 Is MP3?
       │                  │      │
       │              Yes │      │ No
       │                  ▼      ▼
       │          Send Audio   Send Video
       │
       ├── [Valid URL] ──► Send Quality Selection Keyboard
       │
       └── [Invalid / Fallback] ──► Send Invalid Link Warning
```

---

## ⚙️ Workflow Nodes

### 1. `Telegram Trigger`
- **Type:** Telegram Trigger
- **Events:** `message`, `callback_query`
- **Purpose:** Entry point. Listens for both plain text messages (containing YouTube links) and inline keyboard button taps (quality selections).

---

### 2. `Code in JavaScript`
- **Type:** Code (JavaScript)
- **Purpose:** Extracts a URL from the incoming message and validates it against a YouTube pattern.
- **Logic:**
  | Step | Detail |
  |---|---|
  | URL extraction | Checks Telegram `entities` for `url` or `text_link` types; falls back to a regex match on raw text |
  | Validation | Matches against `youtube.com/watch?v=`, `youtube.com/embed/`, `youtube.com/shorts/`, `youtube.com/live/`, and `youtu.be/` |
  | Normalisation | Rewrites any matching URL to the canonical `https://www.youtube.com/watch?v=<ID>` form |
- **Output fields:** `isValid` (boolean), `youtubeUrl` (string or null)

---

### 3. `Event Router` (Switch)
- **Type:** Switch
- **Purpose:** Directs execution to the correct branch based on event type and URL validity.
- **Outputs:**

| Output | Condition | Destination |
|---|---|---|
| `Callback` | `callback_query.data` is not empty | Edit message → Apify download flow |
| `On Valid URL` | `isValid` is `true` | Quality selection keyboard |
| *(fallback)* | Everything else | Invalid link warning |

---

### Branch A — Callback (Quality Selected)

#### 4. `Edit a text message`
- **Type:** Telegram (editMessageText)
- **Purpose:** Replaces the quality-selection message with a "⏳ Processing…" status so the user knows the download has started.

#### 5. `Get Video` (Apify)
- **Type:** Apify (`UUhJDfKJT2SsXdclR` actor — YouTube Downloader)
- **Purpose:** Runs the Apify YouTube Downloader actor with the chosen quality and returns a direct download URL.
- **Configuration:**
  - `preferredFormat`: `mp4`
  - `preferredQuality`: value from the callback button (`1080p`, `720p`, `480p`, or `144p` for audio)
  - `storeInKVStore`: `false`
  - `videos[].url`: original YouTube URL embedded in the rich message
- **Error Handling:** On failure, routes to `Send a rich message2` (error alert).

#### 6. `HTTP Request`
- **Type:** HTTP Request
- **Purpose:** Downloads the media file from the Apify-generated direct URL as binary data, ready to be forwarded to Telegram.

#### 7. `Is MP3`
- **Type:** If
- **Purpose:** Checks whether the user selected the `144p` (Audio/MP3) option. Routes to `Send an audio file` on true, `Send a video` on false.

#### 8a. `Send an audio file`
- **Type:** Telegram (sendAudio)
- **Purpose:** Delivers the downloaded audio file to the user's chat with a success caption (Arabic).

#### 8b. `Send a video`
- **Type:** Telegram (sendVideo)
- **Purpose:** Delivers the downloaded video file to the user's chat with a success caption (Arabic).

#### 9. `Send a rich message2` *(Error handler)*
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that the download failed and asks them to try again later.

---

### Branch B — Valid URL Received

#### 10. `Send a rich message1` *(Quality Selection)*
- **Type:** Telegram (sendRichMessage + inlineKeyboard)
- **Purpose:** Confirms receipt of the YouTube link (in Arabic) and presents an inline keyboard with four quality/format options.

| Button | Callback Data |
|---|---|
| 1080p | `1080p` |
| 720p | `720p` |
| 480p | `480p` |
| Audio (MP3) | `144p` |

---

### Branch C — Invalid / Unknown Input

#### 11. `Send a rich message` *(Invalid Link Warning)*
- **Type:** Telegram (sendRichMessage, Markdown)
- **Purpose:** Notifies the user (in Arabic) that the link is invalid and provides accepted URL format examples.

---

## 🔗 Dependencies & Credentials

| Service | Credential Name | Purpose |
|---|---|---|
| Telegram | `Telegram account` | Bot trigger + all message sends |
| Apify | `Apify account` | YouTube Downloader actor `UUhJDfKJT2SsXdclR` |

---

## 🚀 Setup

1. **Import** `workflow.json` into your n8n instance.
2. **Configure credentials:**
   - Connect a Telegram Bot token under `Telegram account`.
   - Add your Apify API key under `Apify account`.
3. **Activate** the workflow — the Telegram webhook registers automatically.
4. **Send** any YouTube link to your Telegram bot to begin.

---

## 💬 Usage

| User Action | Bot Response |
|---|---|
| Send a YouTube URL | Validates link → Shows quality selection keyboard |
| Tap a quality button (1080p / 720p / 480p) | Downloads video → Sends `.mp4` file |
| Tap **Audio (MP3)** | Downloads audio → Sends audio file |
| Send an invalid or non-YouTube URL | Replies with an invalid-link warning and format examples |

### Accepted URL Formats

```
https://www.youtube.com/watch?v=VIDEO_ID
https://youtu.be/VIDEO_ID
https://www.youtube.com/shorts/VIDEO_ID
https://www.youtube.com/live/VIDEO_ID
https://www.youtube.com/embed/VIDEO_ID
```

---

## ⚠️ Notes & Limitations

- **File size limits** — Telegram bots can send files up to **50 MB** via the Bot API. Very long videos at high resolutions may exceed this limit.
- **Processing time** — The Apify actor may take up to a few minutes depending on video length and server load. The "Processing…" message is shown during this wait.
- **Quality availability** — Not all qualities are available for every video. The Apify actor will fall back to the closest available quality.
- **Language** — Status and error messages sent to the user are in **Arabic**. The workflow logic itself is language-agnostic.
- **Audio format** — Selecting **Audio (MP3)** maps internally to quality `144p`, which signals the actor to extract audio only.
- **Shorts & Live** — YouTube Shorts and Live URLs are fully supported and normalised to the standard `watch?v=` format before download.
