# ✍️ Slack to Post Autopilot
This workflow automates the process of transforming Slack messages with images into structured posts in a database. It validates messages, uploads images, generates content using AI, and stores the outputs efficiently.

---

## Table of Contents
- Overview
- How It Works
- Node Architecture
- Tech Stack
- Prerequisites
- Configuration
- Output
- Edge Cases & Fallbacks

---

## Overview
Triggered by a Slack message, this workflow checks for meaningful text and images, uploads the images, and generates bilingual content for database storage. It ensures that all inputs are valid and automatically manages error responses.

---

## How It Works
[Connection data unavailable — diagram cannot be generated]
1. A Slack message triggers the workflow, initiating the process.
2. The provided text is validated for length, and image attachments are checked.
3. If validations pass, images are downloaded and uploaded to Cloudinary.
4. The article text and image URLs are sent to the OpenAI model to generate formatted content.
5. The resulting posts are stored in a Supabase database along with translations.
6. Success or error messages are communicated back to the Slack channel.

---

## Node Architecture
| Node Name                                    | Type                             | Role                                              |
|----------------------------------------------|----------------------------------|---------------------------------------------------|
| Slack Trigger                                | n8n-nodes-base.slackTrigger     | Triggers workflow on Slack message or file share  |
| HTTP Request                                 | n8n-nodes-base.httpRequest      | Downloads image files from Slack                  |
| Upload an asset from file data               | n8n-nodes-cloudinary.cloudinary | Uploads downloaded images to Cloudinary           |
| Code in JavaScript                           | n8n-nodes-base.code             | Validates input from Slack                         |
| Split Out                                    | n8n-nodes-base.splitOut         | Splits output data into separate flows             |
| Failed message                               | n8n-nodes-base.slack            | Sends error messages to Slack                      |
| Build user prompt                            | n8n-nodes-base.code             | Constructs prompt for AI model                     |
| Success message                              | n8n-nodes-base.slack            | Sends success messages to Slack                    |
| If images or text are missing                | n8n-nodes-base.if               | Conditional routing based on validation outcomes    |
| If uploading images failed                   | n8n-nodes-base.if               | Checks for image upload success                     |
| Message a model                              | @n8n/n8n-nodes-langchain.openAi | Interacts with OpenAI model for content generation  |
| create post                                  | n8n-nodes-base.supabase         | Stores posts in the database                       |
| create translation post in English           | n8n-nodes-base.supabase         | Creates the English translation of the post        |
| create translation post in Arabic            | n8n-nodes-base.supabase         | Creates the Arabic translation of the post         |
| Sticky Note                                  | n8n-nodes-base.stickyNote       | Displays notes for documentation                   |

---

## Tech Stack
| Component                  | Technology                |
|----------------------------|---------------------------|
| Slack                       | n8n-nodes-base.slack      |
| HTTP Request                | n8n-nodes-base.httpRequest |
| Cloudinary                  | n8n-nodes-cloudinary.cloudinary |
| JavaScript                  | n8n-nodes-base.code       |
| Supabase                    | n8n-nodes-base.supabase   |
| OpenAI                      | @n8n/n8n-nodes-langchain.openAi |

---

## Prerequisites
- n8n instance v1.0+
- Slack API key required
- Cloudinary API key required
- OpenAI API key required
- Supabase API key required

---

## Configuration
Adjust the channel ID for Slack triggers and ensure API keys for the Integrated services are set correctly in n8n's credentials.

---

## Output
The workflow produces database entries for posts and their translations, along with success/failure messages sent to the originating Slack channel.

---

## Edge Cases & Fallbacks
| Scenario                                      | Behavior                                                          |
|-----------------------------------------------|-------------------------------------------------------------------|
| Missing required fields                       | Sends a failure message to Slack with the reason                  |
| Image upload failure                          | Notifies Slack of the upload error                                 |
| Invalid JSON structure                        | Fails to execute, prompts error message in Slack                  |