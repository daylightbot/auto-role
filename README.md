# Discord Mass Role Assignment — n8n Workflow

Assigns a single Discord role to a large list of users, tracking progress in Google Sheets so any failure can be resumed exactly where it left off — no re-processing from the beginning.

---

## How it works

```
[Manual / Schedule Trigger]
        │
        ▼
[Read Config Sheet]  ← guild_id, role_id, bot_token
        │
        ▼
[Extract Config]
        │
        ▼
[Get Pending Users]  ← only rows where status = blank
        │
        ▼
[Loop — 1 user at a time] ◄──────────────────────┐
        │ (output 0: current user)                 │
        ▼                                          │
[Assign Discord Role]  ← PUT /guilds/.../roles/... │
        │                                          │
        ▼                                          │
[Update Row Status]  → writes done/failed/etc      │
        │                                          │
        └──────────────────────────────────────────┘
        (loops until all pending rows processed)

[output 1: all done → workflow ends cleanly]
```

**Resume logic:** Every run reads only rows with a blank `status` column. If the workflow crashes at row 47,832, the next run skips rows 1–47,831 (already `done`) and picks up at 47,832 automatically.

---

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from file** → select `discord-mass-role-assignment.json`.

### 2. Create your Google Spreadsheet

You need one spreadsheet with **two tabs**:

#### Tab: `Config`

| setting   | value                  |
|-----------|------------------------|
| guild_id  | `123456789012345678`   |
| role_id   | `987654321098765432`   |
| bot_token | `Bot MTIzNDU2...`      |

> ⚠️ Include the `Bot ` prefix in the token value.  
> The Role ID and Guild ID are 18-digit numbers found in Discord (enable Developer Mode → right-click the server/role → Copy ID).

#### Tab: `Users`

| user_id            | status | timestamp | error |
|--------------------|--------|-----------|-------|
| 111222333444555666 |        |           |       |
| 222333444555666777 |        |           |       |
| ...                |        |           |       |

- `status` starts **blank** for every user. The workflow fills it in.
- You can add as many rows as you need (tested to 600k+).
- To retry a failed user, clear their `status` cell and re-run.

### 3. Configure the three Google Sheets nodes

In n8n, open each of these nodes and update the **Spreadsheet ID**:

- `📋 Read Config Sheet`
- `👥 Get Pending Users`
- `✅ Update Row Status`

The spreadsheet ID is in the Google Sheets URL:
```
https://docs.google.com/spreadsheets/d/THIS_IS_YOUR_ID/edit
```

### 4. Connect your Google Sheets credential

In n8n, go to **Credentials → New → Google Sheets OAuth2**, follow the auth flow, then select it in all three Google Sheets nodes.

### 5. Set your Discord bot up

Your bot needs:
- `MANAGE_ROLES` permission in the server
- Its **highest role must be above the role you're assigning** in the server's role hierarchy

### 6. Activate

- For a **one-off run**: use the `▶ Run Manually` trigger
- For **automated batches**: enable the `⏰ Schedule (every 10 min)` trigger (disable the manual one)

---

## Status values

| Status          | Meaning                                                    |
|-----------------|------------------------------------------------------------|
| `done`          | ✅ Role assigned successfully                              |
| `not_in_server` | User has left the server. Row is skipped on future runs.  |
| `rate_limited`  | Hit Discord's rate limit — will auto-retry next run       |
| `failed`        | API error — check the `error` column for details          |

---

## Throughput & time estimates

The workflow processes ~4 users/second (250 ms gap between successful calls, staying well under Discord's rate limits).

| Users    | Time per run (1,000-row batch) | Total time       |
|----------|-------------------------------|------------------|
| 10,000   | ~2.5 min                      | ~42 min          |
| 100,000  | ~2.5 min/batch × 100 runs     | ~7 hours         |
| 600,000  | ~2.5 min/batch × 600 runs     | ~42 hours        |

> **Want faster?** Create two copies of the workflow with separate Users sheets (split your list in half). Run them simultaneously with different bot tokens. Two bots = 2× speed, etc.

---

## Frequently asked questions

**Can I change the Role ID mid-run?**  
Yes — update the `role_id` cell in the Config sheet. The next run will pick up the new value. Already-assigned users are unaffected (their rows are `done`).

**What if a user gets the wrong role?**  
Discord role assignments via this workflow use `PUT` which only adds the role — it doesn't touch any other roles the user has.

**What if my n8n execution times out?**  
Each run picks up from the first blank-status row. Just re-trigger and it continues where it left off. If you hit n8n's execution timeout for a single run, lower the per-run batch by adding a row count limit to the `Get Pending Users` node (Options → Limit).

**How do I handle the bot_token securely?**  
For production, store the token in an n8n **Credential** (type: Header Auth) and reference it from the Code node via `$credentials.yourCredentialName.value` instead of reading it from the sheet.

---

## Files in this repo

| File | Description |
|------|-------------|
| `discord-mass-role-assignment.json` | The importable n8n workflow |
| `README.md` | This document |
