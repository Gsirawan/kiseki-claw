# kiseki-claw Installation Guide (For Your OpenClaw Agent)

Copy this file and send it to your OpenClaw agent. It will set up persistent cross-session memory for you.

---

## Step 1: Check Prerequisites

Verify these are available on the system:

```bash
go version
```
Requires Go 1.23 or later. If missing, stop and tell the user to install Go.

```bash
gcc --version
```
Required for CGo/SQLite compilation. If missing, stop and tell the user to install GCC.

```bash
curl -s http://127.0.0.1:11434/api/tags
```
Ollama must be running. If this fails, stop and tell the user to install and start Ollama.

If any prerequisite is missing, **stop and report which ones need to be installed.** Do not continue.

---

## Step 2: Install Kiseki Binary

Check if kiseki is already installed:

```bash
which kiseki && kiseki status
```

If `kiseki` is found and `kiseki status` shows healthy, skip to Step 3.

If not installed, run:

```bash
CGO_ENABLED=1 go install -tags fts5 github.com/Gsirawan/kiseki-beta@latest
```

Then pull the embedding model:

```bash
ollama pull qwen3-embedding:0.6b
```

Verify the install:

```bash
kiseki status
```

If `kiseki status` shows healthy, continue to Step 3. If it reports errors, **stop and report the error output to the user.**

---

## Step 3: Install the Plugin

Clone the repository to the extensions directory:

```bash
git clone https://github.com/Gsirawan/kiseki-claw.git ~/.openclaw/extensions/kiseki-claw
```

If the directory already exists, pull latest instead:

```bash
cd ~/.openclaw/extensions/kiseki-claw && git pull
```

Enable the plugin in `~/.openclaw/openclaw.json`. **Merge** this into the existing config -- do not overwrite the file:

```json
{
  "plugins": {
    "entries": {
      "kiseki-claw": { "enabled": true }
    }
  }
}
```

If `openclaw.json` already has a `plugins.entries` section, add the `kiseki-claw` key to it. Do not remove existing entries.

---

## Step 4: Configure Environment

Add Kiseki's MCP server environment variables to `~/.openclaw/openclaw.json` under `mcp.servers.kiseki.env`. **Merge** into the existing config -- do not overwrite:

```json
{
  "mcp": {
    "servers": {
      "kiseki": {
        "command": "kiseki",
        "args": ["serve"],
        "env": {
          "KISEKI_DB": "~/.local/share/kiseki/memory.db",
          "KISEKI_PREFIX": "kiseki",
          "EMBED_MODEL": "qwen3-embedding:0.6b",
          "OLLAMA_HOST": "127.0.0.1:11434",
          "KISEKI_INGEST_ROOT": "~/.openclaw/workspace"
        }
      }
    }
  }
}
```

**IMPORTANT:** `OLLAMA_HOST` must be `host:port` only. Do NOT include `http://` prefix. Kiseki prepends the scheme internally, so including it would produce a malformed URL.

**IMPORTANT:** `KISEKI_INGEST_ROOT` is required for the cron job and any `kiseki_ingest` calls that use absolute file paths. Without it, ingestion of files like `MEMORY.md` will be silently rejected. Set it to the workspace root (`~/.openclaw/workspace`).

If `openclaw.json` already has an `mcp.servers` section, add the `kiseki` key to it. Do not remove existing server entries.

**Note for bundle plugin users (Method C in README):** If you installed kiseki-claw as a bundle plugin instead of using config-level MCP, the bundled `.mcp.json` provides default env values. To override them, add your customizations under `skills.entries.kiseki.env` in `openclaw.json`. Values in `skills.entries.*.env` are injected into the process environment and override the bundled defaults.

---

## Step 5: Verify Skill Deployed

Check the skill file exists:

```bash
ls ~/.openclaw/workspace/skills/kiseki/SKILL.md
```

If the file exists, continue to Step 6.

If missing, copy it from the cloned extension:

```bash
mkdir -p ~/.openclaw/workspace/skills
cp -r ~/.openclaw/extensions/kiseki-claw/skills/kiseki ~/.openclaw/workspace/skills/
```

Verify the copy:

```bash
ls ~/.openclaw/workspace/skills/kiseki/SKILL.md
```

If the file still does not exist, **stop and report the error to the user.**

---

## Step 6: Seed Workspace Memory

The cron job (Step 8) ingests markdown files from `~/.openclaw/workspace/memory/` into Kiseki. Without at least one file there, the first cron run has nothing to ingest.

Create the memory directory and seed a `MEMORY.md`:

```bash
mkdir -p ~/.openclaw/workspace/memory
```

