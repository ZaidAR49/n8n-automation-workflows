# 📺 YouTube Video Alert Bot

This n8n workflow watches a YouTube channel for new uploads, generates an AI summary of the video using Google's Gemini model, and broadcasts the summary to Discord and Telegram.[cite: 1]

## 🔄 Workflow Steps
1. **YouTube RSS Trigger:** Polls the channel's RSS feed hourly for new videos.[cite: 1]
2. **Format ID (JavaScript):** Strips the `yt:video:` prefix from the raw RSS video ID.[cite: 1]
3. **Get Video Data (YouTube API):** Fetches the video title, description, and metadata.[cite: 1]
4. **Gemini Video Analysis:** Sends an HTTP request to the Gemini API (`gemini-3.1-pro-preview`) to generate an AI summary of the video.[cite: 1]
5. **Message Chunking:** Combines the title, link, description, and AI summary, splitting the resulting text into chunks to safely bypass Discord's 2000-character limit.[cite: 1]
6. **Loop & Broadcast:** Iterates through the message chunks, sending them simultaneously to a Discord channel and a Telegram chat.[cite: 1]

## 🔑 Required Credentials
*   **YouTube OAuth2 API:** For fetching official video metadata.[cite: 1]
*   **Google Service Account / HTTP Query Auth:** To authenticate the Gemini API request.[cite: 1]
*   **Discord OAuth2 API:** To post messages to the target Discord server.[cite: 1]
*   **Telegram API:** To post messages to the target Telegram chat.[cite: 1]

## ⚠️ Notes
*   Discord restricts messages to a 2000-character maximum, while Telegram allows up to 4096.[cite: 1] The JavaScript chunking node specifically caters to the lowest common denominator (Discord) so no text is cut mid-sentence.[cite: 1]