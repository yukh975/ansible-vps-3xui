# ansible-vps

Ansible playbook for rapid deployment of a VPS node with 3x-ui and Caddy.

Takes a freshly installed Debian 12/13 with root SSH access and turns it into a production-ready node in two stages: hardened SSH, firewall, users, [3x-ui](https://github.com/MHSanaei/3x-ui) with a database from a reference server, and Caddy as a reverse proxy with automatic TLS certificates via Let's Encrypt.

---

## Quick Start

Full step-by-step instructions: **[INSTALL_EN.md](INSTALL_EN.md)**

```
Step 1 — once on the control node: clone the repo, configure vars, copy x-ui.db
Step 2 — per server: run site-init.yml, then site-configure.yml
```

---

## How It Works

### Stage 1 — `site-init.yml` (connection: port 22, user root)

Runs **once** on a fresh server. After it completes, root is locked and SSH moves to a custom port — re-running is not possible.

What it does:
- `apt full-upgrade`
- Package installation
- User creation (SSH keys, passwords, sudo)
- Hostname setup
- SSH hardening (custom port, root locked, key-only auth)
- sysctl, ipset, iptables
- Reboot

### Stage 2 — `site-configure.yml` (connection: `ssh_port`, `deploy_user`)

Can be re-run. Idempotent.

What it does:
- Install 3x-ui + upload `x-ui.db`
- Install Caddy + deploy Caddyfile
- Deploy `cert-sync.sh` (automatic certificate synchronization)
- Final reboot → Caddy obtains a TLS certificate via Let's Encrypt

> **After a successful deployment**, `deploy_user` is no longer needed. It is recommended to delete it manually:
> ```bash
> sudo userdel -r <deploy_user>
> sudo rm -f /etc/sudoers.d/<deploy_user>
> ```

### Certificate Management (automatic)

```
Let's Encrypt → HTTP-01 challenge (port 80) → Caddy obtains cert
→ cert-sync.sh copies to /etc/ssl/<hostname>/ → x-ui restarts
```

On auto-renewal (every ~60 days) everything happens automatically: Caddy renews the certificate → cert-sync is triggered via the systemd `ExecStartPost` hook.

---

## Project Structure

```
ansible-vps/
├── site-init.yml                    # Stage 1 (port 22, root)
├── site-configure.yml               # Stage 2 (custom port, deploy_user)
├── inventory.ini.example            # Inventory template
├── ansible.cfg
├── group_vars/
│   ├── new_vps.yml.example          # Variables template (committed to git)
│   └── new_vps.yml                  # Your variables (in .gitignore — not committed!)
├── roles/bootstrap/
│   ├── tasks/
│   │   ├── main.yml                 # Phase dispatcher
│   │   ├── upgrade.yml              # apt full-upgrade
│   │   ├── init.yml                 # Users, SSH, firewall
│   │   └── configure.yml            # 3x-ui, Caddy, cert-sync
│   ├── handlers/main.yml
│   ├── templates/
│   │   ├── sshd_config.j2
│   │   ├── iptables_v4.j2
│   │   ├── iptables_v6.j2
│   │   ├── Caddyfile.j2
│   │   ├── cert-sync.sh.j2          # Certificate sync script
│   │   └── caddy-override.conf.j2   # systemd drop-in (ExecStartPost)
│   └── files/                       # Files from the reference server (in .gitignore!)
│       └── x-ui.db                  # the only file taken from the reference server
├── INSTALL.md                       # Step-by-step instructions
└── CHANGELOG.md
```

---

## Variables

All settings are in `group_vars/new_vps.yml`. Template: `group_vars/new_vps.yml.example`.

### Server and Access

| Variable | Example | Description |
|---|---|---|
| `hostname` | `vps1.example.com` | Server FQDN — used as hostname and for the ACME certificate |
| `ssh_port` | `275` | SSH port after hardening |
| `deploy_user` | `myuser` | User for stage 2 (must be listed in `users`) |

### Users

```yaml
users:
  myuser:
    groups: ["sudo"]
    shell: /bin/bash
    password: "$6$..."           # openssl passwd -6 'YOUR_PASSWORD'
    sudo_nopasswd: true
    ssh_public_keys:             # public keys: cat ~/.ssh/id_ed25519.pub
      - "ssh-ed25519 AAAA... ansible"
      - "ssh-ed25519 AAAA... laptop"   # multiple keys supported
  root:
    password: "$6$..."             # requires files/root/.bashrc
```

### Certificates

```yaml
acme_email: "admin@example.com"          # email for Let's Encrypt
certs_dest_dir: "/etc/ssl/{{ hostname }}"  # where cert-sync places the certificates
```

> Ansible automatically updates the certificate paths in `x-ui.db` after copying it —
> no changes needed on the reference server.

### Caddy

| Variable | Default | Description |
|---|---|---|
| `acme_email` | — | Email for Let's Encrypt (required) |
| `caddy_listen_port` | `4443` | Port where x-ray sends fallback traffic |
| `caddy_fallback_url` | — | External camouflage site for fallback traffic |

### Firewall

```yaml
allowed_tcp_ports:
  - 80    # HTTP (ACME HTTP-01 challenge)
  - 443   # HTTPS (x-ray inbounds)

ipset_set_name: "allowed_hosts"   # ipset name
ipset_hosts:                       # full access for these IPs
  - "1.2.3.4"
```

Additional iptables rules can be added directly to the templates:
```
roles/bootstrap/templates/iptables_v4.j2
roles/bootstrap/templates/iptables_v6.j2
```
Default policies (`INPUT DROP`, `FORWARD DROP`, `OUTPUT ACCEPT`), loopback, ESTABLISHED, and ICMP are already defined.

### Miscellaneous

| Variable | Default | Description |
|---|---|---|
| `install_3xui` | `true` | Install 3x-ui |
| `xui_version` | `""` (latest) | 3x-ui version |
| `sysctl_settings` | see config | Kernel parameters |
| `extra_packages` | see config | Additional packages |

---

## Ports

```
22              → stage 1 only (root, temporary)
ssh_port        → SSH after hardening
80              → Caddy (ACME challenge)
443             → x-ray (VPN inbounds)
127.0.0.1:4443  → Caddy (fallback from x-ray, localhost only)
```

---

## Security

- `group_vars/new_vps.yml` — password hashes, **not committed** (`.gitignore`)
- `roles/bootstrap/files/` — x-ui DB — **not committed**
- `inventory.ini` — real IPs — **not committed**
- Recommended: `ansible-vault encrypt group_vars/new_vps.yml`

---

## Requirements

- **Ansible** 2.12+ (`brew install ansible` on macOS)
- **Target server:** Debian 12 or 13, root SSH access on port 22
- **DNS:** the domain must point to the server IP **before running stage 2** (required for ACME)
