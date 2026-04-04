---
name: mcp-fix
description: Fix MCP configuration issues found by /mcp-audit. Backs up config before making changes. Use when user says "fix mcp", "mcp fix", or after running /mcp-audit.
---

# MCP Configuration Fix

Apply fixes for issues found by the MCP audit. Always backs up before modifying.

## Steps

1. **Run the audit first** — invoke the /mcp-audit command to identify current issues. If the user already ran it this session, use those results.

2. **Back up current config** before any changes:
   ```bash
   cp ~/.claude/settings.json ~/.claude/settings.json.bak-mcp-fix-$(date +%Y%m%d-%H%M%S) 2>/dev/null
   cp .mcp.json .mcp.json.bak-mcp-fix-$(date +%Y%m%d-%H%M%S) 2>/dev/null
   ```

3. **Present fixable issues** to the user and ask which to apply:

   ### Auto-fixable Issues
   - **Disable stale servers**: Comment out or remove servers that fail to connect
   - **Remove duplicate tools**: If two servers provide the same tool, recommend keeping the more reliable one
   - **Reduce context bloat**: Suggest disabling servers with > 50 tools that aren't actively used in the current project

   ### Manual-fix Issues (provide instructions)
   - **Tool ordering**: Requires updating Claude Code itself or waiting for the upstream fix. Provide the one-liner: `tools: tools.toSorted((a, b) => a.name.localeCompare(b.name))`
   - **Server splitting**: Recommend breaking large MCP servers into smaller, project-specific ones
   - **Transport upgrade**: Recommend SSE/HTTP over stdio for servers that support it (avoids per-turn process spawning)

4. **Apply selected fixes** by editing the config files using the Edit tool.

5. **Verify** by re-running the audit and showing the before/after score.

6. **Show rollback command**:
   ```
   To undo: cp ~/.claude/settings.json.bak-mcp-fix-TIMESTAMP ~/.claude/settings.json
   ```
