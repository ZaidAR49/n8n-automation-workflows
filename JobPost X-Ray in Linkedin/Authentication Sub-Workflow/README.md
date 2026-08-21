# 🔐 Authentication Sub-Workflow (Gatekeeper)

> A reusable n8n sub-workflow that acts as a **security gatekeeper** for Telegram bots. It validates a user's identity and permissions against a Supabase database before allowing access to any protected workflow — returning a structured authorization response in both Arabic and English.

---

## 📋 Overview

This is a **microservice-style sub-workflow** designed to be called from any parent n8n workflow. It receives a Telegram user's data along with the calling workflow's context, queries a Supabase `users` table, and returns a clean JSON response indicating whether the user is authorized.

The workflow handles **6 distinct authorization states**:

| State | Condition | `is_authorized` |
|---|---|---|
| ✅ **Authorized** | `status = approved` AND `is_locked = false` AND workflow in `allowed_workflows` | `true` |
| 🚫 **Workflow Not Allowed** | `status = approved` AND `is_locked = false` BUT workflow NOT in `allowed_workflows` | `false` |
| 🔒 **Locked** | `is_locked = true` (regardless of status) | `false` |
| ⏳ **Pending** | `status = pending` AND `is_locked = false` | `false` |
| ❌ **Rejected** | `status = rejected` AND `is_locked = false` | `false` |
| ⚠️ **Unregistered** | User not found in the database (fallback) | `false` |

---

## 🔄 Workflow Diagram

```
Execute Workflow Trigger (called by parent)
        │
        ▼
Set User Preferences (lan, bot_username)
        │
        ▼
Get a row (Supabase: query users by telegram_id)
        │
        ▼
Switch ──► Authorized          ──► Set Authorized Response
       ├──► WorkflowNotAllowed ──► Set Workflow Not Allowed Response
       ├──► Locked             ──► Set Locked Response
       ├──► Pending            ──► Set Pending Response
       ├──► Rejected           ──► Set Rejected Response
       └──► extra (fallback)   ──► Set Fallback Response
                                           │ (all branches return to parent)
```

---

## ✨ Features

| Feature | Details |
|---|---|
| 🔌 **Plug-and-Play** | Callable from any n8n workflow via the `Execute Workflow` node |
| 🛡️ **6-State Authorization** | Covers every possible user state: authorized, workflow-blocked, locked, pending, rejected, and unregistered |
| 🌐 **Bilingual Responses** | All messages support Arabic (`ar`) and English (`en`) via a configurable `lan` variable |
| 📋 **Workflow-Scoped Access** | Uses an `allowed_workflows` array per user for fine-grained, per-bot permission control |
| 📦 **Structured Output** | Returns a consistent JSON object (`is_authorized`, `role`, `message`) that the parent workflow can use directly |

---

## 🧩 Nodes Breakdown

### Sub-Workflow Core

| Node | Type | Purpose |
|---|---|---|
| **Execute Workflow Trigger** | Execute Workflow Trigger | Entry point; receives `Telegram` payload and `context` (workflow ID) from the parent |
| **Set User Preferences** | Set | Configures `lan` (language) and `bot_username` (your onboarding bot handle) |
| **Get a row** | Supabase | Queries the `users` table by `telegram_id` from the incoming Telegram payload |
| **Switch** | Switch | Routes to one of six branches based on `status`, `is_locked`, and `allowed_workflows` |

### Response Branches

| Node | Output Condition | `is_authorized` |
|---|---|---|
| **Set Authorized Response** | Approved, unlocked, workflow in list | `true` + user's `role` and `is_locked` |
| **Set Workflow Not Allowed Response** | Approved, unlocked, workflow NOT in list | `false` |
| **Set Locked Response** | Account is locked | `false` |
| **Set Pending Response** | Account is pending approval | `false` |
| **Set Rejected Response** | Account has been rejected | `false` |
| **Set Fallback Response** | User not found in DB (no row returned) | `false` |

### Parent Workflow Mockup (included for reference)

| Node | Type | Purpose |
|---|---|---|
| **Telegram Trigger** | Telegram Trigger | Receives messages in the parent workflow |
| **Edit Fields1** | Set | Packages the Telegram payload and `$workflow` context to pass to the sub-workflow |
| **Execute Workflow** | Execute Workflow | Calls this sub-workflow synchronously and waits for the result |
| **If** | IF | Checks `is_authorized`; routes to error message or main bot logic |
| **Welcome message** | Telegram | Sends the authorized welcome message to the user |
| **Send Error** | Telegram | Sends the denial/status message when `is_authorized = false` |
| **Replace Me** | No-Op | Placeholder — delete this and build your actual bot logic here |

---

## 🛠️ Prerequisites & Setup

### Required Credentials

