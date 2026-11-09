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

### Parallel migration sessions (router built with auth modes)

Keep the existing MCP entry until both credentials have been validated against
an isolated edge. Two distinct entries can run side by side:

```jsonc
"mcp": {
  "unimind_oidc": {
    "type": "local",
    "command": ["/path/to/unimind-router", "--client-id", "opencode-oidc"],
    "environment": {
      "UNIMIND_AUTH_MODE": "oidc-device",
      "UNIMIND_OIDC_ISSUER": "https://<keycloak-host>/realms/<realm>",
      "UNIMIND_LOG_FILE": "/path/to/oidc-router.log"
    },
    "enabled": true, "timeout": 300000
  },
  "unimind_recovery": {
    "type": "local",
    "command": ["/path/to/unimind-router", "--client-id", "opencode-recovery"],
    "environment": {
      "UNIMIND_AUTH_MODE": "developer-key",
      "UNIMIND_DEVELOPER_KEY_FILE": "/path/to/protected/operator-key",
      "UNIMIND_LOG_FILE": "/path/to/recovery-router.log"
    },
    "enabled": true, "timeout": 300000
  }
}
```

The recovery router needs `UNIMIND_CA_CERT` or a previously pinned WSS edge.
Both entries use the same server/tunnel settings in `router.json`; neither
entry contains a credential. Each router reports its own jLens URL through
`router_info`. OIDC device approval appears in the OIDC router's log; do not
copy the private device code, access token or operator key into OpenCode config.

This example requires router 0.4.0 or later. In OpenCode, ask for
`unimind_sign_in_status` to get the sign-in link; the Unimind tools load by
themselves once you are signed in and authorized.

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
- **jLens** uses a separate port per router (18080, 18081, …); **tablets** also
  get a separate random port per session.

## Linux

Same configs, with the Linux binary path, e.g. `/home/<you>/.unimind/bin/unimind-router`.
