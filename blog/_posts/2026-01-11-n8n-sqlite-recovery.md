---
layout: post
title: "Recovering from the Void: A Forensic Journey into n8n and SQLite Corruption"
published: true
post_date: 2026-01-11 22:00
---

# Recovering from the Void: A Forensic Journey into n8n and SQLite Corruption

## The Crisis: The "Upgrade" that Vanished
It started with a routine minor upgrade of my n8n instance - this was triggered by seeing some articles mentioning some severe security issues. It was late, and since I'd upgraded n8n heaps of times before and this was a minor upgrade, I didn't even check if it was working as expected beyond the container starting.

Everything seemed fine until I logged in the next morning to investigate why some expected triggers hadn't gone off. To my horror, my workflows had reverted to a state from about 5 days ago. I had lost almost a week of work tuning some automations I was using for sending messages from my Frigate NVR, which I am using for my home security system.

In this post, I’ll walk you through the technical details of recovering data from a corrupted, 14GB SQLite database, the forensic tools we used, the mistakes made, and how I eventually "surgically repaired" my way back to a working instance. I learned a lot, and I figure if that helps someone else one day, it's worth writing up.

*Note: I used ChatGPT and Gemini (once I ran out of messages on ChatGPT) to help me work through all this. While I'm confident I could have got there eventually the old-fashioned way, it sure felt easier using these tools.*

---

## 1. The Mystery of the Missing Days
I initially attempted a restore from a **Restic** snapshot taken right before the upgrade. However, the Docker container refused to stay up, spitting out this error:

```text
n8n-1  | There was an error running database migrations
n8n-1  | SQLITE_IOERR: disk I/O error
n8n-1 exited with code 1
```

### What likely happened during the upgrade:
In a 14GB SQLite database on a busy n8n instance, the database file and its rollback journal are in a constant state of flux. When I performed the upgrade, a few things probably converged to create the "5-day gap":

1.  **Delayed Checkpointing/Merging:** With a database that size, SQLite can struggle to merge journal data back into the main `.sqlite` file. It's possible that days' worth of changes were durable in the journal's eyes but hadn't been fully baked into the main file.
2.  **The "Migration Trap":** When n8n upgrades, it runs database migrations. If a migration is interrupted or crashes due to the sheer size of the tables or memory exhaustion, it can roll back the database to the last consistent state. In my case, that "last consistent state" was unexpectedly old.
3.  **Torn Writes and Volume Locking:** Docker volume mounts can sometimes mismanage how SQLite expects locks to work. The upgrade process likely triggered a lock contention that prevented the latest transactions from being written to the physical file before the container restarted.

By the time I checked the next morning, the system was essentially looking at a "snapshot" of the database from several days prior, with the most recent changes trapped in a journal file that the new version of n8n couldn't reconcile.

### Technical Root Cause:
My database was massive (14.6 GB). SQLite default behavior uses **Rollback Journals** (the `database.sqlite-journal` file). Because the backup was taken while n8n was running (a "hot backup"), the snapshot captured the database in the middle of a transaction.

When I restored it, SQLite saw the journal and tried to "undo" or "redo" that transaction. Because of the sheer size of the DB and likely filesystem locking issues within Docker, it failed, leading to the `SQLITE_IOERR`.

### Why was my database so massive?

Well, I had been working on automations that send notifications via Telegram when my Frigate instance detected people at the front door, etc. What I didn't consider is that 1) these events fire very often and 2) n8n was retaining the binary image/video data of these executions. While I got these cleared up eventually, I didn't notice that in my testing it had caused the n8n database to balloon.

---

## 2. Mistake #1: Removing (and then Replaying) the Journals

In a naive attempt to get the DB to boot, I first moved the journal files (`database.sqlite-journal` and `crash.journal`) out of the way. 

**The Result:** n8n started! **The Problem:** The workflows were still 5 days old.

### The Theory of the "Ghost Journal"
I realized that by deleting the journals, I had essentially told SQLite to ignore all the changes waiting "on the doorstep" and to only look at the building as it stood 5 days ago. I thought I could outsmart SQLite by restoring from backup again and forcing it to "replay" those journals manually.

I tried to open the database using the SQLite CLI, hoping it would see the journal and apply the changes. 

**What actually happened:**
The replay "succeeded" in a technical sense—the journal file disappeared—but when I checked the timestamps on the workflows, **they were still 5 days old.**

