---
name: kiseki
description: "Persistent cross-session memory via Kiseki: search past conversations, record decisions, track solutions across sessions. Use when the user references prior work, when starting complex tasks, or when recording important outcomes."
metadata: { "openclaw": { "emoji": "🧠", "requires": { "bins": ["kiseki"] }, "install": [{ "id": "go", "kind": "go", "module": "github.com/Gsirawan/kiseki-beta@latest", "bins": ["kiseki"], "label": "Install Kiseki (go install)", "buildTags": "fts5" }] } }
---

# Kiseki Memory

You have access to **persistent cross-session memory** through Kiseki MCP tools. Kiseki stores conversations, notes, and documents in a local SQLite database with multi-layer search (FTS5 full-text + vector similarity on chunks + vector similarity on messages). Results are ranked by importance tier and layer overlap.

**This is different from OpenClaw's built-in `memory_search`.** OpenClaw's memory searches workspace markdown files (`MEMORY.md`, `memory/*.md`). Kiseki searches ingested documents and captured conversation history across all sessions - it is your long-term recall.

| OpenClaw `memory_search` | Kiseki `kiseki_search` |
|---|---|
| Current workspace markdown files | All ingested docs + conversation history |
| Single-session scope | Cross-session, cross-project |
| Text chunks from `.md` files | Multi-layer: FTS5 + vector chunks + vector messages |
| No importance ranking | Importance tiers: stone > solution > key > normal |

**Use both.** OpenClaw memory for recent workspace notes. Kiseki for historical recall across sessions.

---

## When to Search Kiseki

Search Kiseki when any of these conditions are true:

**Session continuity.** At the start of a session that continues prior work, search for context on the topic before diving in. This is how you recover what was discussed, decided, or configured in previous sessions.

**User references the past.** Phrases like "last time", "we discussed", "remember when", "previously", "the way I like it", "like before", "my usual" all signal that the user expects you to recall something from a prior session.

**User preferences and configurations.** Before any task where the user's past preferences matter (email tone, report format, scheduling patterns, naming conventions, notification preferences), search kiseki for stored preferences. The user should not have to repeat themselves.

**Checking for prior solutions.** Before debugging a problem or making a decision, check whether a similar problem was solved before or a relevant decision was already made. Use `kiseki_stone_search` first for recorded solutions, then `kiseki_search` for broader context.

**Do NOT search Kiseki for:**
- Information clearly present in the current conversation
- Generic knowledge questions unrelated to the user's history
- Transient data like today's weather, stock prices, or news headlines

---

## What to Store in Kiseki

Kiseki is for **patterns, preferences, decisions, and solutions** - not for logs or transient data. Before storing anything, apply the **30-day test**: "Will this information still be useful 30 days from now?" If not, it probably does not belong in kiseki.

### STORE (high value, long-lived)

- **User preferences:** Communication style, report format, scheduling patterns, notification settings, tool configurations, naming conventions. These are the things the user should never have to repeat.
- **Decisions and rationale:** Architecture choices, workflow decisions, policy changes, and why they were made. Record the reasoning, not just the outcome.
- **Solutions to problems:** Bugs fixed, workarounds discovered, troubleshooting steps that worked. Use stones for these.
- **Corrections and feedback:** When the user corrects you ("no, I prefer it this way", "that's not how I want it"), store the correction. This is the highest-signal data for improving future interactions.
- **Important configurations:** API setups, integration settings, environment-specific details the user has configured over time.
- **User-specific workflows:** Recurring processes the user follows, custom steps, preferred tools for specific tasks.

### DO NOT STORE (noise, transient, or sensitive)

- **Transient data:** Daily news summaries, stock prices, weather reports, calendar events. These change constantly and create noise in search results. Store the user's *preferences about* these reports (format, timing, which stocks to track) but not the reports themselves.
- **Raw content from external systems:** Do not bulk-ingest email content, calendar entries, or document dumps. These belong in their native systems. Only store the user's *preferences and patterns* around these systems.
- **Routine assistant responses:** Do not store every response you generate. Only record responses that contain decisions, solutions, or configurations the user might reference later.
- **Sensitive data (NEVER):** Passwords, API keys, tokens, financial account numbers, medical information, social security numbers, credit card numbers. These must NEVER be stored in kiseki. If the user asks you to remember credentials, decline and suggest a password manager instead.

### Importance Tiers

When creating stones or marking items, use the correct importance tier:

| Tier | When to use | Example |
|---|---|---|
| `solution` | A fix or workaround that solved a specific problem | "Fix: OAuth token refresh race condition" |
| `key` | An important decision, preference, or configuration | "User prefers markdown tables over bullet lists for reports" |
| `normal` | General context worth preserving but not critical | "Discussed migrating to new email provider, no decision yet" |

