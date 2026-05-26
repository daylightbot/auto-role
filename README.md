# Discord Mass Role Assignment — n8n Workflow

Assigns a single Discord role to a large list of users. Reads User IDs from Google Sheets, assigns the role via the native n8n Discord node, and stamps each row with `done` or `failed` so any interrupted run resumes exactly where it left off.

---

## Workflow (9 nodes + sticky note)

```
[Trigger]
  → Read Config           — reads guild_id and role_id from a single Config row
  → Get Pending Users     — fetches rows where status is blank
  → Outer Loop (50)       — processes users in batches of 50
      → Add Discord Role  — native Discord node, all 50 items in the batch
      → Inner Loop (1)    — feeds results to Sheets one at a time
          → Sheets Delay  — 1-second throttle between writes
          → Update Row Status
          → back to Inner Loop
      ↩ Inner Loop "done" → back to Outer Loop
```

**Why two loops?**
Discord can handle 50 role assignments in a batch quickly. Google Sheets write quota (~60/min) means we must pace each write. The outer loop handles Discord throughput; the inner loop throttles only the Sheets writes.

**Loop output wiring (critical):**

| Node | Output index | Connects to |
|------|-------------|-------------|
| Outer Loop "done" | 0 | *(unconnected)* |
| Outer Loop "loop" | 1 | Add Discord Role |
| Inner Loop "done" | 0 | Outer Loop input |
| Inner Loop "loop" | 1 | Sheets Delay |

---

## Google Sheets setup

### Tab: `Config`

One header row + **one data row**:

| guild_id           | role_id            |
|--------------------|--------------------|
| 123456789012345678 | 987654321098765432 |

> Column names must be exactly `guild_id` and `role_id`.  
> A single data row means the workflow reads exactly one item — no duplication.

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
| `failed` | ❌ API error — see the `error` column |

To retry failed rows: filter the sheet for `status = failed`, clear those status cells, and re-run.

---

## Throughput

- **Discord:** 50 role assignments per outer batch — fast, no enforced delay
- **Sheets writes:** 1 per second (inner loop + 1-second Wait) ≈ 60 rows/minute
- **10,000 users ≈ ~2.5 hours** (background, via Schedule trigger every 10 min)
- **600,000 users ≈ ~7 days** (use the Schedule trigger; workflow resumes from last blank row each run)

If you still see Google Sheets quota errors, increase the Sheets Delay from 1 second to 2 seconds (~30 writes/minute).
