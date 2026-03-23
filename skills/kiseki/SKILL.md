---
name: kiseki
description: "Persistent cross-session memory via Kiseki: search past conversations, record decisions, track solutions across sessions. Use when the user references prior work, when starting complex tasks, or when recording important outcomes."
metadata: { "openclaw": { "emoji": "🧠", "requires": { "bins": ["kiseki"] }, "install": [{ "id": "go", "kind": "go", "module": "github.com/Gsirawan/kiseki-beta@latest", "bins": ["kiseki"], "label": "Install Kiseki (go install)", "buildTags": "fts5" }] } }
---

# Kiseki Memory

You have access to **persistent cross-session memory** through Kiseki MCP tools. Kiseki stores conversations, notes, and documents in a local SQLite database with multi-layer search (FTS5 + vector search on chunks + vector search on messages). Results are ranked by importance tier and layer overlap.

**This is different from OpenClaw's built-in `memory_search`.** OpenClaw's memory searches workspace markdown files (`MEMORY.md`, `memory/*.md`). Kiseki searches ingested documents and captured conversation history across all sessions -- it is your long-term recall.

| OpenClaw `memory_search` | Kiseki `kiseki_search` |
|---|---|
| Current workspace markdown files | All ingested docs + conversation history |
| Single-session scope | Cross-session, cross-project |
| Text chunks from `.md` files | Multi-layer: FTS5 + vector chunks + vector messages |
| No importance ranking | Importance tiers: stone > solution > key > normal |

**Use both.** OpenClaw memory for recent workspace notes. Kiseki for historical recall.

---

## Available Tools

### kiseki_search

Multi-layer search across all memories. Returns chunks and messages ranked by layer overlap and similarity.

**When to use:**
- User references past conversations ("last time", "we discussed", "remember when")
- Before starting complex work on a topic that may have prior context
- Looking for a decision, fix, or approach from earlier sessions
- Checking if a problem was solved before

**Parameters:**
- `query` (required): Search query -- be specific and topical
- `limit` (optional): Maximum results. **Always set to 10-20.** Default is 250 which floods context.
- `as_of` (optional): ISO date to filter results before a date

**Example:**
```json
{ "query": "auth module OAuth token refresh", "limit": 15 }
```

### kiseki_search_msg

Search messages with conversation context window. Returns snippets of actual conversations around matching messages.

**When to use:**
- Looking for a specific discussion or phrase
- Need to see the conversation flow around a topic, not just isolated chunks
- User asks "what did we talk about when..."

**Parameters:**
- `query` (required): Search query
- `limit` (optional): Maximum results. **Set to 10-15.**
- `fts` (optional): Use exact phrase matching instead of semantic search
- `context` (optional): Context window in minutes around each match (default 3)

**Example:**
```json
{ "query": "database migration decision", "limit": 10, "context": 5 }
```

### kiseki_stone_add

Create a named record for an important solution, decision, important configuration, user preference, or pattern. Stones are permanent, searchable, and rank above all other results.

**When to use:**
- After solving a hard problem -- record the fix
- After making an architecture or design decision
- When discovering a pattern or workaround worth preserving
- When the user states a preference or important configuration
- User explicitly asks to save something for future reference

**Parameters:**
- `title` (required): Descriptive title
- `category` (optional): `fix`, `decision`, `pattern`, `workaround`, `config`
- `problem` (optional): What the problem was
- `solution` (optional): What solved it
- `tags` (optional): Comma-separated tags for filtering

**Example:**
```json
{
  "title": "Fix: OAuth token refresh race condition",
  "category": "fix",
  "problem": "Concurrent requests caused duplicate token refresh calls, leading to 401 errors",
  "solution": "Added mutex lock around refresh logic with sync.Once pattern",
  "tags": "auth,oauth,concurrency,go"
}
```

### kiseki_stone_search

Search existing stones by title, tags, category, or week.

**When to use:**
- Looking for a known solution or decision
- Checking if a similar problem was solved before
- Reviewing decisions made during a specific period

**Parameters:**
- `query` (optional): Text search across title, tags, problem, solution
- `category` (optional): Filter by category
- `week` (optional): Filter by ISO week (e.g., `2026-W12`)
- `limit` (optional): Maximum results. **Set to 10-20.**

### kiseki_history

Chronological timeline of all mentions of an entity (person, project, component, tool).

**When to use:**
- Tracing the evolution of a topic over time
- Getting full context on a person or project
- Understanding when and how decisions changed

**Parameters:**
- `entity` (required): Entity name to trace
- `limit` (optional): Maximum results. **Set to 20.**

### kiseki_ingest

Ingest a markdown file into Kiseki's memory store. The file is chunked, embedded, and indexed.

**When to use:**
- After creating important documentation
- Periodically ingest OpenClaw's own memory files (`memory/*.md`, `MEMORY.md`) so Kiseki has them
- When the user provides a document they want searchable across sessions
- At session end if significant work was documented

**Parameters:**
- `file_path` (required): Path to the markdown file
- `valid_at` (optional): ISO date stamp for the content

**Important:** Requires `KISEKI_INGEST_ROOT` env var to be set for absolute paths.

### kiseki_status

Check Kiseki system health: database stats, embedding model status, chunk/message counts.

**When to use:**
- Diagnosing search issues (is the database empty?)
- Verifying Kiseki is operational
- Checking how much memory is stored

---

## Usage Patterns

### At Session Start

If the session involves continuing prior work, search Kiseki for context:

