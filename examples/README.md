# Example configs

Copy one to `%USERPROFILE%\.unimind\router.json` (Windows) or `~/.unimind/router.json` (Linux),
then fill in the `<...>` values.

| File | Use when |
|---|---|
| `router.json` | **Default.** Built-in Bastion tunnel straight to the edge (`bastion-ip`), all retry settings shown |
| `router.bastion-ssh.json` | Your Bastion doesn't have IP-based connect, so the tunnel goes through a jump VM with SSH |
| `router.no-tunnel.json` | You reach the server directly (VPN, private network, or your own tunnel) |

Host (AI tool) snippets are in [../docs/HOSTS.md](../docs/HOSTS.md). Every key is explained in
[../docs/CONFIGURE.md](../docs/CONFIGURE.md).
