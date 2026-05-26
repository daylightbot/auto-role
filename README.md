# Discord Mass Role Assignment — n8n Workflow

Assigns a single Discord role to a large list of users. Reads User IDs from Google Sheets, assigns the role via the native n8n Discord node, and stamps each row with `done` or `failed` so any interrupted run resumes exactly where it left off.

---

## Workflow (7 nodes)

```
[Trigger]
  → Read Config        — reads guild_id and role_id from a single sheet row
  → Get Pending Users  — fetches rows where status is blank
  → Loop               — processes one user at a time
  → Add Discord Role   — native Discord node, credential attached directly
  → Update Row Status  — writes done/failed back to the sheet
  → back to Loop
```

The "done" output of the Loop node is **left unconnected** — the workflow ends naturally when all pending users are processed.

---

## Google Sheets setup

### Tab: `Config`

One header row + **one data row**:

| guild_id           | role_id            |
|--------------------|--------------------|
| 123456789012345678 | 987654321098765432 |

> Column names must be exactly `guild_id` and `role_id`.  
> A single data row means the workflow reads exactly one item and runs `Get Pending Users` exactly once — no duplication.

### Tab: `Users`

| user_id            | status | timestamp | error |
|--------------------|--------|-----------|-------|
| 111222333444555666 |        |           |       |
| 222333444555666777 |        |           |       |

- Leave `status` **blank** for pending rows.
- Rows already marked `done` or `failed` are skipped automatically on re-runs.
- To retry a failed row, clear its `status` cell.

---

## Setup checklist

1. Replace `YOUR_SPREADSHEET_ID` on **Read Config**, **Get Pending Users**, and **Update Row Status**
2. Set your **Google Sheets OAuth2** credential on those same three nodes
3. Set your **Discord Bot** credential on **Add Discord Role**
4. Make sure the bot has `MANAGE_ROLES` permission and its highest role sits **above** the target role in the server hierarchy

---

## Status values

| Status   | Meaning                              |
|----------|--------------------------------------|
| `done`   | ✅ Role assigned successfully        |
| `failed` | API error — see the `error` column   |

To retry failed rows: filter the sheet for `status = failed`, clear those status cells, and re-run.

---

## Throughput

The native Discord node is rate-limited by Discord's API. For mass assignments:
- ~5 requests/second is safe for most servers
- 10,000 users ≈ ~30 minutes
- 600,000 users — use the Schedule trigger (every 10 min) to run in batches over ~42 hours
