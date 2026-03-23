# kiseki-claw

Persistent cross-session memory for [OpenClaw](https://github.com/openclaw/openclaw) via [Kiseki](https://github.com/Gsirawan/kiseki-beta).

## What It Does

Agents forget everything after compaction. Kiseki fixes that. It gives your OpenClaw agent persistent, cross-session memory with multi-layer search (FTS5 full-text + vector similarity on chunks and messages), importance-ranked results, solution/decision records, and entity timeline tracking. Seven MCP tools are exposed to the agent through a single SKILL.md that teaches it when and how to use each one.

## Prerequisites

- **Go 1.23+** - for building Kiseki
- **Ollama** running with an embedding model
- **GCC** - required for SQLite CGo compilation
- **Kiseki binary on PATH**

```bash
# Install Kiseki (requires GCC for CGo)
CGO_ENABLED=1 go install -tags fts5 github.com/Gsirawan/kiseki-beta@latest

# Pull an embedding model
ollama pull qwen3-embedding:0.6b

# Verify both are working
kiseki status
```

## Installation

Three methods, ordered from simplest to most portable.

### Method A: Config-Level MCP (Simplest)

Add the MCP server directly to your OpenClaw config. No plugin install needed.

> **Note:** Requires OpenClaw built from source or a version that supports the `mcp.servers` config key. If OpenClaw rejects the key, use Method B or C instead.

**Step 1.** Add `mcp.servers.kiseki` to `~/.openclaw/openclaw.json`:

```json
{
  "mcp": {
    "servers": {
      "kiseki": {
        "command": "kiseki",
        "args": ["serve"],
        "env": {
          "KISEKI_DB": "/path/to/your/kiseki.db",
          "KISEKI_PREFIX": "kiseki",
          "EMBED_MODEL": "qwen3-embedding:0.6b",
          "OLLAMA_HOST": "localhost:11434"
        }
      }
    }
  }
}
```

**Step 2.** Copy the skill to your workspace:

```bash
# Clone the repo
git clone https://github.com/Gsirawan/kiseki-claw.git

# Copy skill into your workspace
cp -r kiseki-claw/skills/kiseki ~/.openclaw/workspace/skills/
```

**Step 3.** Restart the gateway and verify:

```bash
openclaw gateway restart
```

Start a new session. The agent should have `kiseki_search` and other tools available.

---

### Method B: mcporter (Works with Any OpenClaw Version)

[mcporter](https://github.com/nicepkg/mcporter) registers MCP servers without touching OpenClaw internals. Works with any version.

**Step 1.** Install mcporter and register Kiseki:

```bash
npm i -g mcporter

mcporter config add kiseki \
  --command "kiseki" \
  --arg "serve" \
  --env KISEKI_DB=/path/to/your/kiseki.db \
  --env KISEKI_PREFIX=kiseki \
  --env EMBED_MODEL=qwen3-embedding:0.6b \
  --env OLLAMA_HOST=localhost:11434 \
  --scope home
```

**Step 2.** Enable the mcporter skill in OpenClaw if prompted.

**Step 3.** Copy the SKILL.md to your workspace:

```bash
git clone https://github.com/Gsirawan/kiseki-claw.git
cp -r kiseki-claw/skills/kiseki ~/.openclaw/workspace/skills/
```

**Step 4.** Restart and verify:

```bash
openclaw gateway restart
```

---

### Method C: Bundle Plugin

Install as a Claude-compatible bundle. OpenClaw auto-detects the `.claude-plugin/plugin.json` manifest and maps the MCP config and skills together.

**Step 1.** Clone and copy to extensions:

```bash
git clone https://github.com/Gsirawan/kiseki-claw.git
cp -r kiseki-claw ~/.openclaw/extensions/kiseki-claw
```

**Step 2.** Enable the plugin in `~/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "kiseki-claw": { "enabled": true }
    }
  }
}
```

**Step 3.** (Optional) Override default env vars. The bundled `.mcp.json` ships with portable defaults (`kiseki.db` for the database, `localhost:11434` for Ollama). To customize:

```json
{
  "skills": {
    "entries": {
      "kiseki": {
        "enabled": true,
        "env": {
          "KISEKI_DB": "/path/to/your/kiseki.db",
          "OLLAMA_HOST": "localhost:11434"
        }
      }
    }
  }
}
```

Values in `skills.entries.*.env` are injected into the process environment and override the defaults in `.mcp.json`.

**Step 4.** Restart the gateway:

```bash
openclaw gateway restart
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|---|---|---|
| `KISEKI_DB` | `kiseki.db` | Path to the Kiseki SQLite database |
| `OLLAMA_HOST` | `localhost:11434` | Ollama server address (host:port, **no** `http://` prefix) |
| `EMBED_MODEL` | `qwen3-embedding:0.6b` | Ollama embedding model name |
| `KISEKI_PREFIX` | `kiseki` | MCP tool name prefix (`kiseki` produces `kiseki_search`, etc.) |
| `KISEKI_ENTITIES` | - | Path to entities YAML file for alias expansion |
| `KISEKI_INGEST_ROOT` | - | Required base path for ingesting files via absolute paths |

**Important:** `OLLAMA_HOST` must be `host:port` only. Do not include `http://`. Kiseki prepends the scheme internally, so `http://localhost:11434` would result in a malformed URL.

### Database Location

Choose a path that fits your setup:

- **Single agent, single machine:** `~/.local/share/kiseki/memory.db`
- **Per-project memory:** `./project/.kiseki/memory.db`
- **Shared across agents:** Any common path all agents can access

### Entity Graph (Optional)

Define entities with aliases so searches expand automatically:

```yaml
# entities.yaml
entities:
  - name: PostgreSQL
    aliases: [postgres, pg, psql]
  - name: React
    aliases: [ReactJS, react.js]
```

Set `KISEKI_ENTITIES=/path/to/entities.yaml` in your MCP server env.

## Exposed Tools

The SKILL.md exposes 7 of Kiseki's 18 MCP tools, chosen for agent usefulness:

| Tool | Purpose |
|---|---|
| `kiseki_search` | Multi-layer search (FTS5 + vector chunks + vector messages) |
| `kiseki_search_msg` | Search messages with conversation context window |
| `kiseki_stone_add` | Record solutions, decisions, patterns as permanent entries |
| `kiseki_stone_search` | Find recorded solutions and decisions |
| `kiseki_history` | Chronological entity timeline |
| `kiseki_ingest` | Ingest markdown files into the memory store |
| `kiseki_status` | System health and database stats |

The remaining 11 tools are still available via MCP but not documented in SKILL.md to keep the agent focused. Advanced users can call them directly.

## What the Agent Learns

The SKILL.md teaches the agent:

1. **When to search** - user references past work, session start on continuing topics, before complex tasks
2. **When to record** - after solving problems, making decisions, discovering patterns worth preserving
3. **When to ingest** - after creating documentation, periodic sync of workspace memory files
4. **How to disambiguate** - Kiseki for cross-session historical recall vs. OpenClaw's `memory_search` for current workspace files
5. **Search discipline** - always cap results at 10-20, use multiple queries from different angles, search topics not pronouns
6. **Subagent delegation** - offload memory retrieval to a cheaper subagent to avoid flooding the primary model's context

## Architecture

```
+-------------------+
|  OpenClaw Agent   |
|  (any model)      |
+--------+----------+
         |
         |  MCP (stdio)
         |  kiseki_search, kiseki_stone_add, ...
         |
+--------v----------+
|  kiseki serve     |
|  (MCP subprocess) |
+--------+----------+
         |
    +----+----+
    |         |
+---v---+ +---v----+
| SQLite| | Ollama |
| + vec | | (local)|
+-------+ +--------+
```

Kiseki runs as a stdio subprocess managed by OpenClaw. No network services, no cloud dependencies. Everything stays local.

## Roadmap

**v1 (current):** MCP server + SKILL.md. Zero integration code. Agent-driven search and recording.

**v2 (planned):** Hook integration.
- Compaction hook: auto-ingest conversation summary before compaction
- Session start hook: auto-search for context on session topics
- Standing orders: periodic ingestion of workspace memory files

**v3 (future):** Context engine plugin. Kiseki-backed context assembly with automatic relevance injection into the agent's prompt.

## License

[MIT](LICENSE)
