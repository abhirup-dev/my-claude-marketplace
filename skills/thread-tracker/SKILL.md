---
name: thread-tracker
description: Track and summarise Slack threads locally. Use /track-thread <url> to add a new thread, or /update-threads to refresh all tracked threads whose last-read-time is older than 1 hour.
---

# Thread Tracker Skill

Track and summarise Slack threads locally inside an agent's assigned working folder.

---

## Storage Layout

All files live **relative to the agent's assigned folder** (the folder the cowork agent has been given as its workspace).

```
<assigned-folder>/
├── Threads.txt          # Human-readable index of all tracked threads
└── threads/             # One .md file per thread
    └── <filename>.md
```

### `Threads.txt` format

Each entry is a YAML-like block separated by blank lines:

```
- url: <slack_thread_url>
  file: <filename>
  channel: <channel_name>
  thread_id: <thread_ts>
  first_message_summary: <one-sentence summary of the opening message>
  last_read_time: <ISO 8601>
```

### Thread filename format

```
<channel>__<3-word-context>__<@person1-@person2>__<YYYY-MM-DD>.md
```

- **channel** — extracted from the Slack URL (e.g. `C01AB2CD3EF` → use the human-readable channel name if discoverable, otherwise the raw ID)
- **3-word-context** — three hyphen-joined lowercase words summarising the thread topic (generated after reading the first few messages)
- **@person1-@person2** — `@handle` of the most active/tagged people in the first few messages, up to 3 handles joined with `-`; omit the `@` in the filename itself (e.g. `alice-bob-carol`)
- **YYYY-MM-DD** — date of the first message in the thread

Example: `general__deploy-pipeline-broken__alice-bob__2024-03-15.md`

---

## Thread File Template

```markdown
## Info
- Thread link: <url>
- Channel: <channel name>
- Start time: <ISO 8601 of first message>
- Last-read-time: <ISO 8601>
- Context: <one-sentence description of what the thread is about>
- People involved: <@handle1>, <@handle2>, ...
- Project details: <project/team name if discernible, else "N/A">

## Content
<!-- Bullet-point summary of the discussion. Attribution format: **@handle** — point made. -->

## References
| Link | Shared by | Context |
|------|-----------|---------|

## Impactful
<!-- Who drove the conversation most — ranked list with brief reasoning. -->

## Conclusion
<!-- Only written if the last reply is > 1 month ago. Otherwise this entire section is OMITTED. -->
<!-- Summary of outcome/resolution. -->
```

---

## Commands

---

### `/track-thread <slack-url> [<slack-url2> ...]`

**Purpose:** Add one or more Slack thread URLs to the tracker and generate summary files.

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

   c. Construct the filename using the format above.

   d. Create `threads/<filename>.md` populated with all five sections from the template. For `Last-read-time`, use the current time in ISO 8601.

   e. Return: `{ url, filename, channel, thread_id, first_message_summary, last_read_time }`.

5. **Update `Threads.txt`** — append a new entry block for each successfully processed thread.

6. **Report to user** — list the files created and any skipped duplicates.

---

### `/update-threads`

**Purpose:** Re-read all tracked threads whose `last_read_time` is older than 1 hour and update their files in-place.

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

---

## Implementation Notes

- Always use the **Slack MCP tool** (e.g. `mcp__claude_ai_Slack__slack_read_thread`) to fetch thread content. Never ask the user to paste thread content manually.
- When spawning Explore agents in parallel, pass each agent the full thread URL, the assigned folder path, and the existing `Threads.txt` content so it has all necessary context.
- ISO 8601 timestamps should include timezone offset, e.g. `2024-03-15T14:32:00+05:30`.
- Handle Slack rate-limit errors gracefully: if an agent fails to fetch a thread, log the error in the thread file under a `## Errors` section and continue processing other threads.
- Never overwrite a thread file from scratch on update — always merge to preserve historical Content bullets and References rows.
