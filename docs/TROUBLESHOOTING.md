# Troubleshooting

## Step 1: look at the metrics

While your AI tool is running, open **http://127.0.0.1:18080/__router/metrics**. It's JSON, so
any browser works. The fields that matter:

| Field | Healthy value | If not |
|---|---|---|
| `connected` | `true` | See the `tunnel` and `liveness` fields below |
| `tunnel.state` | `up` | `starting`, `retry_wait`, `waiting_network`, `throttled`, `error`. Explained below. |
| `tunnel.last_error` | `null` | Usually tells you exactly what's wrong |
| `liveness.l1/l2/l3.ok` | `true` | L1 = WebSocket ping, L2 = WSLink heartbeat, L3 = MCP ping. After 3 failures the router reconnects by itself. |
| `tablet_url` | a URL | `null` means the tablet is off |

If the page doesn't load at all, no router is running. Your tool hasn't started it (check the
MCP config, see [HOSTS.md](HOSTS.md)), or jLens is turned off (`ports.jlens: 0`).

## Step 2: read the log

Set `UNIMIND_LOG_FILE` in your tool's MCP `env` (see [HOSTS.md](HOSTS.md)). A healthy start looks like:

```
Built-in Azure tunnel configured — bringing it up
tunnel: starting Azure Bastion IP tunnel <bastion> -> 192.0.2.10:3389 on 127.0.0.1:8443
tunnel: UP (bastion-ip 192.0.2.10 forwards ["8443:3389"])
CONNECT step 1/5 … step 3/5
Auth ready (1.0s) — token source: entra-az
CONNECT step 5/5: WebSocket ESTABLISHED to wss://127.0.0.1:8443/ws
Ready
```

Add `"UNIMIND_LOG_LEVEL": "debug"` for more detail. The tunnel's own output (from `az`) is in
`~/.unimind/tunnel-bastion.log`.

---

## Common problems

### `tunnel.state: waiting_network`
The router can't resolve or reach `network_probe` (`management.azure.com:443` by default). It
isn't trying the tunnel on purpose, and it retries immediately once the network is back.
- Check your connection or VPN: `nslookup management.azure.com`
- Behind a proxy that blocks raw TCP? Set `"network_probe": ""` to turn the check off.

### `tunnel.state: retry_wait`
The network is fine, but the last tunnel attempt failed. `next_attempt_in_s` shows when it
tries again (default every 300 s). Check `last_error` and `~/.unimind/tunnel-bastion.log`.
Common causes:
- **`az` not logged in or token expired:** run `az login`. The next attempt picks it up.
- **Wrong subscription or permissions:** `az network bastion show -n <bastion> -g <rg> --subscription <sub>` must work for you.
- **`Custom ports are not allowed … 22, 3389`:** `forwards` must use remote port 3389 (or 22) in `bastion-ip` mode.
- **IP-based connect disabled on the Bastion:** ask your admin (`enableIpConnect`).

Don't want to wait? Restart your AI tool. The first attempt after start is always immediate.

### `local port 8443 is held by another (unhealthy) process`
Something else is listening on the forward's local port but doesn't respond: often a stale
manual `az network bastion tunnel` or `ssh -L`. The router never kills processes it didn't start.
Find and close it:
```powershell
netstat -ano | findstr :8443
Stop-Process -Id <pid>
```

### `az CLI not found`
Install the Azure CLI, or set `UNIMIND_TUNNEL_AZ_BIN` to the full path of `az.cmd` (Windows) / `az`.

### Connected, but `Auth` / 401 / 403 errors
- Check the `Auth ready … token source: …` log line (or `auth_source` in metrics) to see which token was used.
- Using Keycloak/OAuth2: make sure the token file is fresh. An expired JWT is skipped and the router falls back to Entra.
- `az account show` must show the account that has Unimind access.
  `az login --tenant <tenant>` if you have several tenants.
- If your deployment uses MSAL instead, check `UNIMIND_ENTRA_TENANT` / `UNIMIND_ENTRA_CLIENT_ID`.
  A device-code prompt appears in the log the first time.

### `TLS CERTIFICATE PIN MISMATCH ... connection REFUSED`
The server presented a different certificate from the one the router pinned on first use (see
[TLS certificate pinning](CONFIGURE.md#tls-certificate-pinning-tofu)). The log shows the pinned
and the presented SHA-256 fingerprints.
- If the Unimind server's certificate was renewed, check the new fingerprint with the
  administrator (`openssl x509 -in <cert.pem> -noout -fingerprint -sha256` on the server). Then run
  `unimind-router --repin`, confirm, and restart your IDE.
- If nobody changed the certificate, **don't re-pin**. Something may be intercepting the
  connection (a TLS-inspection proxy or MITM), or the tunnel or port may now point at a different
  server.

### `TLS certificate verification DISABLED for 127.0.0.1`
This is expected with the built-in tunnel: the connection to `127.0.0.1` is local, and the
Bastion leg is encrypted by Azure. The certificate is still pinned on first use (TOFU), so a
changed certificate is detected. To also verify the certificate chain, set `UNIMIND_CA_CERT` and
`UNIMIND_TLS_SERVER_NAME`.

### jLens shows "forbidden host"
Open it as `http://127.0.0.1:18080/jlens` or `http://localhost:18080/jlens`. Other hostnames are
rejected on purpose (DNS-rebinding protection).

### Tools are missing or the tool says the MCP server failed to start
- Run the binary by hand: `unimind-router.exe --version`. If that fails, the binary is wrong for
  your OS or blocked (on Windows, check *Properties → Unblock* for downloaded files).
- Give the host a longer start timeout (OpenCode: `"timeout": 300000`), since the first start waits for the tunnel.

### Several sessions, only one tunnel
That's normal: see *Several tools or windows at once* in [HOSTS.md](HOSTS.md). If you close the
window that owns the tunnel, other sessions take over within about 15 seconds.

---

## Reporting a problem

Include:
- `unimind-router --version`
- the output of `/__router/metrics`
- the last ~100 lines of your router log and of `~/.unimind/tunnel-bastion.log`

Tokens are never logged, but check the logs for anything you consider sensitive before sharing.
