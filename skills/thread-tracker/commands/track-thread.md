---
description: Add one or more Slack thread URLs to the tracker
allowed-tools: Read, Write, Bash, Task
argument-hint: <slack-url> [<slack-url2> ...]
---

Add one or more Slack thread URLs to the tracker and generate summary files.

**Steps:**

1. **Resolve assigned folder** — use the agent's configured workspace directory.

2. **Initialise storage** — if `Threads.txt` does not exist, create it as an empty file. If `threads/` directory does not exist, create it.

3. **Deduplicate** — for each provided URL, check `Threads.txt`. If the URL is already present, skip it and note the skip to the user.

4. **Spawn parallel Explore agents** — launch one Explore agent per new URL concurrently. Each agent must:

   a. Use the Slack MCP tool to fetch the full thread (all replies) for the given URL.

   b. Parse the thread to extract:
      - Channel name
      - Thread timestamp (`thread_ts`)
      - Date of the first message
      - Handles of the most active/tagged participants (up to 3)
      - A 3-word topic summary (hyphen-joined, lowercase)
      - A one-sentence `first_message_summary`

   c. Construct the filename using the format defined in the thread-tracker skill.

   d. Create `threads/<filename>.md` populated with all five sections from the template. For `Last-read-time`, use the current time in ISO 8601.

   e. Return: `{ url, filename, channel, thread_id, first_message_summary, last_read_time }`.

5. **Update `Threads.txt`** — append a new entry block for each successfully processed thread.

6. **Report to user** — list the files created and any skipped duplicates.