Create `~/.openclaw/workspace/memory/MEMORY.md` with initial context. Ask the user what to include -- at minimum, write a file like:

```markdown
# Memory

## About
- Name: [user's name]
- Date started: [today's date]

## Notes
[Any initial context the user wants the agent to remember across sessions.]
```

The user can add more files to `~/.openclaw/workspace/memory/` at any time. The cron job will pick up any `.md` files modified in the last 6 hours and ingest them automatically.

If the user has no specific context to seed, create the file with just the header and date. An empty directory is fine too -- the cron will simply report 0 files ingested until content is added.

---

## Step 7: Create Capture Hooks

Create two hooks for automatic memory capture. These ensure conversations are captured to kiseki without relying on the agent to remember.

### Hook 1: Pre-compaction capture

This hook fires before OpenClaw compacts a conversation, capturing the session into kiseki so context is not lost.

Create the directory:

```bash
mkdir -p ~/.openclaw/hooks/kiseki-compact-capture
```

Create `~/.openclaw/hooks/kiseki-compact-capture/HOOK.md` with this exact content:

```markdown
---
name: kiseki-compact-capture
description: "Captures conversation to kiseki before compaction discards context"
metadata: { "openclaw": { "emoji": "brain", "events": ["session:compact:before"], "requires": { "bins": ["kiseki"] } } }
---

Captures the current session into kiseki memory before compaction summarizes and discards conversation history.
```

Create `~/.openclaw/hooks/kiseki-compact-capture/handler.ts` with this exact content:

```typescript
import { spawnSync } from "child_process";

const handler = async (event: any) => {
  if (event.type !== "session" || event.action !== "compact:before") return;

  const sessionId = event.context?.sessionId || event.sessionKey;
  if (!sessionId) {
    console.error("[kiseki-compact-capture] No session ID found in event context");
    return;
  }

  const env = {
    ...process.env,
    KISEKI_DB: process.env.KISEKI_DB || "~/.local/share/kiseki/memory.db",
    OLLAMA_HOST: process.env.OLLAMA_HOST || "127.0.0.1:11434",
    EMBED_MODEL: process.env.EMBED_MODEL || "qwen3-embedding:0.6b",
  };

  const result = spawnSync("kiseki", ["batch-oc", "--session", sessionId], {
    env,
    timeout: 60000,
    encoding: "utf-8",
  });

  if (result.status !== 0) {
    console.error(
      `[kiseki-compact-capture] batch-oc failed (exit ${result.status}): ${result.stderr}`
    );
  }
};

export default handler;
```

### Hook 2: Session-end capture

This hook fires when the user starts a new session (`/new`) or resets (`/reset`), capturing the previous session.

Create the directory:

```bash
mkdir -p ~/.openclaw/hooks/kiseki-session-capture
```

Create `~/.openclaw/hooks/kiseki-session-capture/HOOK.md` with this exact content:

```markdown
---
name: kiseki-session-capture
description: "Captures session to kiseki on /new and /reset"
metadata: { "openclaw": { "emoji": "brain", "events": ["command:new", "command:reset"], "requires": { "bins": ["kiseki"] } } }
---

Captures the completed session into kiseki memory when the user starts a new session or resets.
```

Create `~/.openclaw/hooks/kiseki-session-capture/handler.ts` with this exact content:

```typescript
import { spawnSync } from "child_process";

const handler = async (event: any) => {
  if (event.type !== "command") return;
  if (event.action !== "new" && event.action !== "reset") return;

  const sessionId =
    event.context?.previousSessionEntry?.id ||
    event.context?.previousSessionId ||
    event.context?.sessionId;
  if (!sessionId) {
    console.error("[kiseki-session-capture] No previous session ID found in event context");
    return;
  }

  const env = {
    ...process.env,
    KISEKI_DB: process.env.KISEKI_DB || "~/.local/share/kiseki/memory.db",
    OLLAMA_HOST: process.env.OLLAMA_HOST || "127.0.0.1:11434",
    EMBED_MODEL: process.env.EMBED_MODEL || "qwen3-embedding:0.6b",
  };

  const result = spawnSync("kiseki", ["batch-oc", "--session", sessionId], {
    env,
    timeout: 60000,
    encoding: "utf-8",
  });

  if (result.status !== 0) {
    console.error(
      `[kiseki-session-capture] batch-oc failed (exit ${result.status}): ${result.stderr}`
    );
  }
};

export default handler;
```

### Enable both hooks

```bash
openclaw hooks enable kiseki-compact-capture
openclaw hooks enable kiseki-session-capture
```

Verify they are enabled:

```bash
openclaw hooks list
```

