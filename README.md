# Discord Mass Role Assignment — n8n Workflow

Assigns one Discord role to ~570,000 users from a Google Sheet, using only default n8n nodes (Google Sheets, Discord, Code, Loop Over Items, Wait). It runs unattended on a 5-minute schedule, throttles itself under Discord's rate limits, survives crashes and restarts via a single checkpoint value, and logs every run and every failed user back to the sheet.

```
[Every 5 min / Run Manually]
 → Read Config          checkpoint from Config tab
 → Read User Batch      next 250 rows of the Users tab
 → Normalize IDs        extract user_id from each row
 → Loop Over Items      10 users per cycle
     → Assign Role      Discord member:roleAdd, continues past errors
     → Throttle 6s      stay under Discord's rate limit
 → Compute Summary      counts + list of failed users
 → Advance Checkpoint   1 Sheets write per run
 → Append Run Log       1 heartbeat row per run
 → Append Failures      failed users, batched into 1 write (only if any)
```

---

## Design decisions (and why)

### Google Sheet, not a Code node full of IDs — and one full list, not 57 files

570k user IDs embedded in a Code node is ~11 MB of workflow JSON. n8n stores the workflow in its database and loads it into the editor on every open — a workflow that size is somewhere between painful and broken, and it gives you no persistent record of progress or failures. Even split into 57 batches of 10k, you'd be hand-pasting and re-importing 57 times with no audit trail.

A Google Sheet holds all 570k rows in one tab without issue (the limit is 10M cells), the workflow only ever reads 251 rows per run so memory stays flat, and the same spreadsheet doubles as the progress tracker and failure log. **Put the full list in one tab — your pre-split 10k files are no longer needed.** The checkpoint does the batching for you.

### Progress tracking without hitting Google's quota

Your earlier quota problems came from writing to Sheets once per user. This workflow makes **at most 3 write calls per 5-minute run** (checkpoint update, log append, failures append — and the failures append batches all failed rows of the run into one call). Google allows 60 write requests per user per minute, so we sit at well under 1% of quota.

Successes are tracked implicitly — everything below the checkpoint that isn't in the Failures tab succeeded. Only failures get their own rows.

### Discord rate limits

There is no bulk role-assignment endpoint — Discord requires one API call per member (`PUT /guilds/{guild}/members/{user}/roles/{role}`). Discord doesn't publish the limit for this route, but the commonly observed bucket is roughly **10 requests per 10 seconds per guild**. The workflow paces itself at ~1.1 requests/second (10 users, then a 6-second wait), which is the honest planning number for this job:

| Setting | Rate | 570k users takes |
|---|---|---|
| Default: 250 users / 5 min | 3,000/hr | **~8 days** |
| Tuned: 400 users / 5 min, 3s throttle | 4,800/hr | ~5 days |

That multi-day wall clock is a Discord constraint, not an n8n one — no tool can bulk-assign a role to 570k members meaningfully faster. (Sidebar: if the real goal is "everyone in the server gets this role", consider whether adjusting the `@everyone` role's permissions, or Discord's onboarding/auto-roles, gets you there without half a million API calls.)

### Crash safety

Adding a role a user already has is a harmless no-op, so the checkpoint only needs to be *approximately* right. It's written once per run, after the batch completes; if n8n dies mid-batch, the next run redoes at most 250 users — a couple of wasted minutes, never lost users. The checkpoint also advances by **rows read**, not by successes, so a malformed or blank row can never wedge the workflow into re-reading the same batch forever — bad rows land in the Failures tab and the run moves on.

---

## Google Sheets setup

One spreadsheet, four tabs. Column headers must match exactly.

### Tab `Config` — one data row

| name | value |
|---|---|
| checkpoint | 0 |

### Tab `Users` — one column, all 570k IDs

| user_id |
|---|
| 111222333444555666 |
| 222333444555666777 |

> ### ⚠️ Format the `user_id` column as **Plain text** *before* pasting the IDs
> This is the single most common way this job silently fails. Google Sheets stores numbers as floating point, which only holds ~15 significant digits — Discord IDs have 17–19. An ID stored as a number gets **rounded**, e.g. `843109278293819801` becomes `843109278293819000`, and every request for it fails (or worse, targets nothing). Select column A → Format → Number → Plain text, *then* paste. If your old sheet has lots of IDs ending in `00`, they're already corrupted — re-import from the original source.