### The Insight:
When a journal replay fails to show "new" data, it’s usually because of one of two brutal realities:
1. **The Journal was a Rollback, not a Roll-forward:** In "Rollback Journal" mode (the default), the journal contains the *old* data to be restored if a transaction fails. If the database was captured mid-migration, replaying the journal actually *ensures* the database stays in its older, pre-migration state.
2. **Inconsistency:** Because the "hot backup" was taken while the file was being written, the journal and the main DB file were subtly out of sync. SQLite looked at them, decided they didn't match perfectly, and chose to preserve the database's "safe" state—the 5-day-old version—rather than risking corruption by replaying a mismatched journal.

At this point, I realized that replaying the journal was effectively a dead end. To get the data back, I had to stop acting like a database administrator and start acting like a digital archaeologist.

---

## 3. The Forensic Breakthrough: `strings` and `.recover`

I knew the data was there. I had a feeling that if I could just see if the "missing" text existed in the file, I'd know if I was chasing a ghost. 

I was using **Gemini** to help brainstorm, and it gave me a simple but brilliant idea: use the Linux `strings` utility to search the binary file for the names of the workflows I knew I had edited in the last 48 hours.

I ran a quick check on the raw binary of the 14GB file:
```bash
strings database.sqlite | grep "Frigate - Reviews (Stitcher)"
```
**It returned results!** 

Seeing that specific string—a workflow name I had only created two days prior—was the "smoking gun." It meant the JSON text was physically on the disk in the "unallocated space" or trapped in pages the index had ignored. It wasn't "gone"; it was just "unindexed."

### The Tool: `sqlite3 .recover`
Standard SQL queries follow the index. If the index is corrupt or rolled back, the data is "invisible." The `.recover` command (added in SQLite 3.29.0) ignores the index and scans every byte of the file, "carving" out valid-looking rows.

**The Command:**
```bash
# Using Alpine because it ships with a modern, feature-complete SQLite
docker run --rm -v "$(pwd):/workspace" -w /workspace alpine sh -c "
  apk add --no-cache sqlite && \
  sqlite3 database.sqlite '.recover' | sqlite3 recovered_forensic.sqlite"
```

**Crucial Insight:** I had to restore the backup *again* and run this **without letting the journal replay**. Replaying the journal would have "officially" deleted the forensic data I was trying to find by overwriting those "free" pages with the old 5-day-old state. By deleting the journal and running `.recover` on the "dirty" database file, I was forcing the tool to scrape the "quiet" parts of the file where the newer data was still sitting, without letting the journal "clean it up."

---

## 4. The 14GB Elephant: Cleaning the Bloat
The recovery worked, but I now had a 14.9 GB recovered file. I knew that my workflows certainly didn't add up to 14 GB, which meant the file was mostly "bloat"—likely old execution history and logs.

### Investigating the Bloat
To confirm exactly where the space was going, I used a SQLite query to check the size of each table. Since the file was too large to open easily in a GUI, I ran this via Docker:

```bash
docker run --rm -v "$(pwd):/workspace" -w /workspace alpine sh -c "
  apk add --no-cache sqlite && \
  sqlite3 recovered_forensic.sqlite '
    SELECT 
      name, 
      SUM(pgsize)/1024/1024 AS size_mb 
    FROM dbstat 
    GROUP BY name 
    ORDER BY size_mb DESC 
    LIMIT 10;'
"
```

The results were eye-opening:
- **`execution_data`**: 14,212 MB
- **`workflow_entity`**: 0 MB (Relative to the GBs of logs, my actual work was a rounding error)

It turned out that `execution_data`—the table that stores the massive JSON blobs and binary metadata from every node in every past run—was responsible for 99.9% of the database size.

### The Surgical Clean
To make the database safe for n8n to use again, I had to perform a "surgical strike" to remove this bloat. If I hadn't, the first thing n8n would do on startup is try to run migrations on these 14GB tables, which would almost certainly crash the process again.

```sql
DELETE FROM execution_data;
DELETE FROM execution_entity;
VACUUM;
```
The `VACUUM` command is the most important part here—it tells SQLite to actually rebuild the file and release the empty space back to the OS. After this, the DB shrank from **14.9GB to roughly 20MB**.

---

## 5. Database Surgery: Solving the "Broken JSON" Crash
Even with the data recovered and the "bloat" removed, n8n still refused to boot. It crashed with a familiar but frustrating error:
`SyntaxError: Unexpected end of JSON input`

Forensic recovery isn't perfect. It scans for valid-looking rows, but sometimes it recovers "torn writes"—scraps of data where the beginning of a JSON string was found, but the end had been overwritten.