Both `kiseki-compact-capture` and `kiseki-session-capture` should appear as enabled. If either fails to enable, **stop and report the error to the user.**

---

## Step 8: Create Cron Jobs

Set up automated memory sync and health monitoring using the `openclaw cron add` command. These are OpenClaw-managed cron jobs -- they run inside the gateway process, not the system crontab.

### Workspace memory sync (every 6 hours)

This cron job runs every 6 hours (at 00:00, 06:00, 12:00, 18:00). It scans `~/.openclaw/workspace/memory/` for `.md` files modified in the last 6 hours and ingests them into Kiseki via `kiseki_ingest`. This is how `MEMORY.md` and any other workspace memory files get into Kiseki's searchable index automatically.

```bash
openclaw cron add \
  --name "kiseki-ingest-memory" \
  --cron "0 */6 * * *" \
  --session isolated \
  --message "Periodic kiseki memory sync. List the memory/ directory in the workspace. For each .md file modified in the last 6 hours, call kiseki_ingest with the file_path and today's date as valid_at. Skip files that have not changed. Report the count of files ingested." \
  --light-context \
  --no-announce
```

### Health check (daily)

This cron job runs once daily at 08:00. It calls `kiseki_status` to verify Ollama is healthy and the database has data. If anything is wrong, it announces the issue. This is your early warning system for silent failures.

```bash
openclaw cron add \
  --name "kiseki-health-check" \
  --cron "0 8 * * *" \
  --session isolated \
  --message "Call kiseki_status. If OllamaHealthy is false or chunk count is 0, report the issue. Otherwise reply with just HEARTBEAT_OK." \
  --light-context \
  --announce
```

### Verify cron jobs

Confirm both jobs were created:

```bash
openclaw cron list
```

Both `kiseki-ingest-memory` and `kiseki-health-check` should appear in the output. The list shows each job's name, schedule, and next run time.

To verify a cron job will actually work, you can trigger a manual run:

```bash
openclaw cron run kiseki-health-check
```

This runs the health check immediately. If it succeeds and returns `HEARTBEAT_OK` (or reports a real issue), the cron is configured correctly. If it fails with a tool error, the MCP server is not connected -- go back to Step 4 and verify the configuration.

If either cron job fails to create, **stop and report the error to the user.**

---

## Step 9: Restart and Verify

Restart the gateway to pick up all changes:

```bash
openclaw gateway restart
```

Then verify each component:

**1. Kiseki health:**

```bash
kiseki status
```

Should show healthy with Ollama connected.

**2. Plugin loaded:**

```bash
openclaw plugins list
```

Should show `kiseki-claw` as enabled.

**3. Memory tools available:**

Call `kiseki_search` with a test query:

```json
{ "query": "test", "limit": 5 }
```

This should return results (or an empty result set if the database is fresh). If it returns an error about the tool not being found, the MCP server is not connected -- check that the plugin is enabled and the gateway was restarted.

**4. Hooks active:**

```bash
openclaw hooks list
```

Should show both `kiseki-compact-capture` and `kiseki-session-capture` as enabled.

**5. Cron jobs scheduled:**

```bash
openclaw cron list
```

Should show both `kiseki-ingest-memory` and `kiseki-health-check`.

---

## Done

Report to the user:

> Kiseki persistent memory is installed and running. Your conversations will be automatically captured across sessions. Pre-compaction and session-end hooks ensure nothing is lost, and a cron job syncs workspace memory files every 6 hours. A daily health check monitors the system.
>
> Available memory tools: `kiseki_search`, `kiseki_search_msg`, `kiseki_stone_add`, `kiseki_stone_search`, `kiseki_history`, `kiseki_ingest`, `kiseki_status`.

---

## Troubleshooting

**`kiseki status` shows Ollama unhealthy:**
- Verify Ollama is running: `curl -s http://127.0.0.1:11434/api/tags`
- Verify the embedding model is pulled: `ollama list` should show `qwen3-embedding:0.6b`
- Check that `OLLAMA_HOST` does not include `http://` prefix

**`kiseki_search` tool not found:**
- Verify the plugin is enabled: `openclaw plugins list`
- Verify the skill file exists: `ls ~/.openclaw/workspace/skills/kiseki/SKILL.md`
- Restart the gateway: `openclaw gateway restart`

**Hooks not firing:**
- Verify hooks are enabled: `openclaw hooks list`
- Check handler.ts syntax -- TypeScript errors prevent hook execution
- Check that `kiseki` binary is on PATH

**Cron jobs not running:**
- Verify jobs exist: `openclaw cron list`
- Check that the gateway is running (cron jobs are managed by the gateway process)
