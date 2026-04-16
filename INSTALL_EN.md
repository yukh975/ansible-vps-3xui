# Deployment Instructions

## Overview

```
Step 1 (once)         — clone the repo, install Ansible, configure variables
Step 2 (each server)  — upload SSH key, create inventory, run 2 playbooks
```

**Two playbooks — two stages:**
- `site-init.yml` — port 22, root. Configures the OS, users, SSH, firewall.
  After completion: root is locked, SSH moves to the custom port. **Run once per server.**
- `site-configure.yml` — custom port, your user. Installs 3x-ui and Caddy.
  **Idempotent** — can be re-run to update the configuration.

---

## Step 1. Initial Setup (once)

### 1.1. Clone the repository

```bash
git clone https://github.com/yukh975/ansible-vps-3xui.git ansible-vps
cd ansible-vps
```

---

### 1.2. Install Ansible

**macOS:**
```bash
brew install ansible
```

**Debian/Ubuntu:**
```bash
sudo apt install ansible
```

Verify:
```bash
ansible --version
```

---

### 1.3. Generate an SSH key (if you don't have one)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ansible"
```

Print the public key — you'll need it in the config:
```bash
cat ~/.ssh/id_ed25519.pub
```

---

### 1.4. Copy x-ui.db from the reference server

This is the **only file** that comes from the reference server.
It contains inbounds and connection settings.

```bash
scp root@<REFERENCE_IP>:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db
```

> **About certificate paths:**
> The x-ui panel is accessed via a secret URL in Caddy — TLS is handled by Caddy,
> the panel runs on localhost over plain HTTP. The `webCertFile` and `webKeyFile`
> fields are cleared automatically in the DB. `webListen` is set to `127.0.0.1`.
>
> **⚠️ Certificate paths in inbounds:**
> If inbounds on the reference server have explicit certificate paths configured —
> after deployment open the x-ui panel and update those paths manually.

---

### 1.5. Create the variables file

```bash
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
nano group_vars/new_vps.yml
```

> 💡 **Deploying multiple servers at once?**
> `group_vars/new_vps.yml` holds the **shared** variables for every host in the group (users, ssh_port, packages, sysctl). Per-host values (at minimum `hostname` — for the ACME certificate) go into `host_vars/<host>.yml`. See the "Multi-Server Deployment" section in [README_EN.md](README_EN.md) and the `host_vars/server1.yml.example` template.

#### Required parameters

| Parameter | Description |
|---|---|
| `hostname` | Server FQDN, e.g. `vps1.example.com` — used as hostname and for the ACME certificate |
| `ssh_port` | SSH port after hardening (not 22, e.g. `275`) |
| `deploy_user` | Username for stage 2 — **must be listed in `users`** |
| `acme_email` | Email for Let's Encrypt registration |
| `users.<name>.password` | SHA-512 password hash |
| `users.<name>.ssh_public_keys` | List of public SSH keys (at least one) |
| `caddy_fallback_url` | Camouflage site — where Caddy redirects regular traffic |
| `xui_panel_path` | Secret path for the x-ui panel, e.g. `/my-secret-panel` |
| `xray_ws_path` | Secret WebSocket path for xray, e.g. `/my-secret-ws` |

#### User parameters (optional)

| Parameter | Default | Description |
|---|---|---|
| `users.<name>.groups` | `[]` | Linux groups, e.g. `["sudo"]` |
| `users.<name>.sudo_nopasswd` | `false` | NOPASSWD in sudoers |
| `users.<name>.has_bashrc` | `false` | Copy `.bashrc` from `files/<name>/.bashrc` |

#### Services (optional)

| Parameter | Default | Description |
|---|---|---|
| `xui_panel_port` | `54321` | x-ui panel port (localhost) |
| `xray_port` | `10000` | xray WebSocket port (localhost) |
| `install_3xui` | `true` | Install 3x-ui |
| `xui_version` | `""` (latest) | 3x-ui version, e.g. `2.3.11` |

#### Firewall (optional)

| Parameter | Default | Description |
|---|---|---|
| `ipset_hosts` | `[]` | Trusted IPs with full access |
| `ipset_set_name` | `allowed_hosts` | ipset name |
| `allowed_tcp_ports` | `[80, 443]` | Open TCP ports |
| `allowed_udp_ports` | `[]` | Open UDP ports |

---

#### How to fill in the values

**SSH keys** — paste directly into the config, multiple keys supported:
```bash
cat ~/.ssh/id_ed25519.pub
```
```yaml
users:
  myuser:
    ssh_public_keys:
      - "ssh-ed25519 AAAA...ansible_key... ansible"
      - "ssh-ed25519 AAAA...personal_key... personal-laptop"   # optional
