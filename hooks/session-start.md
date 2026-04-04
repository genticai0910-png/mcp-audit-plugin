---
name: mcp-health-check-on-start
event: SessionStart
description: Quick MCP health check on session start — warns about high tool count, unresponsive servers, or known performance issues.
---

Run a quick MCP health check at session start. This should be fast (< 2 seconds) and non-blocking.

## Check these things silently:

1. **Tool count**: Count the total MCP tools available. If > 100, add a note:
   "MCP Health: {{count}} tools loaded across {{server_count}} servers. Consider scoping with project-level .mcp.json to reduce context size. Run /mcp-audit for details."

2. **Known issues**: If the user's Claude Code version doesn't include the tool-sorting fix, note:
   "MCP Health: Tool ordering may not be deterministic — this can bust prompt caching. Run /mcp-audit for details."

3. **If everything looks healthy**: Say nothing. Don't add noise to clean sessions.

## Output rules:
- Only speak if there's an actionable issue
- Keep it to one line
- Always reference /mcp-audit for details
- Never block the session — this is informational only
