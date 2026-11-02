# How unimind-router connects to Unimind

This document describes how the client (`unimind-router`) connects to and exchanges data with
**Unimind**. It covers the client's components, the protocol layers, how connectivity and
authentication work (direct, or through an Azure Bastion tunnel), tool calls, jLens, tunnel
recovery and liveness.

Unimind is treated as one service behind a TLS endpoint. Its internals are out of scope here.

Each diagram is shown as Mermaid (it renders on GitHub and Azure DevOps) and also available as a
JPG in [`diagrams/`](diagrams/) (Mermaid sources: `diagrams/*.mmd`).

> Addresses and names are placeholders. The real values for a deployment go in `router.json`
> ([CONFIGURE.md](CONFIGURE.md)).

---

## 1. Components

```mermaid
flowchart TB
  subgraph WS["User workstation"]
    direction TB
    HOST["AI tool<br/>OpenCode / Copilot CLI / VS Code / Claude Code"]
    BROWSER["Web browser"]
    subgraph RT["unimind-router (one process per AI-tool session)"]
      direction TB
      MCP["MCP stdio bridge<br/>JSON-RPC"]
      SESS["WSLink session<br/>chat / file / socket-proxy / ping"]
      AUTH["Token provider<br/>OAuth2 / Keycloak / Entra"]
      RES["Resilience<br/>caches, write queue"]
      JL["jLens proxy<br/>127.0.0.1:18080"]
      TAB["Tablet<br/>127.0.0.1:random"]
      SUP["Tunnel supervisor<br/>network gate + retry"]
    end
    AZ["az network bastion tunnel<br/>(only when no direct path)"]
    LP(("local endpoint<br/>127.0.0.1:8443"))
  end
  IDP["Identity provider<br/>Entra ID / Keycloak / any OAuth2-OIDC"]
  subgraph PATH["Connectivity (choose one)"]
    DIRECT["Direct<br/>VPN / private network"]
    BAS["Azure Bastion<br/>IP-based connect<br/>tunnel authenticated with Entra"]
  end
  UM(["Unimind<br/>TLS endpoint: /ws, /jlens, /health"])
  HOST <-- "stdio JSON-RPC" --> MCP
  BROWSER <-- "HTTP loopback" --> JL
  BROWSER <-- "HTTP loopback" --> TAB
  MCP --- SESS
  MCP --- RES
  SESS --- AUTH
  AUTH -. "obtain / refresh token" .-> IDP
  SUP -- "spawn / supervise" --> AZ
  SESS -- "wss + Bearer token" --> LP
  JL -- "https + Bearer token" --> LP
  LP --- AZ
  AZ == "Entra-authenticated tunnel" ==> BAS
  BAS == "TLS (end to end)" ==> UM
  SESS -. "or: wss://unimind-host/ws" .-> DIRECT
  DIRECT -. "TLS (end to end)" .-> UM
```

![01-components](diagrams/01-components.jpg)
*JPG: [diagrams/01-components.jpg](diagrams/01-components.jpg) - source: [diagrams/01-components.mmd](diagrams/01-components.mmd)*

| Part | Role |
|---|---|
| **AI tool** | Starts the router as a local MCP server over stdio |
| **unimind-router** | One process per AI-tool session. Bridges MCP to a WSLink session, obtains tokens, keeps the connection alive, and serves jLens and the tablet on loopback. |
| **Identity provider** | Issues the bearer token the router presents to Unimind: **Entra ID**, **Keycloak**, or any **OAuth2/OIDC** provider |
| **Connectivity** | **Direct** (VPN or private network) when Unimind is reachable, or the router's built-in **Azure Bastion tunnel** when it isn't |
| **Unimind** | One TLS endpoint: `/ws` (session), `/jlens` (web UI), `/health` |

**Two separate authentications**