Most items are `normal`. Reserve `solution` for actual fixes and `key` for real decisions or preferences. Over-promoting items degrades search ranking quality.

---

## Available Tools

### kiseki_search

Multi-layer search across all memories. Returns chunks and messages ranked by layer overlap and similarity. Uses three parallel search layers: FTS5 full-text, vector similarity on document chunks, and vector similarity on conversation messages. Items found by multiple layers rank higher.

**Parameters:**
- `query` (string, required): Search query. Be specific and topical.
- `limit` (integer, optional): Maximum results. **Always set to 10-20.** Default is 250, which floods context.
- `as_of` (string, optional): ISO date filter. Only returns results from before this date.

**Example:**
```json
{ "query": "email report format preferences", "limit": 15 }
```

### kiseki_search_msg

Search messages with conversation context window. Returns snippets of actual conversations around matching messages. Supports both semantic search (default) and exact phrase matching via the `fts` flag.

**When to use instead of `kiseki_search`:**
- Looking for a specific discussion or phrase the user said
- Need to see the conversation flow around a topic, not just isolated chunks
- User asks "what did we talk about when..." or "what did I say about..."

**Parameters:**
- `query` (string, required): Search query
- `limit` (integer, optional): Maximum results. **Set to 10-15.**
- `fts` (boolean, optional): Use exact phrase matching (FTS5/LIKE) instead of semantic search. Useful when you know the exact words used.
- `context` (integer, optional): Context window in minutes around each match. Default is 3.

**Example:**
```json
{ "query": "scheduling preferences morning meetings", "limit": 10, "context": 5 }
```

### kiseki_stone_add

Create a named record for an important solution, decision, configuration, preference, or pattern. Stones are permanent, searchable, and rank above all other results in search.

**When to create a stone:**
- After solving a problem the user struggled with - record the fix
- After the user states a preference or configuration - record it
- After making a decision with rationale - record what was chosen and why
- When discovering a pattern or workaround worth preserving
- When the user explicitly asks to remember something for future reference

**Parameters:**
- `title` (string, required): Descriptive title. Use a prefix: "Fix:", "Preference:", "Decision:", "Config:", "Pattern:"
- `category` (string, optional): `fix`, `decision`, `pattern`, `workaround`, `config`
- `problem` (string, optional): What the problem or question was
- `solution` (string, optional): What solved it, what was decided, or what the preference is
- `tags` (string, optional): Comma-separated tags for filtering
- `chunk_ids` (string, optional): Comma-separated related chunk/message IDs from search results
- `key_chunk_ids` (string, optional): Comma-separated key chunk/message IDs
- `source_session` (string, optional): Source session ID for traceability

**Example - recording a user preference:**
```json
{
  "title": "Preference: Weekly report format",
  "category": "config",
  "problem": "User wants consistent formatting for weekly summary reports",
  "solution": "Use markdown tables for metrics, bullet points for action items, keep under 500 words. Send every Friday at 4pm.",
  "tags": "reports,formatting,weekly,preferences"
}
```

**Example - recording a fix:**
```json
{
  "title": "Fix: Calendar sync timezone mismatch",
  "category": "fix",
  "problem": "Calendar events showed wrong times after syncing with Google Calendar",
  "solution": "User's Google Calendar was set to UTC. Changed to America/New_York in Google settings. Kiseki now converts all times to user's local timezone before displaying.",
  "tags": "calendar,timezone,google,sync"
}
```

### kiseki_stone_search

Search existing stones by title, tags, category, or week.

**When to use:**
- Looking for a known solution or decision recorded previously
- Checking if a similar problem was solved before
- Reviewing the user's stored preferences or configurations
- Reviewing decisions made during a specific period

**Parameters:**
- `query` (string, optional): Text search across title, tags, problem, solution fields
- `category` (string, optional): Filter by category (`fix`, `decision`, `pattern`, `workaround`, `config`)
- `week` (string, optional): Filter by ISO week (e.g., `2026-W12`)
- `limit` (integer, optional): Maximum results. **Set to 10-20.**

**Example:**
```json
{ "query": "email preferences", "category": "config", "limit": 10 }
```

### kiseki_history

Chronological timeline of all mentions of an entity (person, project, tool, service, concept).

**When to use:**
- Tracing the evolution of a topic over time
- Getting full context on a person, project, or service the user works with
- Understanding when and how decisions or preferences changed

**Parameters:**
- `entity` (string, required): Entity name to trace (person, project, tool, etc.)
- `limit` (integer, optional): Maximum results. **Set to 20.**

**Example:**
```json
{ "entity": "weekly-report", "limit": 20 }
```

### kiseki_ingest

Ingest a markdown file into Kiseki's memory store. The file is chunked, embedded, and indexed for future search.

**When to use:**
- After creating important documentation the user will reference later
- When the user provides a document they want searchable across sessions
- At session end if significant decisions or configurations were documented

