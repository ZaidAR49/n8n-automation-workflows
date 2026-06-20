# 🎤 Interview Preparation Assistant
This workflow helps users prepare for interviews by scraping LinkedIn profiles, extracting CV text, and generating a personalized interview prep report. It sends the report directly via email, ensuring comprehensive preparation for candidates.

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
The workflow is triggered by a user submitting a form which includes their LinkedIn profile data, optional interviewer and company LinkedIn profiles, and an optional CV file upload. It checks for necessary information, scrapes LinkedIn profiles, and consolidates all the data to create a comprehensive interview prep report.

---

## How It Works
[Connection data unavailable — diagram cannot be generated]
Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 6 → Step 7 → Step 8 

1. The workflow starts with a form submission, collecting user information including LinkedIn profiles and CV uploads.  
2. It validates the presence of at least one LinkedIn profile.  
3. If valid, it scrapes the user's LinkedIn profile data.  
4. It checks if the interviewer or company LinkedIn profiles are provided and scrapes them if they exist.  
5. Once all data is collected, it consolidates the information for report generation.  
6. The AI Interview Coach generates a structured HTML report tailored for the user.  
7. Finally, the report is sent to the user via email.  
8. If any required data is missing, an error email is dispatched instead.

---

## Node Architecture
| Node Name | Type | Role |
|---|---|---|
| Form Trigger | n8n-nodes-base.formTrigger | Collects user input from the form submission |
| Validate: Need At Least One | n8n-nodes-base.if | Validates presence of LinkedIn URLs |
| Send Error Email | n8n-nodes-base.gmail | Sends error emails if validation fails |
| Run: Scrape User LinkedIn | @apify/n8n-nodes-apify.apify | Scrapes user LinkedIn data |
| Has Interviewer LinkedIn? | n8n-nodes-base.if | Checks for the Interviewer LinkedIn presence |
| Run: Scrape Interviewer LinkedIn | @apify/n8n-nodes-apify.apify | Scrapes interviewer LinkedIn data |
| Has Company LinkedIn? | n8n-nodes-base.if | Checks for the Company LinkedIn presence |
| Run: Scrape Company LinkedIn | @apify/n8n-nodes-apify.apify | Scrapes company LinkedIn data |
| Consolidate All Data | n8n-nodes-base.code | Merges all collected data for report generation |
| AI Interview Coach | @n8n/n8n-nodes-langchain.agent | Generates personalized interview report |
| Send Interview Report | n8n-nodes-base.gmail | Sends the prepared report via email |
| Extract from File | n8n-nodes-base.extractFromFile | Extracts text from uploaded CVs |

---

## Tech Stack
| Component | Technology |
|---|---|
| Form Trigger | n8n-nodes-base.formTrigger |
| Validation | n8n-nodes-base.if |
| Email Notification | n8n-nodes-base.gmail |
| LinkedIn Scraping | @apify/n8n-nodes-apify.apify |
| Consolidation | n8n-nodes-base.code |
| AI Generation | @n8n/n8n-nodes-langchain.agent |
| File Extraction | n8n-nodes-base.extractFromFile |

---

## Prerequisites
- n8n instance v1.0+
- Apify OAuth2 API key required
- Gmail OAuth2 API key required

---

## Configuration
Users must ensure that their LinkedIn URLs and CVs are correctly inputted in the form submission. Proper API keys must be configured for both Apify and Gmail integrations.

---

## Output
Upon successful execution, the workflow produces a personalized interview preparation report as an HTML document emailed to the user. The report includes detailed analyses from the scraped LinkedIn data and the uploaded CV.

---

## Edge Cases & Fallbacks
| Scenario | Behavior |
|---|---|
| Missing required form fields | Sends an error email to the user informing them of the missing information |
| Malformed workflow JSON input | _[Error: workflow JSON contains no nodes. README cannot be generated.]_ |
| No LinkedIn profiles provided | Sends an error email due to validation failure |
| Missing CV file | Report generation proceeds without CV content but denotes the absence in the output |
