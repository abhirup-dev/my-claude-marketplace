---
description: Refresh all tracked threads stale for over 1 hour
allowed-tools: Read, Write, Bash, Task
---

Re-read all tracked threads whose `last_read_time` is older than 1 hour and update their files in-place.

**Steps:**

1. **Read `Threads.txt`** — parse all entries.

2. **Identify stale entries** — collect every entry where `last_read_time` is more than 1 hour before the current time.

3. **Skip if none** — if no stale entries exist, report "All threads are up to date." and stop.

4. **Spawn parallel Explore agents** — launch one Explore agent per stale thread concurrently. Each agent must:

   a. Use the Slack MCP tool to re-fetch the full thread.

   b. Open the existing `threads/<filename>.md`.

   c. **Merge updates** (do not duplicate existing content):
      - Refresh `Last-read-time` to now.
      - Add new Content bullet points for messages that appeared after the previous `last_read_time`. Do **not** re-add existing points.
      - Append new rows to the References table for any links not already present.
      - Re-evaluate and rewrite the **Impactful** section based on the full thread.
      - Add the **Conclusion** section if the last reply is now > 1 month old; remove it if the thread has become active again.

   d. Write the updated file back.

   e. Return: `{ url, filename, new_last_read_time }`.

5. **Update `Threads.txt`** — overwrite `last_read_time` for each refreshed entry.

6. **Report to user** — list the files updated and the count of new messages incorporated per thread.