```
kiseki_search({ "query": "<topic from user's first message>", "limit": 15 })
```

This gives you historical context before diving in. Do not do this for clearly new/unrelated topics.

### When User References the Past

Any of these signals should trigger a Kiseki search:
- "last time", "we discussed", "remember when", "previously"
- "the fix for", "that bug with", "the decision about"
- "what did we decide", "how did we solve"

Run 2-3 searches from different angles if the first one does not return strong results.

### After Solving Problems

Record the solution as a stone:

```
kiseki_stone_add({
  "title": "Fix: <concise description>",
  "category": "fix",
  "problem": "<what was wrong>",
  "solution": "<what fixed it>",
  "tags": "<relevant,tags>"
})
```

### After Making Decisions

Record architecture and design decisions:

```
kiseki_stone_add({
  "title": "Decision: <what was decided>",
  "category": "decision",
  "problem": "<the question or tradeoff>",
  "solution": "<what was chosen and why>",
  "tags": "<relevant,tags>"
})
```

### Periodic Memory Sync

When significant work is documented in OpenClaw's workspace memory files, ingest them into Kiseki so they are searchable across sessions:

```
kiseki_ingest({ "file_path": "<workspace>/memory/2026-03-22.md" })
```

### Search Strategy

1. **Search broadly first.** Topics may be stored under different terms.
2. **Use multiple queries (2-3).** If the first search feels thin, try synonyms or related terms.
3. **Search for topics, not pronouns.** "database migration" not "what he said about the database".
4. **Always set limit to 10-20.** Never use the default 250 -- it floods your context.
5. **Use stone_search for known solutions.** If you are looking for a recorded fix or decision, search stones first.

### Subagent Memory Retrieval

Memory searches can return large result sets that consume expensive primary-model context. For anything beyond a quick single-query check, **delegate memory retrieval to a subagent** using `sessions_spawn`. The subagent runs on a cheaper model (configured via `agents.defaults.subagents.model`), performs the searches, and announces a processed summary back to you.

**When to delegate to a subagent:**

- Topic research ("what did we discuss about X", "gather context on Y")
- Multi-query searches where you need 2-4 searches from different angles
- Gathering context before starting complex work
- Any search that might return 10+ results needing synthesis
- When you are unsure about facts or need to verify something before continuing

**When to call Kiseki directly (no subagent needed):**

- `kiseki_status` -- quick health check
- A single focused `kiseki_stone_search` for a known solution
- A single `kiseki_search` where you expect 1-3 results and the query is specific

**How to spawn a memory retrieval subagent:**

Use the `sessions_spawn` tool. The subagent inherits your workspace (including this skill and all Kiseki tools). Write a clear task that tells the subagent exactly what to search for and how to report back.

```json
{
  "task": "Search Kiseki memory for all context about <topic>. Run multiple searches from different angles (try: '<term1>', '<term2>', '<term3>'). Set limit to 15 per search. Synthesize the findings into a summary: what was decided, what changed over time, what the current state is, and any gaps. Include dates when available.",
  "label": "memory-retrieval",
  "model": "anthropic/claude-sonnet-4-20250514"
}
```

**Task prompt guidelines for memory subagents:**

1. State the topic clearly -- what information is needed and why.
2. Suggest specific search terms (2-4 variations) so the subagent searches broadly.
3. Tell the subagent to set `limit` to 15-20 per search.
4. Ask for a structured summary, not a raw dump of chunks.
5. Ask the subagent to flag contradictions or evolution (earlier decision changed later).
6. Ask the subagent to note what was NOT found (gaps).

**Example -- context gathering before complex work:**

```json
{
  "task": "I need full context on the auth module before making changes. Search Kiseki for: 'auth module OAuth', 'token refresh', 'authentication architecture', 'JWT session'. Limit 15 per search. Report: (1) current auth architecture and key decisions, (2) any known bugs or workarounds, (3) what changed over time, (4) gaps in memory.",
  "label": "auth-context"
}
```

**Example -- verifying a fact before responding:**

```json
{
  "task": "The user mentioned we decided to use PostgreSQL for the new service. Search Kiseki for 'database choice', 'PostgreSQL decision', 'database migration'. Limit 10 per search. Confirm whether this decision exists in memory, when it was made, and what the reasoning was. If no such decision is found, say so clearly.",
  "label": "fact-check"
}
```

**Important notes:**

- `sessions_spawn` is non-blocking. It returns immediately with `{ status: "accepted", runId }`. The subagent will announce its results back to you when done -- do not poll.
- The `model` parameter is optional. If `agents.defaults.subagents.model` is configured in `openclaw.json`, subagents will use that model by default. Only pass `model` if you need to override the default for a specific retrieval.
- Subagents have access to all Kiseki tools (`kiseki_search`, `kiseki_search_msg`, `kiseki_stone_search`, `kiseki_history`, etc.) because they inherit workspace tools.
- For time-sensitive requests where you need memory context before you can respond to the user, spawn the subagent and tell the user you are gathering context. Then incorporate the results when the announce arrives.

---

## What NOT to Do

- Do not search Kiseki for things clearly in the current conversation -- use your context.
- Do not ingest files on every turn -- only when meaningful documents are created or updated.
- Do not create stones for trivial changes -- reserve them for significant fixes, decisions, and patterns.
- Do not set search limit above 20 unless specifically asked -- large result sets waste context tokens.
- Do not confuse Kiseki with OpenClaw's `memory_search` -- they search different data stores.
