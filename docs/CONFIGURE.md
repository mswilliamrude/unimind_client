# Configuring unimind-router

The router reads its settings from three places. Higher in this list wins:

1. **Command-line flags**, e.g. `--server`, `--client-id` (`unimind-router --help` lists them all)
2. **Environment variables** (`UNIMIND_*`), e.g. set in your AI tool's MCP `env` block
3. **`router.json`**: `%USERPROFILE%\.unimind\router.json` (Windows) or `~/.unimind/router.json`
   (Linux). Override the path with `UNIMIND_ROUTER_CONFIG`.

Most users only need `router.json`. Keep your AI tool's MCP entry minimal (just the path to the exe),
so one file controls every tool on your machine.

> `router.json` is strict JSON: no comments, no trailing commas.

---

## Minimal config

```json
{
  "server": "wss://127.0.0.1:8443/ws",
  "tunnel": {
    "enabled": true,
    "mode": "bastion-ip",
    "subscription": "<subscription>",
    "bastion_name": "<bastion>",
    "bastion_resource_group": "<bastion-rg>",
    "target_ip": "<edge-private-ip>",
    "forwards": ["8443:3389"]
  }
}
```

The router starts the Bastion tunnel, connects to `server` through it, and keeps both alive.
Ready-to-edit files are in [`../examples/`](../examples/).

---

## Top-level keys

