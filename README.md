# unimind-client

`unimind-router` is a single executable that connects your AI coding tool
(OpenCode, GitHub Copilot CLI, VS Code, Claude Code) to a Unimind server.

Your tool starts it as a local MCP server over stdio. The router then:

- authenticates you to Unimind with **Entra ID**, **Keycloak** or any **OAuth2/OIDC** token, with no passwords in config files
- connects directly, or opens and maintains an Azure Bastion tunnel when there's no direct path (no SSH or VPN needed)
- pins the server's TLS certificate on first use and refuses connections if it changes unexpectedly
- reconnects automatically after sleep, network loss or server restarts
- serves **jLens** (the knowledge explorer) at `http://127.0.0.1:18080/jlens`; with several IDE sessions open, each gets its own port (18081, 18082, …) and the `router_info` tool tells you which one is yours
- serves a per-session **tablet** on a local port for interfaces Unimind generates

It's one static file with no runtime dependencies.

## Download

Get the binary for your platform from the [latest release](https://github.com/mswilliamrude/unimind_client/releases/latest):

| Platform | File |
|---|---|
| Windows 10/11 x64 | `unimind-router-windows-x86_64.exe` |
| Linux x64 (any distro, static) | `unimind-router-linux-x86_64` |
| Linux arm64 (any distro, static) | `unimind-router-linux-aarch64` |
| macOS 11.7+ Apple Silicon | `unimind-router-macos-aarch64` |
| macOS 11.7+ Intel | `unimind-router-macos-x86_64` |

**Verify the download** against `SHA256SUMS` from the same release:

```bash
sha256sum -c SHA256SUMS --ignore-missing                     # Linux
shasum -a 256 -c SHA256SUMS --ignore-missing                 # macOS
```
```powershell
(Get-FileHash .\unimind-router-windows-x86_64.exe).Hash      # Windows: compare with SHA256SUMS
```

**Install** it somewhere stable as `unimind-router`:

```powershell
# Windows
New-Item -ItemType Directory -Force "$env:USERPROFILE\.unimind\bin" | Out-Null
Move-Item .\unimind-router-windows-x86_64.exe "$env:USERPROFILE\.unimind\bin\unimind-router.exe"
Unblock-File "$env:USERPROFILE\.unimind\bin\unimind-router.exe"
```
```bash
# Linux / macOS (pick your file)
install -Dm755 unimind-router-linux-x86_64 ~/.unimind/bin/unimind-router
# macOS only: the binaries aren't notarized, so clear the download quarantine flag
xattr -d com.apple.quarantine ~/.unimind/bin/unimind-router 2>/dev/null || true
```

The binaries aren't code-signed, so Windows SmartScreen may warn on first run. Check the
SHA-256 first, then choose *More info → Run anyway*.

## Quick start

1. **Sign in to Azure:** `az login` (the account that has access to the Unimind Bastion).
2. **Create `~/.unimind/router.json`** (Windows: `%USERPROFILE%\.unimind\router.json`):
   ```json
   {
     "server": "wss://127.0.0.1:8443/ws",
     "tunnel": {
       "enabled": true,
       "mode": "bastion-ip",
       "subscription": "<subscription name or id>",
       "bastion_name": "<bastion name>",
       "bastion_resource_group": "<bastion resource group>",
       "target_ip": "<Unimind edge private IP>",
       "forwards": ["8443:3389"]
     }
   }
   ```
   Your Unimind admin can give you the four `<...>` values. More examples are in [examples/](examples/).
3. **Add it to your tool**. For OpenCode (`~/.config/opencode/opencode.json`):
   ```json
   { "mcp": { "unimind": { "type": "local",
       "command": ["C:/Users/<you>/.unimind/bin/unimind-router.exe", "--client-id", "opencode"] } } }
   ```
   For other tools, see [docs/HOSTS.md](docs/HOSTS.md).
4. **Restart your tool.** Unimind's tools appear within a few seconds. To check it's working, open
   `http://127.0.0.1:18080/__router/metrics`: it should show `"connected": true`.

## TLS certificate pinning

On the first connection the router saves the server certificate's SHA-256 fingerprint, with a
timestamp, in `router.json` under `"tls_pins"`. If the server later presents a different
certificate, the router **refuses to connect** and logs both fingerprints and what to do.

If the Unimind server's certificate was legitimately replaced, confirm the new fingerprint with
your Unimind admin, then run:

```bash
unimind-router --repin
```

It shows the old and new fingerprints and asks before saving. Details are in
[docs/CONFIGURE.md](docs/CONFIGURE.md#tls-certificate-pinning-tofu).

## Documentation

| Guide | What it covers |
|---|---|
| [docs/CONFIGURE.md](docs/CONFIGURE.md) | Full `router.json` reference: server, ports, tunnel, retry policy, TLS pinning, auth, environment variables |
| [docs/HOSTS.md](docs/HOSTS.md) | Wiring the router into OpenCode, Copilot CLI, VS Code and Claude Code |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Diagnosing connection, tunnel, certificate and sign-in problems |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | How the client talks to Unimind: components, protocol layers, data-flow diagrams |
| [examples/](examples/) | Ready-to-edit config files |

## License

Licensed under either of [Apache License 2.0](LICENSE-APACHE) or [MIT](LICENSE-MIT), at your option.