```

**Password hash** — generate locally:
```bash
openssl passwd -6 'YOUR_PASSWORD'
```

**`.bashrc`** — if you need aliases and a custom prompt:
```bash
mkdir -p roles/bootstrap/files/myuser/
nano roles/bootstrap/files/myuser/.bashrc
```
Then in the config: `has_bashrc: true`.

---

### 1.6. (Optional) Configure iptables rules

Ports and ipset are set in the config (see step 1.5). If you need additional rules, edit the templates directly **before running the playbook**:

```
roles/bootstrap/templates/iptables_v4.j2   — IPv4 rules
roles/bootstrap/templates/iptables_v6.j2   — IPv6 rules
```

Already included: `INPUT DROP / FORWARD DROP / OUTPUT ACCEPT` policies, loopback, ESTABLISHED, ICMP, SSH port, ports from `allowed_tcp_ports` / `allowed_udp_ports`, and ipset.

Add your own rules between existing blocks, for example:

```
# Example: allow incoming WireGuard
-A INPUT -p udp --dport 51820 -j ACCEPT
```

---

### 1.7. (Optional) Encrypt the config with ansible-vault

```bash
ansible-vault encrypt group_vars/new_vps.yml
```

Add this flag on every playbook run:
```bash
ansible-playbook ... --ask-vault-pass
```

To make changes — edit without decrypting (opens in $EDITOR):
```bash
ansible-vault edit group_vars/new_vps.yml
```

---

> ✅ **Step 1 complete.** When deploying additional servers, start from Step 2.

---

## Step 2. Deploy a New Server

### 2.1. Make sure DNS is configured

DNS is required for obtaining a TLS certificate via Caddy ACME (HTTP-01 challenge — Let's Encrypt connects to port 80 of the domain). Everything else — SSH, users, firewall, 3x-ui — works without DNS.

**What exactly needs to be configured:** an **A record** in the domain's DNS zone pointing to the IP of the new server.

```bash
dig +short vps1.example.com
# should return the new server's IP, e.g.: 1.2.3.4
```

---

### 2.2. Upload the SSH key to the server

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP>
```

> The command will ask for the root password — use the one provided by the hosting provider.
> This is the **only time** it will be needed: after stage 1 the root password will be changed,
> password-based SSH login will be disabled, and access will be key-only.

---

### 2.3. Create the inventory

```bash
cp inventory.ini.example inventory.ini
nano inventory.ini   # replace YOUR_SERVER_IP with the actual server IP
```

One `inventory.ini` works for both stages.

> 💡 **Multiple servers at once?** Add more lines to the `[new_vps]` section and create `host_vars/<host>.yml` for each (with a unique `hostname`). Ansible deploys in parallel. Details in [README_EN.md](README_EN.md).

---

### 2.4. Run stage 1 (port 22, root)

```bash
ansible-playbook -i inventory.ini site-init.yml
# with vault:
ansible-playbook -i inventory.ini site-init.yml --ask-vault-pass
```

What happens:
1. `apt full-upgrade` (if there are updates — automatic reboot and continuation)
2. Wait for cloud-init and apt lock to clear (up to 30 minutes — some providers run updates right after provisioning)
3. Install packages from `extra_packages`
4. Create users, SSH keys from config, sudoers
5. Set hostname
6. SSH hardening: custom port, root locked, key-only auth
7. sysctl, ipset, iptables
8. Reboot → SSH comes up on `ssh_port`

⚠️ **After this step, stage 1 cannot be re-run** — SSH on port 22 is closed, root is locked.

---

### 2.5. Run stage 2 (ssh_port, deploy_user)

```bash
ansible-playbook -i inventory.ini site-configure.yml
# with vault:
ansible-playbook -i inventory.ini site-configure.yml --ask-vault-pass
```

What happens:
1. Install 3x-ui (if not already installed)
2. Upload `x-ui.db`, update settings in DB (webListen, webBasePath, clear certificate paths)
3. Install Caddy from the official repository
4. Deploy Caddyfile (Caddy on port 443: ACME + panel proxy + WebSocket → xray + fallback)
5. Final reboot → wait for ACME certificate → start x-ui
   The playbook completes only when Caddy has obtained the certificate and x-ui is active

This step is **idempotent** — it can be re-run to update the configuration.

---

### 2.6. Verify the result

```bash
# Connect to the server
ssh -p <ssh_port> <deploy_user>@<IP>

# Service status
sudo systemctl status x-ui caddy

# x-ui panel is available at:
# https://<hostname><xui_panel_path>
```

---

### 2.7. Remove the technical user (recommended)

`deploy_user` is a technical user needed only by Ansible for stage 2. After a successful deployment it is no longer needed.

Connect as your main user (or root) and remove it:

```bash
sudo userdel -r <deploy_user>
sudo rm -f /etc/sudoers.d/<deploy_user>
```

> **Important:** make sure you have another user with SSH access before deleting `deploy_user`.

---

## Updating Configuration on a Running Server

Run stage 2 only — it is idempotent and connects via `ssh_port`:

```bash
ansible-playbook -i inventory.ini site-configure.yml
```

Use this to: update `x-ui.db`, change the Caddyfile, or change the 3x-ui version.

---

## Troubleshooting

**Stage 1 hung or failed before finishing:**
The server may have rebooted with the old SSH config. Try connecting on port 22 as root — if it works, stage 1 did not complete. Fix the issue and re-run.

**apt is blocked for a long time:**
This is normal — some hosting providers run updates right after provisioning.
The playbook waits up to 30 minutes (30 retries × 60 seconds). You'll see RETRYING messages in the output.

**Caddy did not obtain a certificate:**
```bash
sudo systemctl status caddy
sudo journalctl -u caddy -n 50
```
Make sure DNS is configured (`dig +short <hostname>` returns the server IP) and port 80 is open (`allowed_tcp_ports` contains `80`).

**x-ui panel is not accessible:**
Check that Caddy is running and `xui_panel_path` is correct:
```bash
sudo systemctl status caddy
curl -s http://localhost:<xui_panel_port>  # should respond
```
