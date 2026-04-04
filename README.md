# mcp-audit — MCP Configuration Auditor for Claude Code

Detect and fix performance issues in your MCP server configuration. Built by [Gentic AI](https://gentic.pro).

## Why this exists

We discovered that Claude Code's MCP runtime doesn't sort tools deterministically before sending them to the API. The MCP spec doesn't guarantee `listTools()` order. When tool ordering changes between turns, it busts the prompt cache — costing you **2-3x more in API fees** without any visible error.

This plugin audits your entire MCP setup and catches this and other performance killers.

## What it checks

| Issue | Severity | Impact |
|-------|----------|--------|
| **Tool ordering instability** | Critical | Prompt cache bust every turn — 2-3x API cost |
| **Duplicate tool names** | Critical | Silent tool collision — second server's tool is dropped |
| **High tool count (>50/server)** | Warning | Context bloat — ~10K extra tokens per server |
| **Total tools >100** | Warning | ~20K+ tokens in tools block — cache and cost impact |
| **Stale/disconnected servers** | Warning | Failed tool calls, wasted config |
| **stdio transport overhead** | Info | Process spawned per turn in unpatched versions |

## Installation

```bash
# From the Claude Code plugin marketplace
claude plugins install mcp-audit

# Or clone manually
git clone https://github.com/genticai/mcp-audit-plugin ~/.claude/plugins/installed/mcp-audit
```

## Usage

### Quick audit
```
/mcp-audit
```
Produces a health report with score (0-100), issues found, cost impact estimate, and prioritized fixes.

### Apply fixes
```
/mcp-fix
```
Backs up your config, then offers to apply auto-fixable changes (disable stale servers, remove duplicates, reduce bloat).

### Deep diagnostics
The `mcp-health-checker` agent runs thorough connection testing, latency measurement, and schema stability analysis. It's automatically available when you need deeper investigation.

### Automatic monitoring
A `SessionStart` hook runs a quick health check every session. It only speaks up if there's an actionable issue — no noise on clean sessions.

## Example output

```
## MCP Health Report

Score: 62/100 — DEGRADED

### Servers Detected
| Server | Tools | Transport | Status |
|--------|-------|-----------|--------|
| filesystem | 12 | stdio | Healthy |
| memory | 9 | stdio | Healthy |
| n8n-mcp | 34 | stdio | Healthy |
| plugin_playwright | 18 | stdio | Healthy |
| claude_ai_Supabase | 22 | sse | Healthy |
| plugin_stripe | 16 | sse | Healthy |

### Issues Found

🔴 CRITICAL: Tool ordering not deterministic
   111 tools across 6 servers — ordering may change between turns.
   Fix: Update Claude Code or apply the sort patch.

🟡 WARNING: Total tool count is 111 (>100)
   Estimated tools block: ~22,200 tokens.
   Fix: Use project-level .mcp.json to load only relevant servers.

🟡 WARNING: n8n-mcp has 34 tools
   Consider splitting into domain-specific servers.

### Prompt Cache Impact
- Estimated waste per 20-turn session: ~421,800 tokens ($1.27)
- Monthly estimate (100 sessions/day): $3,800
- Fix tool ordering → saves $2,850/month
- Reduce to 50 tools → saves additional $570/month

### Recommendations
1. Apply tool sorting fix (or update Claude Code when patched)
2. Create project-level .mcp.json files to scope servers
3. Consider splitting n8n-mcp into smaller servers
```

## The technical details

### The prompt cache bug

`createBundleMcpToolRuntime` in Claude Code collects tools via `client.listTools()` without sorting. The tools array becomes part of the API request's cache key. When order changes between turns, the cache misses and you pay full price for the entire tools block.

**The fix** (one line):
```typescript
tools: tools.toSorted((a, b) => a.name.localeCompare(b.name))
```

### The session-scope optimization

Beyond sorting, the runtime is recreated every turn — spawning stdio transports, calling listTools(), then disposing. Caching the runtime at session scope (keyed on config hash) would eliminate this overhead entirely.

## Built by

[Gentic AI](https://gentic.pro) — AI infrastructure optimization for businesses that run on automation.

- [n8n Template Marketplace](https://gentic-n8n.deals) — Production-ready automation workflows
- [VoiceScheduleAI](https://voicescheduleai.com) — AI voice agents for appointment scheduling
- [DealiQ](https://dealiq.click) — AI-powered real estate intelligence

## License

MIT
