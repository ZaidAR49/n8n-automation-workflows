# 🎮 Discord Free Game Notifier

This n8n workflow automatically fetches active, high-value PC game giveaways from the GamerPower API, ensures they haven't been posted before using Google Sheets, and sends a rich embed alert to a Discord channel.

## 🔄 Workflow Steps
1. **Schedule Trigger:** Runs on a set interval (e.g., daily) to check for new giveaways.
2. **Get Games (HTTP Request):** Fetches active PC game promotions directly from the GamerPower API.
3. **Filter Value (JavaScript):** Removes promotions without end dates and isolates games originally priced over $10.
4. **Get Existing IDs (Google Sheets):** Reads the existing tracker sheet to pull previously posted game IDs.
5. **Deduplicate (JavaScript):** Cross-references the incoming GamerPower IDs against the Google Sheet IDs to return only brand new giveaways.
6. **Check for New Games (If Node):** Routes the workflow to stop if no new valid games are found.
7. **Discord Broadcast:** Sends a cleanly formatted rich embed to Discord, featuring the game art, original price, platform, and expiration date.
8. **Update Tracker (Google Sheets):** Appends the successfully posted game IDs and titles to the sheet to prevent future duplicate alerts.

## 🔑 Required Credentials
*   **Google Sheets OAuth2 API:** To read existing IDs and append new rows to the tracking sheet.
*   **Discord OAuth2 API:** To authenticate and post the rich embeds to your target Discord channel/webhook.

## ⚠️ Notes
*   The workflow relies on the GamerPower API's unique giveaway ID. If a game goes free, expires, and goes free again later, it receives a new ID and will correctly trigger a new alert.
*   Ensure the **Discord** node executes immediately *before* the **Append row in sheet** node. This prevents the Google Sheets success response from overwriting your game data before Discord can format the embed.