# Discord Mass Role Assignment — n8n Workflow

Assigns a single Discord role to a large list of users. Reads User IDs from Google Sheets in batches, assigns the role via the native n8n Discord node, and records progress in a single checkpoint value — so any interrupted run resumes exactly where it left off.

---

## Workflow (6 nodes + sticky note)

```
[Trigger]
  → Read Config          — reads guild_id, role_id, checkpoint from Config tab
  → Read Next Batch      — reads up to 300 users starting at checkpoint offset
  → Add Discord Role     — assigns role to all items, continues past individual errors
  → Compute Checkpoint   — counts processed items, collapses to 1 output item
  → Advance Checkpoint   — writes new checkpoint to Config tab (1 write per batch)
```

When Read Next Batch returns 0 rows (all users done), the workflow stops naturally — the downstream nodes never execute.

---

## How resume works

The `checkpoint` column in the Config tab tracks how many users have been processed. Each run:

1. Reads `checkpoint` (e.g. `600`)
2. Reads rows `602–901` of the Users tab (next 300)
3. Assigns the role to all 300
4. Writes `checkpoint = 900` back to Config
5. Next run picks up from row `902`

To restart from scratch: set `checkpoint` back to `0`.  
To resume from a specific point: set `checkpoint` to the number of users already processed.

---

## Google Sheets setup

### Tab: `Config`

One header row + **one data row**:

| guild_id           | role_id            | checkpoint |
|--------------------|--------------------|------------|
| 123456789012345678 | 987654321098765432 | 0          |

> Column names must be exactly `guild_id`, `role_id`, `checkpoint`.

### Tab: `Users`

One column only — no status tracking per row needed:

| user_id            |
|--------------------|
| 111222333444555666 |
| 222333444555666777 |

Row 1 = header. User IDs start at row 2.

---

## Setup checklist

1. Replace `YOUR_SPREADSHEET_ID` on **Read Config**, **Read Next Batch**, and **Advance Checkpoint**
2. Set your **Google Sheets OAuth2** credential on those same three nodes
3. Set your **Discord Bot** credential on **Add Discord Role**
4. Ensure the bot has `MANAGE_ROLES` permission and its highest role sits **above** the target role in the server hierarchy
5. Set `checkpoint` to `0` in the Config tab
6. In Workflow Settings → set **Max concurrent executions = 1** to prevent two scheduled runs from overlapping and processing the same batch twice

---

## Error handling

Individual Discord errors (user not in server, permission denied) do not stop the workflow — `continueOnFail` is enabled on the Discord node. Those users are skipped and the checkpoint advances past them.

If you need to identify which users failed, enable **Save Manual Executions** (already on) and inspect the execution log — failed items show in the Discord node's output with the error detail.

---

## Throughput

| Metric | Value |
|--------|-------|
| Batch size | 300 users |
| Discord rate limit | ~5 req/sec (handled automatically) |
| Time per batch | ~1–3 min |
| Schedule interval | Every 5 min |
| Effective rate | ~18,000 users/hr |
| 10,000 users | ~35 min |
| 600,000 users | ~33 hrs (background) |

**To increase throughput:** change `301` → `501` in the Read Next Batch range expression and adjust the schedule to every 8 min to keep a safe buffer against Discord rate limits.
