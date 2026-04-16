# ansible-vps

Ansible playbook for rapid deployment of a VPS node with 3x-ui and Caddy.

Takes a freshly installed Debian 12/13 with root SSH access and turns it into a production-ready node in two stages: hardened SSH, firewall, users, [3x-ui](https://github.com/MHSanaei/3x-ui) with a database from a reference server, and Caddy as a reverse proxy with automatic TLS certificates via Let's Encrypt.

> ⚠️ **This playbook targets xray VLESS + WebSocket.**
> Caddy on port 443 proxies WebSocket traffic at a secret path to xray (`localhost:xray_port`). This works for VLESS/VMess/Trojan with the `ws` (WebSocket) transport. Other transports (TCP, Reality, gRPC, mKCP, QUIC, xHTTP, etc.) will not pass through this reverse proxy — they need a different architecture (for example xray listening on 443 directly, without Caddy).
>
> Support for other connection types will come later — either in this repository or as separate Ansible scenarios.

> ❗ **IMPORTANT: after exporting a link from 3x-ui, change `security=none` to `security=tls`**
>
> TLS is terminated at Caddy (port 443, Let's Encrypt certificate), and traffic between Caddy and xray flows over localhost without encryption — so TLS is **deliberately disabled in the 3x-ui inbound**.
>
> As a consequence, the link the panel generates for clients (`vless://...?security=none&type=ws&...`) **will not work as-is**. The client sees `security=none` and won't expect TLS from the server — but the server (Caddy) is serving TLS.
>
> **Fix:** in the client link (or subscription template), manually change `security=none` → `security=tls`. After that the connection works.
>
> This step must be done **every time** you export a link from the panel. Alternatively, set up a subscription service that rewrites the parameter automatically.

---

## Quick Start

Full step-by-step instructions: **[INSTALL_EN.md](INSTALL_EN.md)**

```
Step 1 — environment setup and node preparation:
         clone repo, configure vars, copy x-ui.db → site-init.yml
Step 2 — node configuration:
         site-configure.yml
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
- Install 3x-ui + upload `x-ui.db` (automatically: `webListen=127.0.0.1`, `webBasePath=xui_panel_path`, certificate paths cleared)
- Install Caddy + deploy Caddyfile (reverse-proxy to x-ui panel + WebSocket to xray + fallback to camouflage site)
- Final reboot → Caddy obtains a TLS certificate via Let's Encrypt → start x-ui

> **After a successful deployment**, `deploy_user` is no longer needed. It is recommended to delete it manually:
> ```bash
> sudo userdel -r <deploy_user>
> sudo rm -f /etc/sudoers.d/<deploy_user>
> ```

### Traffic flow

```
Client → 443 (Caddy, TLS)
           ├─ path xui_panel_path/* → localhost:xui_panel_port (x-ui panel)
           ├─ path xray_ws_path/*   → localhost:xray_port (xray, WebSocket)
           └─ everything else       → caddy_fallback_url (redirect, camouflage)

Let's Encrypt → HTTP-01 (port 80) → Caddy obtains and auto-renews the certificate
```

The x-ui panel runs on localhost over plain HTTP; TLS is terminated at Caddy. The certificate is renewed automatically every ~60 days — no sync scripts or service restarts required.

---

## Project Structure

```
ansible-vps/
├── site-init.yml                    # Stage 1 (port 22, root)
├── site-configure.yml               # Stage 2 (custom port, deploy_user)
├── inventory.ini.example            # Inventory template
├── ansible.cfg
├── group_vars/
│   ├── new_vps.yml.example          # Shared group variables template (committed)
│   └── new_vps.yml                  # Shared variables (in .gitignore — not committed!)
├── host_vars/                       # Per-host variables (for multi-server)
│   ├── server1.yml.example          # Per-host template (committed)
│   └── server1.yml                  # Your server1 variables (in .gitignore!)
├── roles/bootstrap/
│   ├── tasks/
│   │   ├── main.yml                 # Phase dispatcher
│   │   ├── upgrade.yml              # apt full-upgrade
│   │   ├── init.yml                 # Users, SSH, firewall
│   │   └── configure.yml            # 3x-ui, Caddy
│   ├── handlers/main.yml
│   ├── templates/
│   │   ├── sshd_config.j2
│   │   ├── iptables_v4.j2
│   │   ├── iptables_v6.j2
│   │   └── Caddyfile.j2
│   └── files/                       # Files from the reference server (in .gitignore!)
│       └── x-ui.db                  # the only file taken from the reference server
├── INSTALL.md                       # Step-by-step instructions
└── CHANGELOG.md
```

> ⚠️ **The name `new_vps` is used in three places and must match:**
> 1. `inventory.ini` → `[new_vps]` group
> 2. `group_vars/new_vps.yml` → filename
> 3. `site-init.yml` / `site-configure.yml` → `hosts: new_vps`
>
> If you want to rename the group (e.g. to `vpn_nodes`), it must be changed in **all three places** — otherwise Ansible won't pick up the variables or the playbook won't find the hosts.

---

## Multi-Server Deployment

The playbook supports deploying to multiple servers simultaneously. Ansible runs hosts in parallel by default (5 forks, tunable via `forks` in `ansible.cfg` or the `-f N` flag).

### Splitting variables

```
group_vars/new_vps.yml   → shared: users, ssh_port, acme_email, extra_packages, sysctl, ...
host_vars/server1.yml    → per-host for server1: hostname, (optional) paths, ports
host_vars/server2.yml    → per-host for server2
```

**`hostname` must be unique per server** — Caddy obtains a separate TLS certificate per FQDN via Let's Encrypt.

### Example for 3 servers

`inventory.ini`:
```ini
[new_vps]
server1 ansible_host=1.2.3.4
server2 ansible_host=5.6.7.8
server3 ansible_host=9.10.11.12

[new_vps:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

`host_vars/server1.yml`:
```yaml
hostname: "vps1.example.com"
```

`host_vars/server2.yml`:
```yaml
hostname: "vps2.example.com"
```

`host_vars/server3.yml`:
```yaml
hostname: "vps3.example.com"
```

> ⚠️ **The filename in `host_vars/` must match the inventory host name** (left column, not IP). So for `server1 ansible_host=1.2.3.4` → `host_vars/server1.yml`.

### Preconditions

- The same SSH key uploaded to every server beforehand (`ssh-copy-id root@<IP>` for each IP)
- For each FQDN, a DNS A-record pointing to the corresponding server IP

> **About `x-ui.db`:** the project always has exactly one `roles/bootstrap/files/x-ui.db`, and it gets rolled out to every server in the group — this is by design, a farm receives identical inbounds. If you need different inbounds per server, configure them manually in the panel after deployment (or split the servers into different groups with separate DB files).

### When servers are genuinely different

Create separate groups:

```ini
[node_fr]
fr1 ansible_host=1.2.3.4

[node_de]
de1 ansible_host=5.6.7.8

[new_vps:children]
node_fr
node_de
```

And split variables: `group_vars/node_fr.yml`, `group_vars/node_de.yml` for group-specific settings, `group_vars/new_vps.yml` for everything shared.

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

Caddy obtains and auto-renews the TLS certificate via Let's Encrypt (HTTP-01 challenge on port 80). The cert is stored in Caddy's storage; x-ui itself runs on `127.0.0.1` over plain HTTP — panel access goes through Caddy at a secret URL.

```yaml
acme_email: "admin@example.com"   # email for Let's Encrypt
```

> Ansible automatically clears `webCertFile`/`webKeyFile` in `x-ui.db` and
> sets `webListen=127.0.0.1`, `webBasePath=xui_panel_path` — no changes
> needed on the reference server.

> **Inbound TLS certificates** — if any 3X-UI inbounds use TLS certificates (separate from the panel
> certificate), they are **not copied automatically**. After deployment you must manually upload them
> to the server and verify the paths and file permissions in the inbound settings.

### Caddy

Caddy listens on port 443, obtains a TLS certificate via ACME, and manages traffic:

- **x-ui panel** is accessible at `xui_panel_path` (reverse-proxied to `localhost:xui_panel_port`)
- **VPN traffic** matching `xray_ws_path` is proxied to xray via WebSocket
- **Other traffic** is redirected to `caddy_fallback_url` (camouflage site)

| Variable | Example | Description |
|---|---|---|
| `acme_email` | `admin@example.com` | Email for Let's Encrypt (required) |
| `caddy_fallback_url` | `https://example.com` | Camouflage site for non-VPN traffic |
| `xui_panel_path` | `/my-secret-panel` | Secret path to the x-ui panel (starts with `/`) |
| `xui_panel_port` | `54321` | x-ui panel port on localhost |
| `xray_ws_path` | `/your-secret-path` | Secret WebSocket path for xray (starts with `/`) |
| `xray_port` | `10000` | xray port on localhost (WebSocket) |

> **debug in Caddyfile** — commented out in the template. Uncomment for debugging,
> disable in production — affects performance and log volume.

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

> **The following packages are installed automatically** — no need to add them to `extra_packages`:
> `iptables`, `iptables-persistent`, `ipset`, `ipset-persistent`, `netfilter-persistent`, `sqlite3`

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
