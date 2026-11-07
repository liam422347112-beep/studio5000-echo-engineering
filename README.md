# studio5000-echo-engineering

Cursor Agent Skill for **Studio 5000 Logix Designer v35** program editing and **FactoryTalk Logix Echo** testing via the `studio5000-mcp` toolchain (L5X edit → GUI import/verify → Echo snapshot/download → CIP tag prove-out).

Validated end-to-end on a PackML machine project (JOYO / Damin-style): import/verify/download, PackML Idle → Execute with master running, then prove a new EM (e.g. sealing-down LIM±Win → time-based MAM).

## What this skill covers

- Safe Echo-only CIP writes and pre-download snapshots
- **No-EDS pre-flight gate** before any L5X export/import (user chooses strip vs install EDS)
- License-free GUI path: export → L5X tools → retarget `1756-L85E` → import → verify → Echo download
- PackML online order: do not bypass state machine to “make tags move”
- New EM clone pitfalls (StartExecutionDone deadlock, `OTL` motion triggers, Selected-module gating)

## Install into Cursor

### Option A — Personal skill (all projects)

Copy the skill folder into your Cursor user Agent Store skills directory:

```text
skills/studio5000-echo-engineering/SKILL.md
```

On this machine that is typically under:

```text
%LOCALAPPDATA%\Cursor\AgentStores\cursor_agent_stores\u529694454\files\skills\
```

Or clone and copy:

```powershell
git clone https://github.com/liam422347112-beep/studio5000-echo-engineering.git
Copy-Item -Recurse .\studio5000-echo-engineering\skills\studio5000-echo-engineering `
  "$env:LOCALAPPDATA\Cursor\AgentStores\cursor_agent_stores\u529694454\files\skills\"
```

### Option B — Project skill

Copy into the repo you are working in:

```text
.cursor/skills/studio5000-echo-engineering/SKILL.md
```

This repository already includes that path for convenience.

### Invoke

In Cursor Agent chat, name the skill or work on ACD/L5X/Echo/PackML topics so the agent loads it. Companion MCP server must be configured (see below).

## Companion MCP: studio5000-mcp

This skill assumes the `studio5000-mcp` Python MCP server is available (tools such as `gui_export_l5x`, `gui_import_l5x`, `gui_verify`, `echo_snapshot`, `echo_download`, `echo_read`, `echo_write`).

Example Cursor MCP config:

```json
{
  "mcpServers": {
    "studio5000-echo": {
      "command": "py",
      "args": ["-3.12", "-m", "studio5000_mcp"],
      "cwd": "d:/path/to/studiomcp",
      "env": {
        "ECHO_IP": "127.0.0.1",
        "ECHO_SLOT": "0"
      }
    }
  }
}
```

Requires Windows + Studio 5000 Logix Designer v35 + FactoryTalk Logix Echo for the full GUI/Echo path.

## Hard rules (summary)

1. **No-EDS first** — if third-party Ethernet modules lack local EDS, stop and ask: temporary strip for Echo, or user installs EDS, or cancel. Never silent-strip.

Full agent instructions: [`skills/studio5000-echo-engineering/SKILL.md`](skills/studio5000-echo-engineering/SKILL.md)  
Extended notes: [`docs/reference.md`](docs/reference.md)

## License

MIT — see [LICENSE](LICENSE).