### Identifying the Radioactive Rows
I used SQL to find exactly which records were invalid JSON. I looked at both the active workflows and the history table:

```sql
SELECT id, name, createdAt, updatedAt, active, versionId, json_valid(nodes), json_valid(connections) FROM workflow_entity ORDER BY updatedAt DESC;
SELECT versionId, workflowId, createdAt, updatedAt, json_valid(nodes), json_valid(connections) FROM workflow_history;
```

This highlighted two major issues:
1. **The Broken Active Workflow:** `Agent Tools Backend` (ID: `pcgNxetwshnFEyB5`) was corrupted. Because this workflow was marked as "Active," n8n tried to parse and instantiate it on boot, causing the global crash.
2. **The Corrupted History:** A specific record for `Frigate - Reviews (Stitcher)` in the `workflow_history` table was also mangled, which would have crashed the UI later if n8n had managed to start.

### The "Fixes.sql" Strategy
I didn't want to play whack-a-mole with a live database. I adopted an iterative repair strategy:
1. Restore the "Recovered & Vacuumed" DB from a safety copy.
2. Run a `fixes.sql` script.
3. Try to boot n8n.
4. If it crashed, analyze the error, add a new fix to `fixes.sql`, and go back to step 1.

This meant I ended up with a single, reproducible script that fixed everything at once.

#### Repairing the "Agent Tools Backend"
Since the main record was toast, I used the `workflow_history` table as an "organ donor." I pulled the last known-good nodes and connections into the main entity table:

```sql
-- Fix Agent Tools Backend (pcgNxetwshnFEyB5) from its own history
UPDATE workflow_entity 
SET nodes = (SELECT nodes FROM workflow_history WHERE workflowId = 'pcgNxetwshnFEyB5' ORDER BY createdAt DESC LIMIT 1),
    connections = (SELECT connections FROM workflow_history WHERE workflowId = 'pcgNxetwshnFEyB5' ORDER BY createdAt DESC LIMIT 1),
    staticData = NULL, -- Clear potentially corrupted state
    pinData = '{}',
    versionId = '6b6369ae-e7c4-46b4-b0af-e07c3b08f612', 
    activeVersionId = '6b6369ae-e7c4-46b4-b0af-e07c3b08f612' 
WHERE id = 'pcgNxetwshnFEyB5';
```
I also had to "borrow" some metadata (like `meta` and `settings`) from another healthy workflow to ensure the row was structurally complete.

#### Cleaning up the History
For the `Frigate` workflow, the history record itself was a problem. I repaired it by reversing the process—copying the valid nodes from the main entity back into the history to stop the background workers from choking:

```sql
UPDATE workflow_history 
SET nodes = (SELECT nodes FROM workflow_entity WHERE id = 'QUx4a1z0HKn0bHPN') 
WHERE workflowId = 'QUx4a1z0HKn0bHPN';
```

After applies these repairs, n8n finally booted to the login screen.

---

## 6. Lessons Learned & Making it Robust
I survived this, but I never want to do it again. Here is the new architecture I've implemented:

### A. Aggressive Pruning
I updated n8n environment variables to prevent the DB from ever hitting 14GB again. If you're doing anything with images or high-frequency triggers, this is mandatory:
- `EXECUTIONS_DATA_MAX_AGE=48` (2 days)
- `EXECUTIONS_DATA_PRUNE=true`

### B. Decoupled Backups (The "Gold Standard")
I no longer rely solely on restic backups of the host (i.e. of the SQLite file). I now run a hourly cron job that exports workflows and credentials to a flat-file directory using the n8n CLI:
```bash
n8n export:workflow --backup --output=/backups/workflows
n8n export:credentials --backup --output=/backups/credentials
```

### C. Git-Based Versioning
I turned the `/backups` folder into a **Git repository**. 
Now, every hour, the server automatically commits the JSON files. I have a full `git diff` history of every change I make to every workflow. If the DB dies, I just `n8n import` the JSON. This is essentially a "Time Machine" for my automations.

---

## Conclusion
If you are running n8n on SQLite:
1. **Monitor your DB size.** Over 1GB is a warning; over 10GB is a disaster waiting to happen.
2. **Don't trust "hot" backups** of SQLite files—always quiesce or stop the container if you can.
3. **Use the CLI export tool.** It is your only true safety net.
4. **Use AI Tools** Used right, they make working through these complex issues alot easier.

Forensic recovery is possible, but it’s much easier to just `git checkout` your way out of a problem.