**When NOT to use:**
- Do not manually ingest workspace memory files -- this is handled by the automated cron job
- Do not ingest transient content (daily news, stock reports, weather summaries)
- Do not bulk-ingest routine assistant output
- Do not ingest files containing sensitive data (credentials, financial details)

**Parameters:**
- `file_path` (string, required): Path to the markdown file. Absolute paths require `KISEKI_INGEST_ROOT` to be set.
- `valid_at` (string, optional): ISO date stamp for the content. Helps with temporal search filtering.

**Example:**
```json
{ "file_path": "memory/2026-03-22.md", "valid_at": "2026-03-22" }
```

### kiseki_status

Check Kiseki system health: database stats, embedding model status, chunk/message counts.

**When to use:**
- Diagnosing why search returns no results (is the database empty? is Ollama down?)
- Verifying Kiseki is operational after setup or configuration changes
- Checking how much memory is stored

**Parameters:** None.

---

## Usage Patterns

### Session Start - Context Recovery

If the session involves continuing prior work or the user's first message references past context, search Kiseki before responding:

```
kiseki_search({ "query": "<topic from user's first message>", "limit": 15 })
```

Also check for stored preferences relevant to the task:

```
kiseki_stone_search({ "query": "<task type> preferences", "limit": 10 })
```

This gives you historical context and user preferences before diving in. Skip this for clearly new topics with no prior history.

### When User References the Past

Any of these signals should trigger a Kiseki search:
- "last time", "we discussed", "remember when", "previously"
- "the fix for", "that bug with", "the decision about"
- "what did we decide", "how did we solve"
- "like I prefer", "the way I like it", "my usual", "like before"
- "the settings we configured", "that thing we set up"

Run 2-3 searches from different angles if the first one does not return strong results.

### After Important Interactions

**Record user preferences immediately.** When the user states a preference, corrects your approach, or configures something, create a stone right away. Do not wait until the end of the session.

```
kiseki_stone_add({
  "title": "Preference: <what the user prefers>",
  "category": "config",
  "solution": "<the specific preference or configuration>",
  "tags": "<relevant,tags>"
})
```

**Record solutions after fixing problems:**

```
kiseki_stone_add({
  "title": "Fix: <concise description>",
  "category": "fix",
  "problem": "<what was wrong>",
  "solution": "<what fixed it>",
  "tags": "<relevant,tags>"
})
```

**Record decisions with rationale:**

```
kiseki_stone_add({
  "title": "Decision: <what was decided>",
  "category": "decision",
  "problem": "<the question or tradeoff>",
  "solution": "<what was chosen and why>",
  "tags": "<relevant,tags>"
})
```

### Periodic Memory Sync (Automated)

Workspace memory files are automatically ingested into Kiseki by a scheduled cron job (default: every 6 hours). Session endings and pre-compaction captures are handled by hooks that shell out to `kiseki batch-oc` directly.

**You do NOT need to:**
- Manually call `kiseki_ingest` for workspace memory files (the cron handles this)
- Remember to store context before `/new` or `/reset` (the session-capture hook handles this)
- Worry about losing context on compaction (the compact-capture hook handles this)

**You DO still need to:**
- Create stones (`kiseki_stone_add`) for important preferences, decisions, and fixes as they happen in conversation. These are high-judgment, in-the-moment captures that automation cannot replace.
- Call `kiseki_ingest` when the user explicitly provides a document they want searchable.

### Search Strategy

1. **Search broadly first.** Topics may be stored under different terms than you expect.
2. **Use multiple queries (2-3).** If the first search feels thin, try synonyms or related terms. Example: "email formatting" might also be stored under "report template" or "weekly summary style".
3. **Search for topics, not pronouns.** Use "database migration decision" not "what he said about the database".
4. **Always set limit to 10-20.** Never use the default 250. Large result sets waste context tokens.
5. **Check stones first for known solutions.** If you are looking for a recorded fix, decision, or preference, search stones before doing a general search. Stones are curated and higher-signal.
6. **Use `kiseki_search_msg` for conversation recall.** When the user asks "what did we discuss about X", message search with context gives you the actual conversation flow.

---

## Subagent Memory Retrieval

Memory searches can return large result sets that consume expensive primary-model context. For anything beyond a quick single-query check, **delegate memory retrieval to a subagent** using `sessions_spawn`. The subagent runs on a cheaper model, performs the searches, and announces a processed summary back to you.

**When to delegate to a subagent:**

- Topic research ("what did we discuss about X", "gather context on Y")
- Multi-query searches where you need 2-4 searches from different angles
- Gathering context before starting complex work
- Any search that might return 10+ results needing synthesis
- When you need to verify something but do not want to flood your own context

**When to call Kiseki directly (no subagent needed):**

