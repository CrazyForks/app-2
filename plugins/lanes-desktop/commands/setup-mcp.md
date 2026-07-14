---
description: Connect Claude Code and Cursor to the running Lanes app via SSE MCP at localhost:5353
allowed-tools: Bash
---

Set up the Lanes MCP server in every installed agent client. Run the section for each client detected; skip sections for clients that aren't installed. Both clients can coexist, and this command is idempotent.

## 1. Detect installed clients

- Claude Code: `command -v claude` (non-empty path means installed).
- Cursor: `test -d ~/.cursor || test -d /Applications/Cursor.app` (either is sufficient).

If neither is installed, stop and tell the user.

## 2. Claude Code

Skip if Claude Code wasn't detected.

1. **Check current state.** Run `claude mcp list` and look for either an entry named `lanes-local` (the current name) or a legacy `lanes` entry whose URL is `http://localhost:5353/sse`. Either one means the local server is already registered, so skip to step 3 and don't add a duplicate. (A `lanes` entry pointing somewhere else is the separate remote Lanes Forms server; leave it alone.)
2. **If neither is present**, register it:
   ```
   claude mcp add --transport sse lanes-local http://localhost:5353/sse --scope user
   ```
   `--scope user` writes to `~/.claude.json` so it applies across all your projects.
3. **Verify registration** with `claude mcp list`. Confirm `lanes-local` (or your existing `lanes`) shows up with the SSE URL.

## 3. Cursor

Skip if Cursor wasn't detected. Cursor has no `cursor mcp add` CLI, so we edit `~/.cursor/mcp.json` directly (using `jq` to preserve any other `mcpServers` entries the user has).

1. **Check current state.** Run:
   ```
   jq -r '(.mcpServers["lanes-local"] // .mcpServers.lanes).url // empty | select(contains("5353"))' ~/.cursor/mcp.json 2>/dev/null
   ```
   Non-empty output means the local server is already registered, as `lanes-local` or as a legacy `lanes` still pointing at `localhost:5353` (don't overwrite without asking). A bare `lanes` entry pointing elsewhere is the separate remote Lanes Forms server; leave it alone and still add `lanes-local` below.
2. **If neither is present**, register it. Preferred (uses `jq`):
   ```
   mkdir -p ~/.cursor
   [ -f ~/.cursor/mcp.json ] || echo '{}' > ~/.cursor/mcp.json
   tmp=$(mktemp)
   jq '.mcpServers["lanes-local"] = {"url": "http://localhost:5353/sse"}' \
      ~/.cursor/mcp.json > "$tmp" && mv "$tmp" ~/.cursor/mcp.json
   ```
   Fallback if `jq` isn't installed:
   ```
   python3 - <<'PY'
   import json, pathlib
   p = pathlib.Path.home() / '.cursor' / 'mcp.json'
   p.parent.mkdir(exist_ok=True)
   data = json.loads(p.read_text()) if p.exists() else {}
   data.setdefault('mcpServers', {})['lanes-local'] = {'url': 'http://localhost:5353/sse'}
   p.write_text(json.dumps(data, indent=2) + '\n')
   PY
   ```
3. **Verify** with `jq '.mcpServers["lanes-local"]' ~/.cursor/mcp.json` (or `python3 -c "import json,pathlib;print(json.loads(pathlib.Path.home().joinpath('.cursor/mcp.json').read_text())['mcpServers']['lanes-local'])"`). Confirm the URL entry is present.

## 4. Probe the endpoint

Run once, regardless of which clients were set up:
```
curl -sS -m 2 -o /dev/null -w "%{http_code}\n" http://localhost:5353/sse
```
- `200` → Lanes is up; the MCP is reachable.
- Connection refused / timeout → Lanes isn't running. Tell the user to launch the Lanes desktop app (`open -a Lanes` on macOS), then re-run this command.

## 5. Report

One short paragraph covering: per-client registration status (Claude Code, Cursor), whether the endpoint is live, and the restart hint. Claude Code needs a fresh terminal so the new MCP loads; Cursor needs the IDE (or `cursor-agent`) restarted.

Notes:
- New installs register the local server as `lanes-local`. Existing `lanes` installs keep working, so you don't need to change anything. Re-add as `lanes-local` only if you also want the separate remote `lanes` server (Lanes Forms), which now claims the bare `lanes` name.
- Don't overwrite an existing `lanes-local` (or legacy `lanes`) MCP entry without asking. The user may have customised the URL.
- The SSE endpoint is local-only (no auth, never leaves the machine). Don't expose `localhost:5353` over the network.
- If the user is on a non-default port (set in Lanes Settings → Local MCP), they should pass that URL instead, in both clients.
