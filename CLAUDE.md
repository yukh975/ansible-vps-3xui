# Project Memory for Claude Code

## Project

Ansible playbook for deploying VPS nodes with 3x-ui and Caddy.

Target OS: Debian 12/13. Managed from macOS or Linux (Ansible control node).

## Architecture

### Two-stage deployment

- **site-init.yml** — runs once on a fresh server (connection: port 22, user root). After completion: root is locked, SSH moves to `ssh_port`. Cannot be re-run.
- **site-configure.yml** — idempotent, can be re-run (connection: `ssh_port`, user `deploy_user`).

### Traffic flow

- x-ray listens on 443 (VPN traffic)
- Caddy: port 80 (ACME HTTP-01 challenge), port `caddy_https_port`/8443 (HTTPS, closed externally), port `127.0.0.1:caddy_listen_port`/4443 (fallback from x-ray)
- cert-sync.sh copies certs from Caddy storage to `certs_dest_dir`, then restarts x-ui
- cert-sync runs via ExecStartPost with root privileges (`+` prefix in systemd drop-in)

### Post-reboot sequence (configure.yml)

After final reboot: stop x-ui → wait for Caddy cert (up to 5 min) → run cert-sync → start x-ui. Playbook only completes when both x-ui and Caddy are active.

## Key Variables

| Variable | Default | Description |
|---|---|---|
| `hostname` | — | FQDN — used for both server hostname and ACME certificate (replaces old `domain`) |
| `ssh_port` | — | SSH port after hardening |
| `deploy_user` | — | User for stage 2 (must be in `users`) |
| `acme_email` | — | Email for Let's Encrypt |
| `certs_dest_dir` | — | Directory where cert-sync places certificates |
| `caddy_https_port` | `8443` | Caddy HTTPS port (not 443 — taken by x-ray) |
| `caddy_listen_port` | `4443` | Caddy fallback port (localhost only, receives traffic from x-ray) |
| `caddy_fallback_url` | — | External camouflage site |
| `install_3xui` | `true` | Install 3x-ui |
| `xui_version` | `""` (latest) | 3x-ui version |

Variables removed in v0.9.0: `domain` (replaced by `hostname`), `install_caddy` (Caddy is now mandatory).

## Package Installation Notes

- `ipset-persistent` must be in `extra_packages` — provides the `10-ipsets` plugin for `netfilter-persistent`, which saves/restores ipset sets before iptables rules on boot (plugin order: 10-ipsets → 15-ip4tables → 25-ip6tables).
- apt task uses `DEBIAN_FRONTEND=noninteractive` — `ipset-persistent` asks an interactive question during install ("save current rules?") which would block Ansible without this.

## No ansible.posix Dependency

All modules are `ansible.builtin`. Removed in v0.8.0:
- `authorized_key` replaced by `copy`
- `sysctl` replaced by `template + command`

No `requirements.yml` needed.

## Important Files

- `group_vars/new_vps.yml` — not committed (`.gitignore`); contains secrets (password hashes, SSH keys)
- `group_vars/new_vps.yml.example` — template, committed to git
- `roles/bootstrap/files/x-ui.db` — not committed (`.gitignore`); copy from reference server with `scp root@<IP>:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db`
- `roles/bootstrap/files/.gitkeep` — keeps the `files/` directory in git

## Database Path Updates

After uploading `x-ui.db`, an Ansible sqlite3 task automatically updates `webCertFile` and `webKeyFile` in the database using `certs_dest_dir` and `hostname`. Inbound TLS paths must be checked manually in the x-ui panel.

## Testing / Idempotency

- `site-init.yml` — NOT idempotent. Run once per server. Re-running fails because SSH port 22 and root login are closed.
- `site-configure.yml` — idempotent. Safe to re-run multiple times for config updates.
