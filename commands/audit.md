---
name: mcp-audit
description: Audit all configured MCP servers for performance issues — tool ordering, duplicates, context bloat, stale connections. Use when the user says "audit mcp", "check mcp", "mcp health", or "mcp performance".
---

# MCP Configuration Audit

Run a comprehensive audit of all configured MCP servers and report performance issues.

## Steps

1. **Read MCP configuration** from all sources:
   - `~/.claude/settings.json` (global)
   - `~/.claude/settings.local.json` (local overrides)
   - `.mcp.json` in the current project directory
   - Any plugin-provided MCP servers

   Use Bash to read these files:
   ```bash
   cat ~/.claude/settings.json 2>/dev/null
   cat ~/.claude/settings.local.json 2>/dev/null
   cat .mcp.json 2>/dev/null
   ```

2. **Count tools per server** by checking the deferred tools list. Search for `mcp__` prefixed tools to identify which servers provide how many tools. Group by server prefix.

3. **Check for issues** and score each:

   ### Critical Issues (score -20 each)
   - **Tool ordering instability**: The MCP spec does not guarantee `listTools()` order. If `createBundleMcpToolRuntime` doesn't sort tools, the tools array changes between turns, busting the prompt cache. Check if the user's Claude Code version includes the sort fix.
   - **Duplicate tool names**: Two servers registering tools with the same name after prefix stripping causes collision. First-server-wins silently drops the duplicate.

   ### Warning Issues (score -10 each)
   - **High tool count per server**: Any server with > 50 tools adds significant context. Flag servers with excessive tool counts.
   - **Total tool count > 100**: Combined tools across all servers exceeding 100 causes major context bloat (~20K+ tokens in the tools block).
   - **Stale/failing servers**: Servers listed in config but not appearing in deferred tools may be disconnected or failing to start.

   ### Info Issues (score -5 each)
   - **Unused servers**: Servers that are configured but disabled.
   - **No health endpoint**: Servers without a standard health check capability.
   - **Stdio transport overhead**: Each stdio server spawns a process per turn in unpatched Claude Code.

4. **Calculate health score**: Start at 100, subtract for each issue found.

5. **Estimate cache impact**:
   - Count total MCP tools
   - Estimate tokens: ~200 tokens per tool (name + description + schema)
   - If tool ordering is unstable: `wasted_tokens_per_session = total_tool_tokens * (turns - 1)`
   - Calculate cost: wasted_tokens * $0.003 per 1K tokens (uncached) vs $0.00075 (cached)

6. **Output the report** in this format:

```
## MCP Health Report

**Score: XX/100** [HEALTHY | DEGRADED | CRITICAL]

### Servers Detected
| Server | Tools | Transport | Status |
|--------|-------|-----------|--------|
| ... | ... | ... | ... |

### Issues Found

🔴 CRITICAL: [issue description]
   Fix: [specific recommendation]

🟡 WARNING: [issue description]
   Fix: [specific recommendation]

🔵 INFO: [issue description]

### Prompt Cache Impact
- Total MCP tools: XX
- Estimated tools block size: ~XX,XXX tokens
- Tool ordering: [stable ✅ | unstable ❌]
- Estimated waste per 20-turn session: ~XXX,XXX tokens ($X.XX)
- Monthly estimate (100 sessions/day): $X,XXX

### Recommendations (priority order)
1. [most impactful fix]
2. [next fix]
3. ...
```
