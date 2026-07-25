# 🎙️ Voice Summarization

**النبذة المختصرة:** مسار عمل مؤتمت يستقبل الملاحظات الصوتية من تيليجرام، يحولها إلى نص حرفي باستخدام Gemini، ثم يحلل المحتوى ويصنفه ويعيده كرسالة نصية منظمة.

This n8n workflow automates the transcription and categorization of Telegram voice notes using Google's Gemini models. It handles file extraction, exact text transcription, intelligent categorization, and safe delivery of long messages back to the user.

## ⚙️ Pipeline Architecture

1. **Telegram Trigger:** Listens for incoming messages containing voice notes or audio files.
2. **Get a File & Extract:** Retrieves the audio binary from Telegram utilizing optional chaining (`voice?.file_id || audio?.file_id`).
3. **Gemini Transcript Request:** Sends the binary directly to the Gemini API (`gemini-3.1-pro-preview`) via HTTP POST for strict, verbatim text extraction.
4. **AI Agent (Voice Note Analyst):** Evaluates the transcribed text using Google Vertex AI (`gemini-3.5-flash`). It classifies the intent (Action Item, Note, Announcement, or Casual), generates a summary, and extracts actionable highlights.
5. **Chunking (Code Node):** Splits the AI-generated analysis into 4,000-character segments to strictly adhere to Telegram's message length limitations.
6. **Delivery:** Iterates through the segmented text chunks and delivers the final formatted response back to the original Telegram chat.

## 🔐 Prerequisites & Credentials

To run this workflow, you need the following configured in your n8n instance:
*   **Telegram API:** A valid bot token for the Trigger and Send nodes.
*   **Google Vertex AI:** Service Account credentials linked to the AI Agent chat model.
*   **Google Gemini API:** Standard HTTP Query Auth for the transcription REST API request.

## 🚀 Installation

1. Copy the raw JSON of the workflow.
2. Open your n8n workspace.
3. Click on **Add Workflow** > **Import from file** or simply paste the JSON directly into the editor.
4. Configure and link the required credentials for Telegram and Google.
5. Save and activate the workflow.