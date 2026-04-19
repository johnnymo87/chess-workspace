# chess

Conversational chess analysis workspace. Human as player, LLM as partner,
MCP as scratchpad. No engines.

See `AGENTS.md` for philosophy and tool usage.
See `games/` for PGNs under analysis.

## Activating the workspace in OpenCode

Launch OpenCode from this directory. `opencode.json` wires up the
`chess-support-mcp` server via `uvx`, which pulls and caches the server
from GitHub on first run and starts instantly thereafter.

First run will take ~30 seconds to build. Subsequent runs start instantly.
