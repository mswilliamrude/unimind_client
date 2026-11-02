# Connecting your AI tool

Every tool starts `unimind-router` as a **local stdio MCP server**. Put the connection settings in
`~/.unimind/router.json` ([CONFIGURE.md](CONFIGURE.md)), so each tool only needs the path to the
binary.

Use forward slashes or escaped backslashes in JSON paths, e.g.
`C:/Users/you/.unimind/bin/unimind-router.exe`.

**`--client-id`:** give each tool its own prefix (`opencode`, `copilot-cli`, `vscode`,
`claude-code`). It names your sessions on the server and decides where the router looks first
for your GitHub Copilot token. A unique suffix is added automatically, so several windows of the
same tool don't collide.

**Logs:** MCP hosts often hide stderr. Set `UNIMIND_LOG_FILE` so you can read what happened
(see [TROUBLESHOOTING.md](TROUBLESHOOTING.md)).

---

## OpenCode

`~/.config/opencode/opencode.json` (or `opencode.jsonc`):

```json
{
  "mcp": {
    "unimind": {
      "type": "local",
      "command": ["C:/Users/<you>/.unimind/bin/unimind-router.exe", "--client-id", "opencode"],
      "environment": { "UNIMIND_LOG_FILE": "C:/Users/<you>/.unimind/router-opencode.log" },
      "enabled": true,
      "timeout": 300000
    }
  }
}
```

`timeout` gives the first start time to bring up the Bastion tunnel (usually 5–20 s, longer on a cold
`az`). Restart OpenCode. Unimind's tools show up prefixed with the MCP server name (`unimind_…`).

## GitHub Copilot CLI

`~/.copilot/mcp-config.json`:

```json
{
  "mcpServers": {
    "unimind": {
      "type": "local",
      "command": "C:/Users/<you>/.unimind/bin/unimind-router.exe",
      "args": ["--client-id", "copilot-cli"],
      "env": { "UNIMIND_LOG_FILE": "C:/Users/<you>/.unimind/router-copilot.log" },
      "tools": ["*"]
    }
  }
}
```

## VS Code (Copilot Chat, agent mode)

Workspace: `.vscode/mcp.json`. For all workspaces, use **MCP: Open User Configuration**.

```json
{
  "servers": {
    "unimind": {
      "type": "stdio",
      "command": "C:/Users/<you>/.unimind/bin/unimind-router.exe",
      "args": ["--client-id", "vscode"],
      "env": { "UNIMIND_LOG_FILE": "C:/Users/<you>/.unimind/router-vscode.log" }
    }
  }
}
```

Then start the server from the MCP view or with **MCP: List Servers → unimind → Start**.

## Claude Code

```bash
claude mcp add unimind --scope user -- C:/Users/<you>/.unimind/bin/unimind-router.exe --client-id claude-code
```

or in a project's `.mcp.json`:

```json
{
  "mcpServers": {
    "unimind": {
      "command": "C:/Users/<you>/.unimind/bin/unimind-router.exe",
      "args": ["--client-id", "claude-code"]
    }
  }
}
```

---

## Several tools or windows at once

That's fine:
- Each process gets its own session on the server.
- **One tunnel serves all of them.** The first router to start brings the Bastion tunnel up. The
  others see the healthy port and use it (`adopt_existing`). If the owning session closes, the
  next health check (≤15 s) in another session restarts the tunnel.
- **jLens** (18080) is served by one router and shared. **Tablets** get a separate random port
  per session.

## Linux

Same configs, with the Linux binary path, e.g. `/home/<you>/.unimind/bin/unimind-router`.