1. **Tunnel** (only when there's no direct path): Azure Bastion admits the tunnel based on the
   **Entra** identity of `az login` and its Azure RBAC. This is an Azure requirement.
2. **Unimind:** every WebSocket and jLens request carries `Authorization: Bearer <token>`. The
   token can come from **OAuth2 / Keycloak** (a token file or environment variable) or **Entra**
   (`az`, MSAL, managed identity). Unimind validates it. The router never makes authorization
   decisions itself.

All router listeners bind to `127.0.0.1` only. Several sessions share one tunnel (the first router
opens it, the others reuse the healthy local port) and one jLens port.

---

## 2. Protocol layers

```mermaid
flowchart LR
  subgraph CL["unimind-router"]
    direction TB
    A1["MCP JSON-RPC 2.0<br/>tools/list, tools/call, ..."]
    A2["WSLink frames<br/>H chat, O/D/C file, s/d/c/e/w socket, P/W ping"]
    A3["WebSocket (binary)<br/>Authorization: Bearer token<br/>(OAuth2 / Keycloak / Entra)"]
    A4["TLS 1.2/1.3 to the Unimind certificate"]
    A1 --> A2 --> A3 --> A4
  end
  subgraph TN["Transport (one of)"]
    direction TB
    B1["Direct TCP<br/>VPN / private network"]
    B2["Azure Bastion tunnel<br/>local port to Bastion to Unimind<br/>(Azure TLS channel, Entra-authenticated)"]
  end
  UM(["Unimind<br/>terminates TLS, validates the bearer token"])
  A4 --> B1 --> UM
  A4 --> B2 --> UM
```

![02-protocol-stack](diagrams/02-protocol-stack.jpg)
*JPG: [diagrams/02-protocol-stack.jpg](diagrams/02-protocol-stack.jpg) - source: [diagrams/02-protocol-stack.mmd](diagrams/02-protocol-stack.mmd)*

- **End to end (router ↔ Unimind):** TLS. Inside it is a WebSocket carrying **WSLink** frames,
  and inside those are MCP JSON-RPC messages.
- **Bastion path only:** `az network bastion tunnel` carries that TLS stream inside Azure's own
  authenticated channel. Bastion relays it and can't read it.
- **Port 3389:** Bastion's IP-based connect only allows remote ports 22 and 3389, so Unimind also
  serves its TLS endpoint on 3389 for this path. It's TLS, not RDP.

**WSLink frame types used by the router**

| Byte | Name | Purpose |
|---|---|---|
| `H` | chat | One MCP JSON-RPC message (request, response or notification) |
| `O` / `D` / `C` / `S` / `V` / `A` / `N` / `Z` | file | Resumable file transfer (open, data, close, seek/resume, verify, ack/nak, done) |
| `s` / `d` / `c` / `e` / `w` | socket proxy | Reverse port-forward and SSH-agent channels (open, data, close, error, window credit). Off unless enabled. |
| `P` / `W` | ping / pong | L2 heartbeat |
| `R` / `Q` | ready | Session readiness |

**Endpoints the client uses**

| Endpoint | Use |
|---|---|
| `GET /ws` (WebSocket upgrade) | The session (long-lived) |
| `/jlens…` | jLens web UI and API, through the local proxy |
| `GET /health` | Liveness |

---

## 3. Connecting

```mermaid
sequenceDiagram
  autonumber
  participant H as AI tool (host)
  participant R as unimind-router
  participant I as Identity provider (Entra / Keycloak / OAuth2)
  participant A as az CLI
  participant B as Azure Bastion
  participant U as Unimind
  H->>R: spawn process (stdio)
  R->>R: load router.json
  H->>R: initialize
  R-->>H: initialize result (answered locally, no wait)
  opt no direct connectivity: built-in Bastion tunnel
    R->>R: network probe (DNS + TCP to Azure)
    R->>A: az network bastion tunnel --target-ip-address UNIMIND --resource-port 3389 --port 8443
    A->>B: open tunnel (Entra identity of az login)
    B-->>A: tunnel ready
    R->>R: local endpoint accepting = tunnel UP
  end
  alt OAuth2 / Keycloak token configured
    R->>R: read token file or env (re-read on each use, rotated externally)
  else Entra
    R->>A: az account get-access-token (or MSAL / managed identity)
    A->>I: token request
    I-->>A: JWT
    A-->>R: JWT
  end
  R->>U: TLS + GET /ws Upgrade, Authorization Bearer token (via tunnel or direct)
  U->>U: validate token, authorize the user
  U-->>R: 101 Switching Protocols
  R->>U: WSLink H: initialize, session setup (reconnect, credential handoff)
  U-->>R: ready
  R->>R: Ready, start L1/L2/L3 liveness probes
  H->>R: tools/list
  R->>U: H: tools/list
  U-->>R: tools
  R-->>H: tools (secret-bearing tools/args removed)
```

![03-connect-sequence](diagrams/03-connect-sequence.jpg)
*JPG: [diagrams/03-connect-sequence.jpg](diagrams/03-connect-sequence.jpg) - source: [diagrams/03-connect-sequence.mmd](diagrams/03-connect-sequence.mmd)*

- **`initialize` is answered locally**, so the AI tool never waits on the network to start.
- **Tunnel (optional):** started only when configured, after a network probe confirms Azure is
  reachable. The tunnel counts as up when the local endpoint accepts connections.
- **Token:** an explicitly configured OAuth2/Keycloak token (`UNIMIND_OAUTH_TOKEN_FILE`,
  `KEYCLOAK_TOKEN_FILE`, `UNIMIND_OAUTH_TOKEN`, `KEYCLOAK_ACCESS_TOKEN`) takes precedence over
  Entra. Token files are re-read on every use, so an external tool can refresh them. Expired JWTs
  are skipped.
- **Credential handoff:** the user's GitHub/Copilot token and optional API keys go **directly to
  the session**, never through the AI model. The tool that does this is hidden from `tools/list`.
- **Restart detection:** if Unimind restarts, the router notices and re-handshakes automatically.

---

## 4. A tool call

```mermaid
sequenceDiagram
  autonumber
  participant H as AI tool
  participant R as unimind-router
  participant C as Local caches / queue
  participant U as Unimind
  H->>R: tools/call {name, arguments}
  alt cacheable read and cache hit (identity-scoped)
    R->>C: lookup
    C-->>R: cached result
    R-->>H: result
  else connected
    R->>U: WSLink H frame (JSON-RPC) over WSS
    U->>U: authorize and execute
    U-->>R: WSLink H frame (result)
    R->>C: store (unclassified reads only, never errors)
    R-->>H: result
  else disconnected and a write
    R->>C: enqueue with idempotency key
    R-->>H: queued (replayed in order after reconnect)
  end
```

![04-tool-call](diagrams/04-tool-call.jpg)
*JPG: [diagrams/04-tool-call.jpg](diagrams/04-tool-call.jpg) - source: [diagrams/04-tool-call.mmd](diagrams/04-tool-call.mmd)*

- **Reads** may be served from a local cache scoped to the signed-in identity. Errors and
  classified results are never written to disk.
- **Writes made while disconnected** are queued with an idempotency key and replayed in order
  after reconnect.
- **Unimind** decides what the user is allowed to do. The router only forwards the request.

---

## 5. jLens through the local proxy

```mermaid
sequenceDiagram
  autonumber
  participant W as Browser
  participant J as Router jLens proxy (127.0.0.1:18080)
  participant U as Unimind
  W->>J: GET http://127.0.0.1:18080/jlens/...
  J->>J: Host must be loopback:18080 (DNS-rebinding defence)
  J->>J: Origin/Referer required for non-GET (CSRF), path limited to /jlens
  J->>J: get bearer token silently (OAuth2 / Keycloak / Entra, never prompts)
  J->>U: HTTPS (via tunnel or direct), Authorization Bearer token
  U->>U: validate token, authorize the user
  U-->>J: response (streamed / chunked)
  J-->>W: piped byte-for-byte
```

![05-jlens](diagrams/05-jlens.jpg)
*JPG: [diagrams/05-jlens.jpg](diagrams/05-jlens.jpg) - source: [diagrams/05-jlens.mmd](diagrams/05-jlens.mmd)*

The browser never holds a token. The proxy:
- **Accepts loopback requests only:** strict `Host` check against DNS rebinding, and
  `Origin`/`Referer` required for state-changing requests (CSRF).
- **Adds the bearer token silently.** It never shows a sign-in prompt.
- **Streams the response back unchanged.**

The **tablet** listener (a random loopback port per session) uses the same loopback and `Host`
protections. It serves local pages only.

---

## 6. Tunnel supervision and recovery (Bastion path)

```mermaid
stateDiagram-v2
  [*] --> starting: router start
  starting --> up: forward port accepting
  starting --> retry_wait: start failed
  up --> recovering: az exited / port down or wedged / WS cannot reach server
  state recovering <<choice>>
  recovering --> waiting_network: network_probe fails
  recovering --> attempt: probe ok and no retry pending
  recovering --> retry_wait: probe ok and retry pending
  waiting_network --> attempt: probe ok again (immediate)
  waiting_network --> waiting_network: recheck every check_interval_s
  attempt --> up: tunnel listening
  attempt --> retry_wait: failed (next attempt in retry_interval_s, default 300s)
  retry_wait --> attempt: interval elapsed
  retry_wait --> waiting_network: probe fails
  attempt --> throttled: max_restarts_per_hour used
  throttled --> attempt: every retry_interval_s (never gives up)
  up --> [*]: router exits (tunnel tree killed)
```

![06-tunnel-states](diagrams/06-tunnel-states.jpg)
*JPG: [diagrams/06-tunnel-states.jpg](diagrams/06-tunnel-states.jpg) - source: [diagrams/06-tunnel-states.mmd](diagrams/06-tunnel-states.mmd)*

| Setting (`router.json` → `tunnel`) | Default | Effect |
|---|---|---|
| `check_interval_s` | 15 | How often tunnel health is checked |
| `network_probe` | `management.azure.com:443` | Gate before every attempt. If it fails, no attempt is made (`waiting_network`). |
| `retry_interval_s` | 300 | Wait after a failed attempt while the network is up (`retry_wait`) |
| `max_restarts_per_hour` | 20 | Beyond this, attempts continue every `retry_interval_s` (`throttled`). The router never gives up. |

- **Two triggers, one gate:** the health loop and the WebSocket reconnect loop both go through
  the same gated recovery.
- **Clean-up:** the `az` process tree belongs to the router and is cleaned up even if the router
  crashes (Windows Job Object).
- **Other processes:** the router never kills a process it didn't start.

---

## 7. Liveness

```mermaid
flowchart LR
  R["unimind-router"] -->|"L1 every 15s: WebSocket ping/pong"| E["WebSocket"]
  R -->|"L2: WSLink P (ping) / W (pong) frame"| W["WSLink session"]
  R -->|"L3: MCP ping JSON-RPC"| S["Unimind"]
  E -.-> F{"3 consecutive failures<br/>on any layer?"}
  W -.-> F
  S -.-> F
  F -->|yes| X["force reconnect<br/>(fail in-flight requests fast)"]
  X --> T["tunnel check (gated)<br/>then WebSocket reconnect with backoff"]
  F -->|no| OK["healthy, metrics updated"]
```

![07-liveness](diagrams/07-liveness.jpg)
*JPG: [diagrams/07-liveness.jpg](diagrams/07-liveness.jpg) - source: [diagrams/07-liveness.mmd](diagrams/07-liveness.mmd)*

| Layer | Probe | Detects |
|---|---|---|
| L1 | WebSocket ping/pong | Dead network path or wedged tunnel |
| L2 | WSLink `P`/`W` | Broken session on a live socket |
| L3 | MCP `ping` | Unimind not answering |

Probes run every 15 s by default. After 3 consecutive failures the router reconnects, and
in-flight requests fail immediately instead of hanging. Live values are at
`http://127.0.0.1:18080/__router/metrics`.

---

## 8. Security summary (client side)

| Concern | Control |
|---|---|
| Reaching Unimind | Direct (VPN or private network), or Azure Bastion: Entra identity + Azure RBAC, no public endpoint needed |
| Authenticating to Unimind | A bearer token from OAuth2 / Keycloak / Entra on every session and jLens request, validated by Unimind |
| Data in transit | TLS end to end to Unimind. On the Bastion path it also travels inside Azure's encrypted channel. |
| Server certificate | Verified when `UNIMIND_CA_CERT` is set. Otherwise unverified **only** for loopback (tunnel) endpoints, with a warning. |
| Secrets | Tokens are held in memory only (token files are read, never written) and never logged. Credentials are handed off out-of-band and hidden from the model. |
| Local listeners | `127.0.0.1` only, `Host`/`Origin` checks, concurrency caps, timeouts |
| Local data | Caches are identity-scoped. Classified results never go to disk. Uploads are limited to allowed folders, and secrets folders are always refused. |