### Tab `Log` — headers only, rows appended by the workflow

| timestamp | checkpoint_before | attempted | succeeded | failed | new_checkpoint |
|---|---|---|---|---|---|

### Tab `Failures` — headers only, rows appended by the workflow

| timestamp | user_id | error | checkpoint_batch |
|---|---|---|---|

---

## n8n setup checklist

1. Import `discord-mass-role-assignment.json`.
2. Replace `YOUR_SPREADSHEET_ID` on all **5 Google Sheets nodes** (Read Config, Read User Batch, Advance Checkpoint, Append Run Log, Append Failures) and set your Google Sheets credential on them.
3. Set your Discord Bot credential on **Assign Role**, then open the node and pick your **Server** and the **Role** from the dropdowns (they load live from your bot).
4. Bot prerequisites: the bot is in the server, has **Manage Roles**, and its highest role sits **above** the role being assigned in the server's role list.
5. `checkpoint` = `0` in the Config tab.

## Test before going live

1. Temporarily point the range at a short list, or just use your real Users tab — with checkpoint `0` a manual run processes only the first 250 users.
2. Click **Run Manually** (leave the workflow deactivated).
3. Verify: a row appeared in `Log` with `attempted` = 250 (or your list size), `new_checkpoint` advanced, roles visible on a few users in Discord, and any expected failures (e.g. a deliberately fake ID) in `Failures`.
4. If `attempted` is much smaller than `checkpoint_before → new_checkpoint` implies, stop and investigate before activating — the Log row is designed to make exactly this kind of drift visible.
5. **Activate** the workflow. It now runs every 5 minutes on its own.

## Monitoring — how you know it's working

- **Log tab**: one row every 5 minutes is your heartbeat. `new_checkpoint` marching upward is your progress bar (÷ 570,000 for percent done).
- **Failures tab**: every user that couldn't be assigned, with Discord's error message and which batch it happened in. Expected entries at this scale: `Unknown Member` (user left the server) and malformed rows. **429 / rate-limit entries mean you've tuned too fast — back off.**
- **Done**: when all rows are processed, runs read an empty batch and end quietly — Log rows stop appearing and `checkpoint` equals your list length. Deactivate the workflow.

## Pause / resume / re-run

- **Pause**: deactivate the workflow. **Resume**: activate it — the checkpoint picks up exactly where it stopped.
- **Restart from scratch**: set `checkpoint` to `0`.
- **Re-run failures**: when the main run finishes, filter the Failures tab for retryable errors (rate limits, timeouts — not `Unknown Member`), paste those IDs into a fresh Users tab (plain text!), set `checkpoint` to `0`, and let it run again. Users that actually succeeded in the meantime are unaffected — re-adding a role is a no-op.

## Tuning

Only tune after the Failures tab has stayed clean of rate-limit errors for a few hours:

| Knob | Where | Default | Faster |
|---|---|---|---|
| Users per run | `251` in both range expressions on Read User Batch | 250 | `401` → 400 |
| Throttle | Wait amount on Throttle | 6s | 3s |
| Schedule | Every 5 Minutes trigger | 5 min | leave at 5 min |

Keep the batch small enough that a run comfortably finishes inside the 5-minute interval (default takes ~3.5 min). If two runs ever overlap, nothing breaks — both would process the same batch and write the same checkpoint, wasting a little work — but sustained overlap doubles your request rate against Discord, so treat it as a signal to shrink the batch.

## Failure modes worth knowing

| Symptom | Cause | Fix |
|---|---|---|
| Everything fails with `Missing Permissions` | Bot's role is below the target role, or lacks Manage Roles | Drag the bot's role above the target role |
| Lots of `Unknown Member` | Those users left the server | Expected at this scale; nothing to do |
| 429 / rate-limit errors in Failures | Throttle tuned too aggressively | Raise the Wait back toward 6s; re-run those users later |
| IDs in Failures end in `00` | Users column was formatted as numbers | Re-import IDs as plain text (see warning above) |
| Log rows stop but checkpoint < list length | An execution is erroring — check n8n's Executions list | Usually credentials expired (Google OAuth); reconnect and it resumes |