- `kiseki_status` - quick health check
- A single focused `kiseki_stone_search` for a known preference or solution
- A single `kiseki_search` where you expect 1-3 results and the query is specific
- `kiseki_stone_add` - recording a preference or decision (always do this directly)

**How to spawn a memory retrieval subagent:**

Use the `sessions_spawn` tool. The subagent inherits your workspace (including this skill and all Kiseki tools). Write a clear task that tells the subagent exactly what to search for and how to report back.

```json
{
  "task": "Search Kiseki memory for all context about <topic>. Run multiple searches from different angles (try: '<term1>', '<term2>', '<term3>'). Set limit to 15 per search. Synthesize the findings into a summary: what was decided, what preferences exist, what changed over time, and any gaps. Include dates when available.",
  "label": "memory-retrieval"
}
```

**Task prompt guidelines for memory subagents:**

1. State the topic clearly - what information is needed and why.
2. Suggest specific search terms (2-4 variations) so the subagent searches broadly.
3. Tell the subagent to set `limit` to 15-20 per search.
4. Ask for a structured summary, not a raw dump of chunks.
5. Ask the subagent to flag contradictions or evolution (earlier preference replaced by a later one).
6. Ask the subagent to note what was NOT found (gaps).

**Example - context gathering before a task:**

```json
{
  "task": "I need the user's preferences for weekly reports before generating this week's summary. Search Kiseki for: 'weekly report format', 'report preferences', 'summary style'. Also check stones with category 'config' for report-related entries. Limit 15 per search. Report: (1) formatting preferences, (2) content preferences, (3) delivery preferences, (4) any changes over time.",
  "label": "report-preferences"
}
```

**Example - verifying a past decision:**

```json
{
  "task": "The user mentioned we decided to use Notion for project tracking. Search Kiseki for 'project tracking decision', 'Notion setup', 'task management'. Limit 10 per search. Confirm whether this decision exists in memory, when it was made, and what the reasoning was. If no such decision is found, say so clearly.",
  "label": "fact-check"
}
```

**Important notes:**

- `sessions_spawn` is non-blocking. It returns immediately with `{ status: "accepted", runId }`. The subagent will announce its results when done - do not poll.
- The `model` parameter is optional. If `agents.defaults.subagents.model` is configured in `openclaw.json`, subagents use that model by default.
- Subagents have access to all Kiseki tools because they inherit workspace tools.
- For time-sensitive requests where you need memory context before responding, spawn the subagent and tell the user you are gathering context. Then incorporate the results when the announcement arrives.

---

## Data Safety

Kiseki stores data in a local SQLite database on the machine where it runs. No data is sent to external services (embeddings are generated locally via Ollama). However, local storage still requires care.

**Never store in Kiseki:**
- Passwords, API keys, access tokens, or secrets of any kind
- Financial account numbers, credit card numbers, bank details
- Medical records, health information, or diagnoses
- Social security numbers, government IDs, or similar identifiers
- Any data the user explicitly marks as confidential or temporary

**If the user asks you to "remember" sensitive information:**
Decline politely. Suggest appropriate alternatives (password manager for credentials, encrypted notes for sensitive data). Kiseki is designed for preferences, decisions, and patterns - not for secrets.

**Sanitize before storing:**
When creating stones or ingesting documents, strip out any sensitive data that may be embedded in the content. For example, if a troubleshooting session involved an API key in error logs, record the fix but omit the key from the stone.

---

## Storage Configuration

Kiseki can be configured to capture different levels of conversation data. The tradeoffs:

**User messages only (cleaner, less noise):**
- Captures what the user said and asked for
- Preferences and corrections are directly captured
- Misses assistant-side context (solutions, explanations)
- Best for privacy-conscious setups

**User + assistant messages (fuller picture, more noise):**
- Captures the full conversation flow
- Solutions and explanations are searchable
- More data to search through, more potential noise
- Best for technical workflows where assistant responses contain valuable solutions

**Default recommendation:** Capture user input and record key assistant outputs as stones. This gives you the user's preferences and corrections naturally, while solutions and decisions are explicitly recorded as high-signal stones rather than bulk conversation data.

---

## What NOT to Do

- Do not search Kiseki for things clearly in the current conversation - use your context.
- Do not ingest files on every turn - only when meaningful documents are created or updated.
- Do not create stones for trivial interactions - reserve them for significant preferences, fixes, decisions, and patterns.
- Do not set search limit above 20 unless specifically asked - large result sets waste context tokens.
- Do not confuse Kiseki with OpenClaw's `memory_search` - they search different data stores.
- Do not store sensitive data (passwords, tokens, financial details) in Kiseki. Ever.
- Do not bulk-ingest transient content (news, weather, stock prices). Store the user's preferences about these, not the content itself.
- Do not over-promote importance tiers. Most items are `normal`. Only real fixes are `solution`, only real decisions and preferences are `key`.