| Service | Credential Type | Where Used |
|---|---|---|
| **Supabase** | `supabaseApi` | `Get a row` node |
| **Telegram Bot** | `telegramApi` | `Telegram Trigger`, `Send Error`, `Welcome message` nodes (parent mockup) |

### Step-by-Step Setup

#### 1. Supabase — Create the `users` Table

Run the following SQL in your Supabase SQL Editor to create the required schema:

```sql
-- 1. Create ENUMs for strict control
CREATE TYPE access_status AS ENUM ('pending', 'approved', 'rejected');
CREATE TYPE user_role AS ENUM ('admin', 'viewer', 'client');

-- 2. Create the primary users table
CREATE TABLE users (
  telegram_id        BIGINT PRIMARY KEY,
  username           TEXT UNIQUE,
  full_name          TEXT NOT NULL,
  reason_for_access  TEXT,
  status             access_status DEFAULT 'pending',
  allowed_workflows  TEXT[] DEFAULT '{}',
  metadata           JSONB DEFAULT '{}'::jsonb,
  is_locked          BOOLEAN DEFAULT FALSE,
  role               user_role DEFAULT 'client',
  created_at         TIMESTAMPTZ DEFAULT NOW(),
  updated_at         TIMESTAMPTZ DEFAULT NOW()
);
```

#### 2. Configure the Sub-Workflow

1. Open the **Set User Preferences** node and update:
   - `lan` → `ar` for Arabic output, `en` for English output
   - `bot_username` → your Telegram onboarding/auth bot handle (e.g., `@MyAuthBot`)

2. In the **Get a row** (Supabase) node:
   - Link your Supabase credential.
   - Confirm the table name is `users` and the filter key is `telegram_id`.

#### 3. Grant Users Access to a Workflow

To authorize a user for a specific bot/workflow, add that workflow's n8n **Workflow ID** to their `allowed_workflows` array in Supabase:

```sql
UPDATE users
SET allowed_workflows = array_append(allowed_workflows, 'YOUR_WORKFLOW_ID')
WHERE telegram_id = 123456789;
```

> The workflow ID is visible in n8n under **Workflow Settings** or from the URL: `https://your-n8n.com/workflow/YOUR_WORKFLOW_ID`.

#### 4. Integrate Into a Parent Workflow

In your parent workflow, before any protected logic:

1. Add a **Set** node to map the Telegram data and workflow context:
   ```
   Telegram  →  {{ $json.toJsonString() }}
   context   →  {{ $workflow }}
   ```

2. Add an **Execute Workflow** node pointing to this sub-workflow with `Wait for sub-workflow` enabled.

3. Add an **IF** node checking `{{ $json.is_authorized }} == true`.

4. On the `false` branch, send `{{ $json.message }}` back to the Telegram user.

5. On the `true` branch, build your actual bot logic (delete the **Replace Me** placeholder).

---

## 📤 Output Schema

The sub-workflow always returns a single JSON object. The parent workflow can use it directly.

```json
// When authorized:
{
  "is_authorized": true,
  "role": "client",
  "is_locked": false,
  "message": "✅ Welcome back, John! You are authorized. How can I help you today?"
}
```

```json
// When not authorized (any denial state):
{
  "is_authorized": false,
  "message": "🔒 Account Locked. Your account has been suspended by the admin.\n\nTo resolve this, please contact @YourAuthBotUsername"
}
```

---

## 💡 Design Decisions

### Why a Sub-Workflow?
Packaging authentication as a standalone sub-workflow means a single update propagates to every bot that calls it — no duplicating logic across workflows.

### Workflow-Scoped Permissions
The `allowed_workflows TEXT[]` column lets you grant or revoke access to individual bots per user without needing separate user tables for each workflow.

### Fallback = Unregistered
The Switch node's `extra` (fallback) output fires when **no database row is found** for the Telegram ID. This cleanly handles completely new or unregistered users without a separate existence check.

### Bilingual at the Source
All response messages are resolved inside the response-setter nodes using a ternary on `lan`. The parent workflow receives a pre-formatted string — it never needs to know the user's language.

---

## ⚠️ Limitations & Notes

- **Language is global:** The `lan` setting in **Set User Preferences** applies to all users. To support per-user language preferences, store a `language` column in the `users` table and read it after the Supabase query.
- **No self-registration:** This workflow only validates — it does not create new users. Implement a separate registration flow to insert rows into the `users` table.
- **`allowed_workflows` is an exact match:** The workflow ID must exactly match the value in the array. Ensure the parent workflow passes `$workflow.id` (not the name).
- **Single Supabase credential:** All users are validated against one shared database via a single credential.

---

## 🔗 Related

- [Parent Workflow README](../README.md)
- [n8n Execute Workflow Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executeworkflow/)
- [Supabase n8n Integration](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.supabase/)
- [Telegram Bot API](https://core.telegram.org/bots/api)
