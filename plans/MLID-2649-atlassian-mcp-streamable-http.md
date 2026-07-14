# MLID-2649 — Migrate Atlassian MCP server from deprecated SSE to Streamable HTTP

## Task Reference

- **Jira**: [MLID-2649](https://localinfusion.atlassian.net/browse/MLID-2649)
- **Story Points**: 1
- **Branch**: `feature/MLID-2649-atlassian-mcp-streamable-http`
- **Base Branch**: `develop`
- **Status**: Done

---

## Summary

Atlassian is retiring the HTTP+SSE transport endpoint (`https://mcp.atlassian.com/v1/sse`) after **30 June 2026**. The committed, team-shared `.mcp.json` at the repo root still points the `atlassian` MCP server at that SSE endpoint. After the cutoff, the Atlassian MCP integration will fail to connect, breaking all Jira/Confluence MCP tooling in Claude Code for the whole team.

The fix is a one-line transport swap in `.mcp.json`: change the `atlassian` entry from the SSE transport (`type: "sse"`, `/v1/sse`) to the Streamable HTTP transport (`type: "http"`, `/v1/mcp`). Because the file is committed, a single change migrates everyone on the team.

This is a **configuration change with zero conditional logic** — it falls under the documented TDD Configuration exception. There is no production code and no unit/integration test to write. Verification is manual, via the Acceptance Criteria below.

---

## Codebase Analysis

Current state of `.mcp.json` at the repo root (verified during planning):

```json
"atlassian": {
  "type": "sse",
  "url": "https://mcp.atlassian.com/v1/sse"
}
```

Key facts:

- `.mcp.json` lives at the repository root and is committed (team-shared).
- Other server entries — `li-mcp` (`sse`, localhost), `mongodb` (`npx`/stdio), `playwright` (`stdio`) — are **out of scope**. The `li-mcp` SSE entry points at a local server (`http://localhost:3002/phi/sse`) and is unaffected by the Atlassian cloud deprecation; do not touch it.
- `type: "http"` is Claude Code's identifier for the Streamable HTTP transport, which is what Atlassian's notice (`/v1/mcp`) requires.
- Existing Atlassian OAuth authorization carries over; no re-authentication expected.
- MCP servers connect at Claude Code startup, so the change only takes effect after a Claude Code restart.

---

## Implementation Steps

### Step 1 — Swap the `atlassian` transport in `.mcp.json`

- **Files**: `.mcp.json` (repo root)
- **What**: In the `mcpServers.atlassian` entry, change `type` from `"sse"` to `"http"` and `url` from `https://mcp.atlassian.com/v1/sse` to `https://mcp.atlassian.com/v1/mcp`. Leave every other server entry untouched.

After the edit, the entry reads:

```json
"atlassian": {
  "type": "http",
  "url": "https://mcp.atlassian.com/v1/mcp"
}
```

### Step 2 — Verify JSON is well-formed

- **Files**: `.mcp.json`
- **What**: Confirm the file still parses as valid JSON after the edit (e.g. `node -e "JSON.parse(require('fs').readFileSync('.mcp.json','utf8'))"` or `npx prettier --check .mcp.json`). The only changes are the two string values inside the `atlassian` entry; structure is unchanged.

### Step 3 — Manual verification (requires Claude Code restart — user-driven)

- **What**: After restarting Claude Code so MCP servers reconnect:
  1. `claude mcp list` shows `atlassian: https://mcp.atlassian.com/v1/mcp (HTTP) - Connected`.
  2. A Jira read (e.g. fetching an MLID issue) succeeds through the new transport.
  3. The SSE deprecation notice no longer appears in Atlassian MCP tool responses.

This step happens after the change lands and Claude Code is restarted; it cannot be verified within the same running session that makes the edit.

---

## Files Affected

| File | Action | Description |
|------|--------|-------------|
| `.mcp.json` | Modify | Swap `atlassian` server `type` `sse` → `http` and `url` `/v1/sse` → `/v1/mcp` |

---

## Testing Strategy

- **Unit tests**: None — pure configuration change, no logic (TDD Configuration exception).
- **Integration tests**: None.
- **Manual verification**: The Acceptance Criteria in Step 3 — `claude mcp list` shows the HTTP transport Connected, a Jira read succeeds, and the SSE deprecation notice is gone. Performed by the user after a Claude Code restart.

---

## Security Considerations

- **Input validation**: N/A — static config.
- **Authorization**: Existing Atlassian OAuth token carries over; no re-authentication expected. No secrets are added or changed.
- **PHI handling**: N/A — no patient data involved.

---

## Acceptance Criteria (from ticket)

1. `.mcp.json` `atlassian` entry uses `type: "http"` and `url: https://mcp.atlassian.com/v1/mcp`.
2. After restarting Claude Code, `claude mcp list` shows `atlassian: https://mcp.atlassian.com/v1/mcp (HTTP) - Connected`.
3. A Jira read operation (e.g. fetching an MLID issue) succeeds through the new transport.
4. The SSE deprecation notice no longer appears in MCP tool responses.

---

## Open Questions

- None. The fix is fully specified in the ticket and confirmed against the current file.
