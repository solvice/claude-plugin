# Connectors

## Solvice API

**MCP endpoint:** `https://api.solvice.io/mcp`
**Auth:** API key (JWT) via `SOLVICE_API_KEY` environment variable
**Required:** Yes — all skills require this connector.

### Setup

Set the `SOLVICE_API_KEY` environment variable to your Solvice API key:

```bash
export SOLVICE_API_KEY=your-key-here
```

Get your API key at [https://app.solvice.io](https://app.solvice.io) → Settings → API Keys.

Run `/solvice:setup` for a guided connection walkthrough.

### Claude Desktop — Known OAuth Issues

Claude Desktop's OAuth flow for custom MCP servers has several active bugs (as of Feb 2026). This is why the plugin uses a static API key (`SOLVICE_API_KEY` env var) instead of OAuth.

| Issue | Impact | Link |
|-------|--------|------|
| Refresh tokens stored but never used | Tokens expire, manual re-auth required multiple times/day | [#21333](https://github.com/anthropics/claude-code/issues/21333) |
| Silent failure when OAuth incomplete | No tools appear, no error shown | [#26917](https://github.com/anthropics/claude-code/issues/26917) |
| Reconnection fails after successful OAuth | Tools unavailable until full restart | [#10250](https://github.com/anthropics/claude-code/issues/10250) |
| OAuth broken after Dec 2025 Desktop update | Desktop opens internal OAuth URL instead of MCP server's endpoints | [claude-ai-mcp#5](https://github.com/anthropics/claude-ai-mcp/issues/5) |
| Scope parameter not sent during OAuth2 | Auth flow fails entirely for servers requiring scopes | [#5826](https://github.com/anthropics/claude-code/issues/5826) |
| 302 redirects not followed with bearer token | Connection drops after redirect | [#23468](https://github.com/anthropics/claude-code/issues/23468) |

**Workaround:** Use static API key auth via `SOLVICE_API_KEY` environment variable (the current default). Do not switch to OAuth until these upstream bugs are resolved.

### General Tools

| Tool | Description |
|------|-------------|
| `list-jobs` | List recent solve jobs across all solvers — paginated with sort options |

### VRP Tools (Route Optimization)

| Tool | Description |
|------|-------------|
| `vrp-get-all` | Get complete solve run (request + solution + explanation) by ID |
| `vrp-get-request` | Get the original request for a solve run |
| `vrp-get-solution` | Get the solution with assigned routes — displays interactive route map |
| `vrp-get-explanation` | Get score breakdown and constraint violations — displays score dashboard |
| `vrp-get-debug` | Get raw debug data for low-level constraint tracing |
| `vrp-get-status` | Check solve run status (solved, pending, failed) |
| `vrp-demo` | Generate a sample OnRouteRequest — use to verify connectivity |
| `vrp-solve` | Submit a VRP problem for async solving — returns job ID |
| `vrp-solve-sync` | Solve synchronously — waits and returns solution directly (~30s) |
| `vrp-evaluate` | Evaluate an existing solution against constraints (async) |
| `vrp-evaluate-sync` | Evaluate an existing solution against constraints (sync) |
| `vrp-suggest` | Get scheduling suggestions for unassigned jobs (async) |
| `vrp-suggest-sync` | Get scheduling suggestions for unassigned jobs (sync) |
| `vrp-change` | Apply incremental changes to an existing solution (what-if analysis) |
| `vrp-what-if` | Open interactive drag-and-drop scenario builder with instant scoring |

### Fill Tools (Shift Filling)

| Tool | Description |
|------|-------------|
| `fill-get-all` | Get complete solve run (request + solution + explanation) by ID |
| `fill-get-request` | Get the original request for a solve run |
| `fill-get-solution` | Get the solution with shift assignments — displays Gantt chart |
| `fill-get-explanation` | Get score breakdown and constraint violations |
| `fill-get-status` | Check solve run status |
| `fill-demo` | Generate a sample Fill request |
| `fill-solve` | Submit a Fill problem for solving |
| `fill-evaluate` | Evaluate an existing Fill solution |

### Create Tools (Shift Creation)

| Tool | Description |
|------|-------------|
| `create-get-all` | Get complete solve run (request + solution + explanation) by ID |
| `create-get-request` | Get the original request for a solve run |
| `create-get-solution` | Get the solution with generated shifts — displays Gantt chart |
| `create-get-explanation` | Get score breakdown |
| `create-get-status` | Check solve run status |
| `create-demo` | Generate a sample Create request |
| `create-solve` | Submit a Create problem for solving |

### Interactive Apps

These render inline in Cowork when the corresponding tool is called:

| App | Triggered by | Description |
|-----|-------------|-------------|
| Route Map | `vrp-get-solution` | Interactive MapLibre GL map showing each vehicle's path and stops |
| Score Dashboard | `vrp-get-explanation` | Constraint violation breakdown with score details |
| What-If Builder | `vrp-what-if` | Drag-and-drop job reassignment with instant scoring |
| Gantt Chart | `fill-get-solution`, `create-get-solution` | Schedule timeline for shift assignments |
