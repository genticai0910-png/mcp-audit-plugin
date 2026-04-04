---
name: mcp-optimization
description: MCP server optimization best practices — tool ordering, caching, transport selection, context management. Use when discussing MCP performance, tool configuration, or server setup.
---

# MCP Optimization Guide

## The Prompt Cache Problem

Claude's API caches the system prompt + tools block between turns. When the cache hits, those tokens are billed at 75% less. MCP tools are part of this cache key.

**If tool ordering changes between turns, the cache busts and you pay full price every turn.**

### Why ordering changes
- `client.listTools()` returns tools in server-internal iteration order
- The MCP spec does not guarantee deterministic ordering
- Third-party servers, paginated responses, or config reloads can shuffle order
- `createBundleMcpToolRuntime` collects without sorting

### The fix
Sort tools alphabetically by name before including in the API request:
```typescript
tools: tools.toSorted((a, b) => a.name.localeCompare(b.name))
```

### Cost impact
- 50 tools ≈ 10,000 tokens in the tools block
- 20-turn session: 190K uncached tokens vs 10K uncached + 190K cached
- At $3/M input: $0.57 wasted per session
- 100 sessions/day: $1,710/month in unnecessary costs

## Best Practices

### 1. Minimize total tool count
Each tool adds ~200 tokens to the context. Use project-level `.mcp.json` to enable only relevant servers per project instead of loading everything globally.

### 2. Choose the right transport
| Transport | Pros | Cons | Use When |
|-----------|------|------|----------|
| **stdio** | Simple, no network | Process spawned per connection, no sharing | Development, simple tools |
| **SSE** | Persistent connection, shared | Needs HTTP server | Production, shared servers |
| **HTTP** | Stateless, scalable | No streaming | High-volume, load-balanced |

### 3. Avoid tool name collisions
When multiple servers are loaded, tool names are prefixed with `mcp__{server}__{tool}`. But if two servers register the same tool name, the first server wins and the duplicate is silently dropped. Audit regularly with `/mcp-audit`.

### 4. Session-scope vs turn-scope runtime
Current Claude Code recreates the MCP runtime every turn (spawning transports, listing tools, then disposing). This adds latency and prevents connection reuse.

**Ideal**: Cache the runtime at session scope, keyed on a hash of the MCP server config. Dispose on session end. This avoids re-spawning MCP servers per turn.

### 5. Monitor tool schema stability
If a server updates its tool schemas between turns (e.g., dynamic tool generation), the tools block changes even with sorting. Pin server versions in production.

### 6. Use deferred tool loading
Claude Code supports deferred tools — tools that appear by name only until explicitly fetched via `ToolSearch`. This keeps the initial tools block small. Only fetch tool schemas when you actually need them.

## Audit Checklist
- [ ] Total tools < 100
- [ ] No server has > 50 tools
- [ ] No duplicate tool names across servers
- [ ] Tool ordering is deterministic (sorted)
- [ ] All configured servers are responsive
- [ ] Using SSE/HTTP transport where possible
- [ ] Project-level `.mcp.json` scopes servers per project
