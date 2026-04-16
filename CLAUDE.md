# Project Memory for Claude Code

## Project

Ansible playbook for deploying VPS nodes with 3x-ui and Caddy.

Target OS: Debian 12/13. Managed from macOS or Linux (Ansible control node).

**Supported xray transport: VLESS/VMess/Trojan over WebSocket only.** Caddy on 443 reverse-proxies the WebSocket path to `localhost:xray_port`. Other transports (TCP, Reality, gRPC, mKCP, QUIC, xHTTP) do not work with this setup — they require a different architecture (e.g. xray listening on 443 directly without Caddy).

**TLS handling — critical operational gotcha.** TLS terminates at Caddy on 443; traffic between Caddy and xray on localhost is unencrypted by design, so the 3x-ui inbound has TLS disabled. As a result, links the panel exports contain `security=none`, which the client must manually change to `security=tls` — otherwise the client won't negotiate TLS with Caddy and the connection fails. This is documented prominently in both READMEs and INSTALL files (step 2.6). Don't "fix" the inbound by re-enabling TLS — that would double-encrypt and break things.

## Architecture

### Two-stage deployment

- **site-init.yml** — runs once on a fresh server (connection: port 22, user root). After completion: root is locked, SSH moves to `ssh_port`. Cannot be re-run.
- **site-configure.yml** — idempotent, can be re-run (connection: `ssh_port`, user `deploy_user`).

### Traffic flow

- Caddy listens on 443 (TLS via ACME, port 80 for HTTP-01 challenge)
- `xui_panel_path/*` → reverse proxy → `localhost:xui_panel_port` (x-ui panel, HTTP)
- `xray_ws_path/*` → WebSocket proxy → `localhost:xray_port` (xray/VPN traffic)
- All other traffic → redirect to `caddy_fallback_url` (camouflage site)
- x-ui panel runs on localhost HTTP only — TLS terminates at Caddy, no cert-sync needed

### Post-reboot sequence (configure.yml)

After final reboot: wait for Caddy to obtain ACME certificate (up to 5 min) → start x-ui. Playbook only completes when both x-ui and Caddy are active.

## Key Variables

| Variable | Default | Description |
|---|---|---|
| `hostname` | — | FQDN — used for both server hostname and ACME certificate (replaces old `domain`) |
| `ssh_port` | — | SSH port after hardening |
| `deploy_user` | — | User for stage 2 (must be in `users`) |
| `acme_email` | — | Email for Let's Encrypt |
| `caddy_fallback_url` | — | External camouflage site |
| `xui_panel_path` | — | Secret path to x-ui panel (starts with `/`; `*` appended in template) |
| `xui_panel_port` | `54321` | x-ui panel port on localhost (reverse-proxied via Caddy) |
| `xray_ws_path` | — | Secret WebSocket path for xray (starts with `/`; `*` appended in template) |
| `xray_port` | `10000` | xray WebSocket port on localhost |
| `install_3xui` | `true` | Install 3x-ui |
| `xui_version` | `""` (latest) | 3x-ui version |

Variables removed in v0.9.0: `domain` (replaced by `hostname`), `install_caddy` (Caddy is now mandatory).

## Package Installation Notes

- `ipset-persistent` is installed automatically (hardcoded in init.yml) — provides the `10-ipsets` plugin for `netfilter-persistent`, which saves/restores ipset sets before iptables rules on boot (plugin order: 10-ipsets → 15-ip4tables → 25-ip6tables).
- apt task uses `DEBIAN_FRONTEND=noninteractive` — `ipset-persistent` asks an interactive question during install ("save current rules?") which would block Ansible without this.

## No ansible.posix Dependency

All modules are `ansible.builtin`. Removed in v0.8.0:
- `authorized_key` replaced by `copy`
- `sysctl` replaced by `template + command`

No `requirements.yml` needed.

## Multi-Server Deployment

Supported natively via Ansible's group/host vars. Layout:

- `group_vars/new_vps.yml` — shared across all hosts in the group (users, ssh_port, acme_email, packages, sysctl, firewall)
- `host_vars/<host>.yml` — per-host overrides. **`hostname` must be per-host** — each server gets its own TLS certificate from Let's Encrypt via HTTP-01; two hosts can't share one FQDN without DNS-01 (not implemented).
- `inventory.ini` lists all hosts in `[new_vps]`; left column (e.g. `server1`) must match the `host_vars/<name>.yml` filename.

Ansible runs hosts in parallel (default 5 forks). Preconditions: SSH key pre-uploaded to each root (`ssh-copy-id`), DNS A-record per FQDN, same `x-ui.db` for all.

## Important Files

- `group_vars/new_vps.yml` — not committed (`.gitignore`); contains secrets (password hashes, SSH keys)
- `group_vars/new_vps.yml.example` — template, committed to git
- `host_vars/<host>.yml` — not committed (`.gitignore`); per-host vars, at minimum `hostname`
- `host_vars/server1.yml.example` — template, committed
- `roles/bootstrap/files/x-ui.db` — not committed (`.gitignore`); copy from reference server with `scp root@<IP>:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db`
- `roles/bootstrap/files/.gitkeep` — keeps the `files/` directory in git

## Database Path Updates

After uploading `x-ui.db`, an Ansible sqlite3 task automatically clears `webCertFile` and `webKeyFile` (TLS is handled by Caddy), sets `webListen` to `127.0.0.1`, and sets `webBasePath` to `xui_panel_path`. Inbound TLS paths must be checked manually in the x-ui panel.

## Testing / Idempotency

- `site-init.yml` — NOT idempotent. Run once per server. Re-running fails because SSH port 22 and root login are closed.
- `site-configure.yml` — idempotent. Safe to re-run multiple times for config updates.