| Key | Default | Meaning |
|---|---|---|
| `server` | `wss://127.0.0.1:8443/ws` | Unimind WebSocket endpoint. With the built-in tunnel this is the **local** end of the forward. |
| `ports.jlens` (or `jlens_port`) | `18080` | Local jLens port: `http://127.0.0.1:<port>/jlens`. `0` turns it off. |
| `ports.tablet` (or `tablet_port`) | `"auto"` | Local tablet port. See [Tablet](#tablet). |
| `tls_pinning` | `true` | TLS certificate pinning (trust on first use). See [TLS certificate pinning](#tls-certificate-pinning-tofu). |
| `tls_pins` | written by the router | Pinned server certificate fingerprints. Managed by the router; see below. |
| `tunnel` | none | Built-in Azure Bastion tunnel. See [Tunnel](#tunnel). Leave it out if you manage your own tunnel or VPN. |
| `UNIMIND_*` | none | Any environment variable from the [table below](#environment-variables) can also be set here as a string, e.g. `"UNIMIND_LOG_LEVEL": "debug"`. Real environment variables still win. |

### jLens

jLens is Unimind's browser UI. The router serves it on `127.0.0.1` only and adds
your Entra token to each request, so the server decides what you're allowed to see.

Every router (one per IDE session) gets **its own** jLens port: the configured one (`18080`) if it's
free, otherwise the next free port (`18081`, `18082`, … up to +19). To find the one that belongs to
your session:

- ask your AI tool for the jLens URL: the router adds a `router_info` tool, and its startup
  instructions name this session's jLens URL;
- or check the router log: `jlens auth-proxy LISTENING on http://127.0.0.1:<port>/jlens`.

Health and diagnostics (on that session's port):

- `http://127.0.0.1:<port>/__router/health`: returns `ok`
- `http://127.0.0.1:<port>/__router/metrics`: connection, tunnel, liveness (L1 WebSocket / L2 WSLink / L3 MCP), ports (`jlens_url`, `tablet_url`), caches

### Tablet

The tablet is a local web page where Unimind can present custom interfaces it builds for you
(for example an ADO work-item view). Each AI-tool session gets its own tablet:

| Value | Behaviour |
|---|---|
| `"auto"` (default), `"0"`, `""` | A random free port, so several sessions never collide |
| a number, e.g. `18181` | Exactly that port. If another session already uses it, the router falls back to a random port and logs it. |
| `"off"` | Disabled |

The actual URL is in `/__router/metrics` (`tablet_url`). It listens on `127.0.0.1` only and
rejects requests whose `Host` header isn't the loopback address. Built-in pages: `/`,
`/__tablet/health`, `/__tablet/status`.

---

## Tunnel

The router runs `az network bastion tunnel` for you, then supervises it. A dropped tunnel is
detected within `check_interval_s` and restarted, and the WebSocket reconnects by itself. Tunnel
processes belong to the router: when the router exits, even if it crashes, they're cleaned up.

**Requirements:** the Azure CLI installed and logged in (`az login`) as an account with *Reader*
on the Bastion and the target. `az` must be on `PATH`, or set `UNIMIND_TUNNEL_AZ_BIN`.

### `bastion-ip` mode (recommended: no SSH)

Bastion connects straight to the Unimind edge's private IP.

| Key | Required | Meaning |
|---|---|---|
| `mode` | yes | `"bastion-ip"` |
| `subscription` | recommended | Subscription name or id of the Bastion |
| `bastion_name`, `bastion_resource_group` | yes | The Azure Bastion resource |
| `target_ip` | yes | Private IPv4 of the Unimind edge |
| `forwards` | yes | `"LOCAL:REMOTE"` pairs, e.g. `"8443:3389"`. One `az` tunnel runs per forward. |

**Why 3389?** Azure Bastion's IP-based connect only allows remote ports **22 and 3389**. The
Unimind edge serves its normal TLS endpoint on 3389 as well (it isn't RDP), so `8443:3389` gives
you `wss://127.0.0.1:8443/ws`. The Bastion must have **IP-based connection** enabled (an admin
setting on the Bastion, Standard SKU or higher).

### `bastion-ssh` mode (legacy)

Bastion connects to a jump VM's SSH port, and `ssh -L` forwards from there. Use this only if
IP-based connect isn't available.

| Key | Meaning |
|---|---|
| `mode` | `"bastion-ssh"` (the default when `mode` is omitted) |
| `vm_name`, `vm_resource_group` (or `vm_resource_id`) | The jump VM |
| `resource_port` | VM SSH port (default `22`) |
| `local_ssh_port` | Local port for the Bastion→SSH leg, e.g. `12222` |
| `ssh_user`, `ssh_key` | SSH login (`ssh_key` may start with `~/`) |
| `ssh_extra_args` | Extra `ssh` arguments (shell metacharacters are rejected) |
| `forwards` | `"LOCAL:HOST:REMOTE"`, e.g. `"8443:192.0.2.10:443"` |

The SSH key must be loaded in an agent or have no passphrase, because the router runs `ssh` with
`BatchMode=yes`. Set `UNIMIND_TUNNEL_SSH_BIN` to choose which `ssh` to use.

### Retry policy (both modes)

| Key | Default | Meaning |
|---|---|---|
| `check_interval_s` | `15` | How often tunnel health is checked |
| `network_probe` | `"management.azure.com:443"` | Checked before **every** tunnel attempt: DNS lookup plus a TCP connect. If it fails (no network, no gateway, no DNS, or Azure unreachable), the router **makes no attempt** and reports `waiting_network`. As soon as the probe succeeds again it retries immediately. `""` turns this check off. |
| `retry_interval_s` | `300` | After a **failed** attempt while the network is up, wait this long before the next one (`retry_wait`). The first attempt after a drop is always immediate. |
| `max_restarts_per_hour` | `20` | Once this is used up, attempts continue every `retry_interval_s`. The router never gives up. |
| `startup_timeout_s` | `60` | How long a new tunnel gets to start listening |
| `adopt_existing` | `true` | If the forward ports are already healthy (e.g. another session's tunnel), use them instead of starting a duplicate |

In practice, after a short blip you're reconnected in about 15 seconds. During an outage the
router waits quietly instead of hammering Bastion, and it reconnects on its own when things come
back. Watch it live in `/__router/metrics` → `tunnel` (`state`, `next_attempt_in_s`,
`network_down`, `last_error`).

---

## TLS certificate pinning (TOFU)

The router pins the Unimind server's TLS certificate on first use. The first successful TLS
connection to the server records the SHA-256 fingerprint of its certificate in router.json, with a
timestamp, keyed by `host:port`:

```json
"tls_pins": {
  "127.0.0.1:8443": {
    "sha256": "3E:45:BA:72:...:82:A9",
    "pinned_at": "2026-11-01T19:05:29Z",
    "method": "tofu"
  }
}
```

From then on, every TLS connection the router makes to that server (WebSocket, jLens, password
login) must present the same certificate. If it doesn't, the router **refuses the connection**
before sending any token or data. The log then shows both fingerprints, explains the likely
causes, and lists what to do next. Pinning is added **on top of** `UNIMIND_CA_CERT`, never instead
of it: with a CA configured the chain must still be valid, and the pin must also match.

A pin also lets the router talk to a **non-loopback** server without a CA. Only the first
connection is unverified (`--insecure`), and from then on the pin verifies the server. Running
`--repin` is the other way to create the first pin.

**When the server's certificate changes** (renewal or rotation), connections fail with
`TLS CERTIFICATE PIN MISMATCH`. That is deliberate: a human has to approve the new certificate.

1. Get the new certificate's SHA-256 fingerprint from the Unimind administrator through a separate
   channel (on the server: `openssl x509 -in <cert.pem> -noout -fingerprint -sha256`).
2. Run the router once by hand, using the same `--server` / router.json / environment as normal:

   ```bash
   unimind-router --repin
   ```

   It connects (bringing up the built-in tunnel if one is configured), shows the pinned and the
   presented fingerprints, and asks `Pin this certificate? [y/N]`. Accept only if the presented
   value matches what the administrator gave you. `--yes` skips the prompt (for scripts). The old
   fingerprint is kept under `previous` for the audit trail.
3. Restart your IDE (or the router).

If the presented fingerprint does **not** match, don't re-pin. Something is intercepting the
connection, or the address points at a different server. Report it.

**Track record (0.4.6+).** When a pin is replaced, the old one is moved into `tls_pin_history`
(newest first, at most 20 per host) with `"active": false`, `retired_at` and its original `method`;
the current pin carries `"active": true`. History is a record only and is never trusted. Routers
from before 0.4.6 ignore the new key and the new field and read the per-host entry exactly as before.

```json
"tls_pin_history": { "host:443": [ {"sha256": "AA:…", "pinned_at": "…", "retired_at": "…", "method": "tofu", "active": false} ] }
```

To turn pinning off, set `"tls_pinning": false` or `UNIMIND_TLS_PIN=0`. This isn't recommended. To
forget a pin, delete its entry under `tls_pins`; the next connection then pins on first use again.

---

## Sign-in (bearer token for Unimind)

### Native Keycloak device sign-in and separate recovery router (0.4.0+)

Set `UNIMIND_AUTH_MODE=oidc-device` on the normal MCP router process and
`UNIMIND_OIDC_ISSUER=https://<keycloak-host>/realms/<realm>`. The public OAuth client id
defaults to `unimind` (`UNIMIND_OIDC_CLIENT_ID`); it is unrelated to `--client-id`.
The router displays a device verification URL and user code on stderr and in its
router log. While sign-in is pending it answers the MCP host immediately with two
local tools, `router_info` and `unimind_sign_in_status` (the latter returns the
verification link and code), and sends `notifications/tools/list_changed` once
connected so the host reloads the full tool list. On Linux, an approved session is shared by routers running as the same
OS user through `~/.unimind/config/auth.json` (owner-only directory/file, serialized
by `auth.lock`). Each router keeps its own WebSocket, rereads the shared session
before use, refreshes before expiry and reconnects with a new access token. Device sign-in requests
`openid offline_access`; if Keycloak grants offline access, the refresh session can survive
the normal SSO idle/max deadlines. `unimind_sign_in_status` reports whether offline access
was granted, the last refresh and any reported refresh expiry. A router also checks the
saved refresh deadline every 15 seconds even without MCP traffic or a WebSocket connection;
while at least one router is running, it renews before a reported deadline, or after about
24 hours for offline tokens without a fixed deadline. A stopped router cannot keep the
session alive indefinitely; Keycloak can still revoke an offline session or enforce its
own offline idle/max limits. If offline access is not granted, normal SSO idle/max limits
apply and the router logs a warning. A restart reuses the saved session while valid. Do not
copy `auth.json` into a shared project or another user's home. This shared-file
mode has not yet been validated on Windows, where cross-process locking is not
implemented. An OIDC error
never selects Entra or a static token in this mode. If authentication is
unavailable, metrics and `router_info` remain available locally.

If the session is revoked or expires, call `unimind_sign_in_status` to get a new device
code; approval is required again. After explicit sign-out, pass `{"sign_in":true}`
to that tool to intentionally sign in again. For explicit sign-out, stop other routers for this OS
user, then run `unimind-router --auth-mode oidc-device --oidc-issuer <realm-issuer> --sign-out`.
This revokes the refresh token at Keycloak and removes the local session; if Keycloak is
unreachable, local sign-out still happens and the command reports the failed revocation.
The saved session is a long-lived credential: protect `~/.unimind/config/auth.json` (0600)
and its containing directory (0700). You can also revoke offline sessions in the Keycloak
account console under applications/consents, or ask an administrator to end the user's
session or consent. Revoking a refresh token does not retroactively revoke a WebSocket
handshake or an already-issued access token; the router closes its own socket on sign-out,
and the server applies its own token expiry and authorization checks.

For a separate operator process set `UNIMIND_AUTH_MODE=developer-key` and
`UNIMIND_DEVELOPER_KEY_FILE` to a protected file containing **only** the key.
On Unix the file must be regular and owner-only (`chmod 600`); no symlinks.
The operator process requires WSS and either a configured CA or an already
established TLS pin. Do not put this path or the key in shared `router.json`;
set them on the dedicated MCP server entry. This mode does not read OAuth,
Entra or `UNIMIND_WS_TOKEN`, and does not drain normal-session queued writes.
Both explicit modes bypass legacy local read caches and offline write queues
so one mode cannot replay another mode's data or writes.
The operator router does not publish the OpenCode plugin local API, so
plugins cannot accidentally send background tool calls under its identity.
`router_info` and `/__router/metrics` report `auth_mode` and
`connection_auth_source` for the *current* WebSocket plus `oidc_state`; `auth_source` is the
most recent token lookup (including jLens). The two processes each have their
own jLens port. Your Unimind edge must independently accept both credential
types before this provides a recovery route.

`legacy` remains the default for existing configurations and retains the
fallback chain below. The new modes must be selected explicitly.

Every connection to Unimind carries `Authorization: Bearer <token>`, and Unimind validates it. The
router takes the first available token from this list:

0. **OAuth2 / OIDC, e.g. Keycloak**, if configured. This always wins over Entra:
   - `UNIMIND_OAUTH_TOKEN_FILE` or `KEYCLOAK_TOKEN_FILE`: a file containing the access token. It's
     **re-read on every use**, so keep it fresh with your IdP's tooling (`kcadm`, `oidc-agent`, a
     cron job, a sidecar).
   - `UNIMIND_OAUTH_TOKEN` or `KEYCLOAK_ACCESS_TOKEN`: the token itself.
   - JWTs that are expired or within 30 s of expiry are skipped, and the router falls through to
     the next source. Opaque tokens are passed as-is.
   - The Unimind deployment must **trust your IdP** (its issuer/JWKS and client id). Ask your
     Unimind admin. A token from an untrusted issuer is rejected with 401.
1. **Azure CLI** (`az account get-access-token`): the default; just run `az login`.
   `UNIMIND_ENTRA_USE_AZ=0` disables it.
2. **MSAL** (silent refresh, then device code), if `UNIMIND_ENTRA_TENANT` and
   `UNIMIND_ENTRA_CLIENT_ID` are set. The token cache is stored in `~/.unimind/token_cache.bin`.
3. Managed identity, on an Azure VM. Uses the `UNIMIND_ENTRA_CLIENT_ID` app, so the MSAL settings must be configured.
4. `--username` / `UNIMIND_PASSWORD` login, or a static `--token` / `UNIMIND_WS_TOKEN` (fallbacks for
   non-Entra deployments).

Tokens are kept in memory only (token files are read, never written). They're never written to
logs. The connect log shows which source was used, e.g. `Auth ready (0.0s) — token source: keycloak-file`
(also `auth_source` in `/__router/metrics`). Sources: `oauth-file`, `keycloak-file`, `oauth-env`,
`keycloak-env`, `entra-az`, `entra-msal`, `entra-managed-identity`, `password-login`, `static-token`.

> **Tunnel vs. Unimind sign-in.** The built-in Azure Bastion tunnel always authenticates with
> **Entra** (`az login`). Azure requires that. The token presented *to Unimind* can still come
> from Keycloak/OAuth2. If Unimind is reachable directly (VPN or private network), leave `tunnel`
> out and Entra isn't needed at all.

### Client id and install id

Every time the router starts it generates a random UUID (version 4), its **install id**, and keeps it
for as long as the process runs. It does not change when the connection drops and comes back, and a
router that is started again gets a new one.

The server is told two things on every connection:

| Header | Value | Example |
|---|---|---|
| `X-Client-ID` | `<install-uuid>:<your client id>` | `3f1c9a52-7e0b-4d2a-9c1e-5b8a6d0e4f17:opencode` |
| `X-Install-ID` | the UUID alone | `3f1c9a52-7e0b-4d2a-9c1e-5b8a6d0e4f17` |

`<your client id>` is whatever you pass as `--client-id` / `UNIMIND_CLIENT_ID`, exactly as you wrote it
(leading and trailing spaces are trimmed; it may not be empty or contain control characters). The router
no longer appends a process id and random suffix, because the UUID already makes each run unique.
`--client-id-exact` / `UNIMIND_CLIENT_ID_EXACT` is still accepted so existing launchers keep working, but
it has no effect.

The separate `X-Install-ID` lets the server record the UUID on every connection and on every usage
record from that connection, including usage reported by plugins that know nothing about it.

Locally the router keeps using your own client id (without the UUID) to choose the GitHub-token lookup
order and the OpenCode queue folder. `router_info` shows `client_id` (the full value), `install_id` and
`client_id_base` (your value).

### Changes to `router.json` and the authorization file

The router never keeps `router.json` or the shared authorization file (`~/.unimind/config/auth.json`)
open. It only checks their timestamps, size, owner and permissions every 2 seconds (set
`UNIMIND_WATCH_INTERVAL_MS`, 100 to 60000). When one changes and has stopped changing, it opens the
file, reads it and closes it again. `router_info` shows `file_watch` for both files, including
`"held_open": false`.

**`router.json`.** If the connection settings change, the router checks them and reconnects with the
new ones. It keeps the same install UUID, so the server sees the same router moving, not a new one.

| Setting in `router.json` | When the file changes |
|---|---|
| `server`, `UNIMIND_SERVER` | Applied by reconnecting. A different server is also pinned afresh (first use). |
| `UNIMIND_CA_CERT`, `UNIMIND_TLS_SERVER_NAME`, `UNIMIND_CLIENT_CERT`, `UNIMIND_CLIENT_KEY` | Applied by reconnecting. |
| Everything else (ports, tunnel, project, auth mode, ...) | Reported as needing a router restart; not applied. |
| Anything also set on the command line or in the environment | The command line and environment still win, exactly as at startup. The change is reported as ignored. |

A change is ignored, with a warning, and the router keeps its previous settings if the file is not
valid JSON, is a symbolic link, is not owned by you, is writable by everyone, is larger than 256 KiB,
or the new settings fail the same safety checks as at startup (for example a missing CA file, plain
`ws://` to a remote host in a sign-in mode, or `developer-key` mode without a CA or TLS pin for the new
server). The router's own TLS-pin writes to this file do not trigger a reconnect.

**The authorization file.** The router tells you when it changes in a way that matters:

| Change | Alert |
|---|---|
| A different account (first 8 characters of the account id are shown) | warning |
| The file was removed, or you were signed out | warning |
| The file became unusable or unsafe (for example wrong permissions) | warning |
| The sign-in was replaced by another router or process (same account) | info |
| A new sign-in you just completed in this router, or the file is usable again | info |
| A routine token refresh (same account, same sign-in) | none |

Alerts go to the router log, to the IDE as a log notification (`notifications/message`, after the IDE
has initialised), and to the `alerts` list in `router_info` and `unimind_sign_in_status`. They never
contain a token. The router does not change its own sign-in because of an edit; it keeps using what it
has until it ends or refreshes.

### Credential handoff

**What this is.** Unimind runs council members and other server-side sub-agents on your behalf.
They call model and search providers (GitHub Copilot, OpenAI, xAI, Perplexity, Brave) using
**your** credentials, which exist only on your machine. The router hands them to the server for
the lifetime of the connection. This is separate from your Unimind sign-in (Keycloak, Entra or
the developer key), which authenticates you to Unimind and is never sent to providers.

**What can be handed off.** Nothing else is ever sent. Each row is one credential *family*:

| Family (argument name) | Where the router reads it | Used by the server for |
|---|---|---|
| `copilot_token` | GitHub/Copilot OAuth token, looked up in the sources below | Council members calling the Copilot API |
| `openai_api_key` | `OPENAI_API_KEY` | OpenAI model calls |
| `xai_api_key` | `XAI_API_KEY` | Grok (xAI) calls |
| `perplexity_api_key` | `PERPLEXITY_API_KEY`, else `PERPLEXITY_KEY` | Perplexity research |
| `brave_api_key` | `BRAVE_API_KEY`, else `BRAVE_KEY` | Brave web search |
| `ghost_brave_key` | `GHOST_BRAVE_KEY` | Brave search through the browser sidecar |
| `ghost_perplexity_key` | `GHOST_PERPLEXITY_KEY` | Perplexity search through the browser sidecar |

An empty variable counts as unset. The server mirrors the plain and "ghost" Brave and Perplexity
slots into each other, so allowing only `brave_api_key` still fills both server-side.

**Where the Copilot token is looked up**, in this order (it depends on the start of `--client-id`):

| `--client-id` starts with | Order |
|---|---|
| `copilot` | env, copilot-cli, copilot-auth-json, gh-keychain, gh-cli, opencode, vscode |
| `opencode` | env, opencode, copilot-cli, copilot-auth-json, gh-keychain, gh-cli, vscode |
| `vscode` | env, vscode, copilot-cli, gh-keychain, gh-cli, opencode |
| anything else | env, opencode, copilot-cli, copilot-auth-json, gh-keychain, vscode, gh-cli |

- `env`: `COPILOT_GITHUB_TOKEN`, `COPILOT_TOKEN`, `GITHUB_COPILOT_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`.
- `opencode`: OpenCode's `auth.json` (`~/.local/share/opencode/`, `$XDG_DATA_HOME/opencode/` or `%LOCALAPPDATA%\opencode\`).
- `copilot-auth-json`: `~/.copilot/auth.json`. `vscode`: `hosts.json` / `apps.json` under `github-copilot` config folders.
- `copilot-cli`, `gh-keychain`: Windows Credential Manager entries (Windows only).
- `gh-cli`: the output of `gh auth token --hostname github.com` (15-second limit).

The router only reads these locations; it never writes to them. A value must be 20 to 512
characters with no spaces or control characters, otherwise it is ignored.

**When it is sent.** After every successful connection, so again after each reconnect and server
restart, and once more (at most once a minute) if the server reports a missing Copilot token.
If nothing is found, nothing is sent. This happens in every sign-in mode.

#### Controls

| Flag | Environment variable | Meaning |
|---|---|---|
| `--forward-credentials <policy>` | `UNIMIND_WATCH_INTERVAL_MS` | How often (milliseconds, default 2000) the router checks `router.json` and the authorization file for changes. See *Changes to `router.json` and the authorization file*. |
| `UNIMIND_FORWARD_CREDENTIALS` | `auto` (default): every family it finds. `none`: nothing is read or sent. Or a comma list of families, e.g. `openai_api_key,brave_api_key`. |
| `--github-token-sources <list>` | `UNIMIND_GITHUB_TOKEN_SOURCES` | Comma list that replaces the Copilot lookup order above: `env`, `opencode`, `copilot-cli`, `gh-keychain`, `copilot-auth-json`, `vscode`, `gh-cli`. |

- A family that the policy does not allow is **not read at all**: no credential file is opened
  and `gh` is not run for it.
- An unknown family or source name, or mixing `none`/`auto` with other values, stops the router
  at startup with a message. A typo never silently forwards or withholds the wrong thing.
- The flag wins over the environment variable. The variable names can also go in `router.json`.

```bash
# Hand off nothing:
unimind-router --forward-credentials none

# Only the Copilot token, and only if you put it in the environment yourself:
unimind-router --forward-credentials copilot_token --github-token-sources env

# Only search keys:
unimind-router --forward-credentials brave_api_key,perplexity_api_key
```

#### How the credentials are protected

On your machine (the router):

- **Only over the authenticated connection.** They go to the server you configured, over the
  TLS-verified WSLink session, as a router-private request. Its reply is consumed by the router.
- **Hidden from the IDE and the model.** The `provide_credentials` tool is removed from the tool
  list, a model attempt to call it is refused locally and nothing is sent, and the Copilot,
  OpenAI, xAI and Perplexity key arguments are stripped from other tools' schemas. Credential
  values never appear on stdout (a harness test checks this).
- **Never logged.** Logs show only family names and the source label (for example
  `Injected credentials: copilot_token, openai_api_key` and `from source 'opencode'`).
- **Not stored by the router.** No queue, cache or telemetry holds them. They are read from
  their original location each time, and `router_info` reports only names.
- **Not passed to helper programs.** The `az` and `gh` programs the router starts get a scrubbed
  environment: `UNIMIND_PASSWORD`, `UNIMIND_WS_TOKEN`, every `*_API_KEY` variable and the Copilot
  token variables are removed. `GH_TOKEN` and `GITHUB_TOKEN` are kept on purpose, because `gh`
  may need them.
- **Not part of your sign-in.** Keycloak tokens (`~/.unimind/config/auth.json`), the developer
  key and TLS keys are never included in a credential handoff.

On the server (as described by the router protocol contract, Section 10; check your server version):

- They are held **in memory only**, tied to the exact connection that provided them, and used
  only for that session's sub-agents and provider calls.
- They are cleared when the connection closes, and are not written to the knowledge base or any
  database. The router provides them again after a reconnect.

#### What this does not protect

- **Plaintext configuration.** If you put API keys in an MCP entry's `environment` (for example
  in `opencode.json`), they sit in that file in clear text. Make the file owner-only, or set the
  variables from your secrets manager or a launcher script. The router has no key-file option for
  provider API keys.
- **The server is trusted with them.** While you are connected, whoever operates the Unimind server
  or its process memory can use them. Treat the server like any service you give an API key to.
- **They spend your quota.** Councils and sub-agents act with your credentials and count against
  your provider plan and cost. To cut access, turn the forwarding off and rotate the key at the
  provider; clearing them server-side only happens on disconnect.
- **The default forwards everything it finds, with no per-family prompt.** That is looser than the
  protocol contract, which says to forward nothing by default. Use `none` or an explicit list to
  match the contract; a future release may make forwarding opt-in.

#### Checking what is handed off

- Ask for **`router_info`**: `credential_forwarding` shows the policy, when the last handoff was,
  and which family names were sent. Values are never shown.
- The router log has `Injected credentials: <names>`, or `Credential forwarding is off` when disabled.
- The server logs `Session credentials stored (volatile): [<names>]`.

---

## Service identity: `dynwarden-device`

For a service that has no person to sign in (a coding agent, a build host), the router signs in as a dynwarden device with its **own private key**.

```
UNIMIND_AUTH_MODE=dynwarden-device
UNIMIND_OIDC_ISSUER=https://auth.example.com/realms/realm
unimind-router enrol --dynwarden-url https://dynwarden.example.com --oidc-issuer https://auth.example.com/realms/realm \
    [--dynwarden-service NAME] [--dynwarden-profile ROLE] [--force]
unimind-router unenrol
```

- **Config block (0.4.6+).** Instead of `UNIMIND_AUTH_MODE`, put this in `router.json`:
  ```json
  "enrol": {"dynwarden_url": "https://dynwarden.example.com", "issuer": "https://auth.example.com/realms/realm",
            "service": "outside-grok", "profile": "coding-agent"}
  ```
  With `dynwarden_url` populated the router is a dynwarden device, and `unimind-router enrol` needs no flags. Empty or absent: normal sign-in, unchanged. If the block is present but this machine is not enrolled, the router reports `not-enrolled` and waits; it never starts enrolment on its own. An explicit `UNIMIND_AUTH_MODE`, environment variable or flag overrides the block. Adding or removing the block while running is reported as needing a restart.
- `enrol` generates a P-256 key on this machine, sends only the public JWK to dynwarden, prints the user code and address for the owner to approve, polls, then stores the `dw-...` client id and the private key in the `dynwarden` section of `~/.unimind/config/auth.json` (0600, directory 0700, under the auth lock). A person's session in the same file is never touched. A second `enrol` is refused unless `--force`.
- At run time the router mints access tokens with a signed client assertion (ES256, `kid` = the key id dynwarden returned, 60 s lifetime, fresh `jti`, audience = token endpoint) via `client_credentials`. There is no refresh token; tokens stay in memory and are re-minted before expiry.
- **Fail closed:** it never falls back to Entra, MSAL, a password, `--token`, an OAuth token file or the developer key. `router_info.dynwarden` reports `ok`, `not-enrolled`, `setup-error`, `provider-unavailable`, `device-disabled`, `key-not-found` (key/kid mismatch) or `clock-skew`, without secrets. Removing or replacing the device in the file closes the socket and raises an alert.
- The server must accept `azp` starting `dw-` (it currently allow-lists only `unimind`).

## Environment variables

All of these can also go in `router.json` as `"NAME": "value"`.

### Connection

| Variable | Default | Meaning |
|---|---|---|
| `UNIMIND_SERVER` | `wss://127.0.0.1:8443/ws` | Same as `server` |
| `UNIMIND_CLIENT_ID` | `unimind-router` | The name you give this client (for example `opencode`). The server sees it as `<install-uuid>:<name>` (see *Client id and install id*). Its prefix picks the GitHub-token lookup order. |
| `UNIMIND_PROJECT` | (none) | Project scope for the automatic `session_init` |
| `UNIMIND_CA_CERT`, `UNIMIND_TLS_SERVER_NAME` | (none), `unimind.internal` | Verify the server certificate. Without a CA, the chain is unverified for loopback (tunnel) endpoints; the TOFU pin still applies. |
| `UNIMIND_TLS_PIN` | `1` | TLS certificate pinning (TOFU). Same as `tls_pinning`. `unimind-router --repin [--yes]` accepts a changed certificate. |
| `UNIMIND_CLIENT_CERT`, `UNIMIND_CLIENT_KEY` | (none) | mTLS client certificate |
| `UNIMIND_HEARTBEAT_INTERVAL` / `_TIMEOUT` | `20` / `60` s | WSLink heartbeat |
| `UNIMIND_LIVENESS_INTERVAL_S` / `UNIMIND_LIVENESS_FAILURES` | `15` / `3` | L1–L3 probes; force a reconnect after N consecutive failures |
| `UNIMIND_RECONNECT_DELAY` | `5` | Base reconnect delay in seconds (exponential, capped at 120 s, with jitter) |
| `UNIMIND_MAX_RECONNECT_FAILURES` | unset (never give up) | Exit after N consecutive failures |
| `UNIMIND_WSLINK_DISABLED` | `0` | Use plain-text WebSocket frames (debugging) |

### Ports and tunnel

| Variable | Meaning |
|---|---|
| `UNIMIND_JLENS_PORT`, `UNIMIND_TABLET_PORT` | Same as `ports.jlens` / `ports.tablet` |
| `UNIMIND_TUNNEL` | Whole `tunnel` object as JSON (overrides router.json) |
| `UNIMIND_TUNNEL_CONFIG` | Path to a JSON file holding just the `tunnel` object |
| `UNIMIND_TUNNEL_MODE`, `_SUBSCRIPTION`, `_BASTION`, `_BASTION_RG`, `_TARGET_IP`, `_FORWARDS` (comma list) | Override single tunnel fields |
| `UNIMIND_TUNNEL_VM`, `_VM_RG`, `_VM_ID`, `_LOCAL_SSH_PORT`, `_SSH_USER`, `_SSH_KEY` | `bastion-ssh` fields |
| `UNIMIND_TUNNEL_RETRY_INTERVAL_S`, `UNIMIND_TUNNEL_NETWORK_PROBE` | Retry policy |
| `UNIMIND_TUNNEL_AZ_BIN`, `UNIMIND_TUNNEL_SSH_BIN` | Explicit `az` / `ssh` executables |

### Sign-in

| Variable | Meaning |
|---|---|
| `UNIMIND_AUTH_MODE` | `legacy` (default: the fallback chain above), `oidc-device` (Keycloak device sign-in only) or `developer-key` (operator key file only). Same as `--auth-mode`. |
| `UNIMIND_OIDC_ISSUER` | Keycloak realm issuer URL for `oidc-device`, e.g. `https://<keycloak-host>/realms/<realm>` (HTTPS required) |
| `UNIMIND_OIDC_CLIENT_ID` | Public OAuth client for `oidc-device` (default `unimind`) |
| `UNIMIND_WORKSPACE_ROOT` | Directory (or comma-separated list of directories) below which the server may read files from this workstation (`client/read_file`). Same as `--workspace-root`. Never defaults to the working directory; with no root configured every server read request is refused immediately. |
| `UNIMIND_REVERSE_READ_ROOTS` | Extra allowed roots for server reads: comma-separated, or separated like `PATH` (`:` on Linux/macOS); both may be mixed and spaces are trimmed. Paths containing a comma are not supported. Missing directories and `/` are ignored. |
| `UNIMIND_REVERSE_READ_MAX_BYTES` | Size cap for one server read (default 2 MiB). Only regular UTF-8 files are returned; symlinks in any path component, directories, FIFOs and other special files are refused. Not supported on Windows (refused). |
| `UNIMIND_DEVELOPER_KEY_FILE` | Owner-only file holding the operator key for `developer-key` |
| `UNIMIND_OAUTH_TOKEN_FILE`, `KEYCLOAK_TOKEN_FILE` | OAuth2/Keycloak access-token file (re-read on every use) |
| `UNIMIND_OAUTH_TOKEN`, `KEYCLOAK_ACCESS_TOKEN` | OAuth2/Keycloak access token |
| `UNIMIND_ENTRA_USE_AZ` | `0` = don't use the Azure CLI |
| `UNIMIND_ENTRA_AZ_RESOURCE` | Token resource (default `https://management.core.windows.net/`) |
| `UNIMIND_ENTRA_AZ_TIMEOUT` | Seconds to wait for `az` (default 20) |
| `UNIMIND_ENTRA_TENANT`, `UNIMIND_ENTRA_CLIENT_ID`, `UNIMIND_TOKEN_CACHE` | MSAL sign-in |
| `UNIMIND_TOKEN_REFRESH_SKEW_SEC` | Refresh this many seconds before expiry |
| `UNIMIND_USERNAME` + `UNIMIND_PASSWORD`, `UNIMIND_WS_TOKEN` | Non-Entra fallbacks. Put the password in the environment only, never on the command line. |
| `UNIMIND_GITHUB_TOKEN_SOURCES` | GitHub token lookup order; same as `--github-token-sources` (see *Credential handoff*) |
| `UNIMIND_FORWARD_CREDENTIALS` | Which credentials are handed to the server: `auto` (default), `none`, or a comma list; same as `--forward-credentials` (see *Credential handoff*) |

### Files, uploads and forwarding

| Variable | Default | Meaning |
|---|---|---|
| `UNIMIND_UPLOAD_ROOTS` | cwd, workspace, temp | Extra folders (`;` or `,` separated) the AI may upload files from. Secrets folders (`.ssh`, `.azure`, keys, `.env`) are always refused. |
| `UNIMIND_LOCAL_API_TOOLS` | `kg_add,checkpoint_save,ingest_transcript,cache_file,report_usage` | Tools that same-user local companions (the OpenCode plugins) may call through the router's local tool API, `POST /__router/tools/call` on the jLens port. Calls need the token from the discovery file and are refused from browsers (`Origin` header). `off` disables the API. See [plugins/opencode](../plugins/opencode/README.md). |
| `UNIMIND_LOCAL_API_FILE` | `~/.unimind/run/local-api-<pid>.json` (one per router) | Discovery file (URL + token, user-only). Each router writes its own and removes it on exit; files of routers that died are removed by the next router to start. Setting this pins one fixed file instead. |
| `UNIMIND_UPLOAD_ALLOW_ANY` | `0` | Lift the upload folder restriction (not recommended) |
| `UNIMIND_RECV_DIR` | `~/.opencode/workspace` | Where server-pushed files land |
| `UNIMIND_LOCAL_BACKUP_DIR` | `~/.unimind/backups` | Where server backups are delivered |
| `UNIMIND_ALLOW_REVERSE_FORWARD` + `UNIMIND_REVERSE_FORWARD_PORTS` | off | Let the server reach specific **loopback** ports on your machine, e.g. `9222` for authenticated browsing through your Chrome. Only listed ports are allowed. |
| `UNIMIND_SSH_AGENT_FORWARD` (+ `_ENDPOINT`, `_ALLOWED_KEYS`) | off | Forward your SSH agent so the server can sign with it (e.g. `git clone`). The server can never add or remove keys. `_ALLOWED_KEYS` restricts which fingerprints it may use. |

### Caches, logs, telemetry

| Variable | Default | Meaning |
|---|---|---|
| `UNIMIND_LOG_FILE` | stderr | Log file path (recommended: MCP hosts often discard stderr) |
| `UNIMIND_LOG_LEVEL` | `info` | `error`, `warn`, `info`, `debug`, `trace` |
| `UNIMIND_RESILIENCE_DIR` | `~/.unimind` | Offline write queue and read cache |
| `UNIMIND_CACHE_TTL`, `UNIMIND_CACHE_MAX_ROWS`, `UNIMIND_L0_TTL`, `UNIMIND_L0_MAX` | 300 s, 2000, 300 s, 512 | Read caches. Classified results are never cached on disk. |
| `UNIMIND_MAX_QUEUE` | 500 | Offline write queue cap |
| `UNIMIND_TELEMETRY` (+ `_DIR`, `_FLUSH_INTERVAL`) | on | Local-only usage/latency telemetry (SQLite). `0` = off. |
| `UNIMIND_WIN_DIAG` | `0` | Extra Windows firewall diagnostics on connection failures |
