# Changelog

## [0.8.0] — 2026-03-27

### Testing on a clean VM, critical bug fixes

### Fixed

- **`configure.yml`** — removed `test_command: systemctl is-active caddy` from the final reboot task. When `install_caddy: false` or ACME was slow, the command always failed and Ansible would hang for 5 minutes until timeout.
- **`init.yml`** — applying iptables rules, saving netfilter-persistent, and rebooting are now combined into a single `async: 30, poll: 0` shell script. Previously, applying the rules would drop the SSH connection before Ansible could execute the subsequent tasks.
- **`init.yml`** — removal of root's `authorized_keys` moved into the same final async script; otherwise the task would run after the connection was dropped and fail with an error.
- **`init.yml`** — removed `{{ item.key }}` from task names (caused `'item' is undefined` warnings). The username is now shown via `loop_control.label`.
- **`init.yml`** — `.bashrc` copying is skipped if the file does not exist (`lookup('fileglob', ...)`), instead of raising an error.
- **`ansible.cfg`** — added `bin_ansible_callbacks = True` for correct operation of the `community.general.counter_enabled` task counter.
- **`group_vars/new_vps.yml.example`** — removed spurious `=` sign in BBR values (`net.core.default_qdisc: = fq` → `net.core.default_qdisc: fq`).

### Added

- **`configure.yml`** — task `[3xui] Update certificate paths in DB`: after uploading `x-ui.db`, automatically updates `webCertFile` and `webKeyFile` via `sqlite3` using the `certs_dest_dir` and `domain` variables. Eliminates the need to manually update paths in x-ui.db on the reference server when changing the domain.
- **`configure.yml`** — a recommendation to remove `deploy_user` after setup is complete is displayed in the final message.
- **`INSTALL.md`** — step 2.7 "Remove the technical user" with the `userdel -r` command.
- **`README.md`** — note about removing `deploy_user` after stage 2.
- **`init.yml`** — loading the `tcp_bbr` module (`modprobe`) and persistent activation via `/etc/modules-load.d/tcp_bbr.conf`.
- **`templates/sysctl.conf.j2`** — template for `/etc/sysctl.d/99-custom.conf`, replaces the `ansible.posix.sysctl` module.
- **`.gitkeep`** — placeholder file in `roles/bootstrap/files/` so the directory exists after `git clone`.

### Removed

- **`requirements.yml`** — removed along with the dependency on the `ansible.posix` collection. All modules replaced with `ansible.builtin`: `authorized_key` → `copy`, `sysctl` → `template + command`.

---

## [0.7.0] — 2026-03-27

### INSTALL.md — complete rewrite

### Changed

- **INSTALL.md** — rewritten from scratch: added `git clone` step, full parameter table split into required/optional, instructions for SSH keys (multiple keys per user), password hash, optional `.bashrc`, and vault encryption.
- **`group_vars/new_vps.yml.example`** — removed hardcoded usernames, replaced with a neutral `myuser`. Removed the second test user.
- **`CHANGELOG.md`** — removed mentions of specific names and domains.

---

## [0.6.0] — 2026-03-27

### SSH keys in config, removed .bashrc support

Eliminated the last dependency on user-specific files. Now only `x-ui.db` is needed to deploy a new server.

### Changed

- **SSH authorized_keys** — instead of copying the `files/<user>/authorized_keys` file, keys are now defined as a list `ssh_public_keys` directly in `group_vars/new_vps.yml`. Uses the `authorized_key` module with `subelements`.
- **`.bashrc`** — support for copying `.bashrc` for users and root has been removed (the system default is sufficient for Ansible-managed servers).
- **`group_vars/new_vps.yml`** — removed `has_authorized_keys` and `has_bashrc` flags.
- **`INSTALL.md`** — simplified Part A: instead of `scp authorized_keys` + `mkdir`, just one `cat ~/.ssh/id_ed25519.pub` command and paste into config.

### Removed

- Tasks `[users] Create ~/.ssh`, `[users] authorized_keys` (file copy) from `init.yml`
- Tasks `[users] .bashrc` and `[users] .bashrc for root` from `init.yml`
- The need to create `roles/bootstrap/files/<user>/` directories

### Summary: files from the reference server

The only file that needs to be copied from the reference server: **`x-ui.db`**.

---

## [0.5.0] — 2026-03-27

### Documentation overhaul

README.md and INSTALL.md completely rewritten for clarity and ease of use.

### Changed

- **README.md** — added Quick Start diagram, port table, description of automatic certificate management. Removed outdated references to manual certificate copying.
- **INSTALL.md** — A/B structure (once / per server). Step-by-step instructions B4/B5 with a clear description of what happens at each step. Added "Troubleshooting" section.

---

## [0.4.0] — 2026-03-27

### Certificate management via Caddy ACME

Manual copying of TLS certificates from the reference server replaced with automatic provisioning via Let's Encrypt.

### Changed

- **Certificates** — removed the copy block from `init.yml`. Certificates are now obtained automatically via Caddy ACME (HTTP-01 challenge).
- **Caddyfile** — added: `email {{ acme_email }}`, `https_port {{ caddy_https_port }}`. Caddy listens on a non-standard HTTPS port (default: 8443) to avoid conflicting with 3x-ui on 443. HTTP-01 challenge works via port 80.
- **iptables** — default ports: `80` (ACME + HTTP), `443` (3x-ui), `8443` (Caddy HTTPS).
- **Documentation** — removed `scp` commands for certificates, added description of ACME and cert-sync.

### Added

- `templates/cert-sync.sh.j2` — certificate sync script from Caddy to `certs_dest_dir`. Waits for the certificate to be issued, copies only if changed, restarts x-ui.
- `templates/caddy-override.conf.j2` — systemd drop-in for Caddy: `ExecStartPost=/usr/local/bin/cert-sync.sh`. Ensures synchronization both on first start and on auto-renewal.
- Variables: `acme_email`, `certs_dest_dir`, `caddy_https_port`.

### Removed

- The `[certs]` block from `init.yml` — no longer needed (Caddy issues the certificates).
- The need to copy certificates from the reference server (`scp .crt/.key`).
- The `certs` variable (dict with `base_dir` and `files`) — replaced by `certs_dest_dir`.

---

## [0.3.0] — 2026-03-27

### Parameterization of sysctl, iptables, Caddyfile

Eliminated all dependencies on files from the reference server for system configs.

### Changed

- **sysctl** — instead of copying a `sysctl.conf` file, a `sysctl_settings` dictionary is used in the config. Ansible applies each parameter via the `sysctl` module (idempotently).
- **iptables** — rules are generated from Jinja2 templates `iptables_v4.j2` / `iptables_v6.j2`. SSH port and ipset are substituted automatically. Configured via `allowed_tcp_ports` / `allowed_udp_ports`.
- **Caddyfile** — generated from the `Caddyfile.j2` template. The proxy URL is set via `caddy_redirect_url` (default: `127.0.0.1:2053`). TLS paths are taken from `certs.base_dir`.
- **ipset in iptables** — addresses from `ipset_hosts` receive full access (`-j ACCEPT` on all protocols).

### Added

- `templates/iptables_v4.j2` — IPv4 rules template with ipset support
- `templates/iptables_v6.j2` — IPv6 rules template
- `templates/Caddyfile.j2` — Caddy config template
- Variables: `sysctl_settings`, `allowed_tcp_ports`, `allowed_udp_ports`, `caddy_redirect_url`

### Removed

- The need to copy from the reference server: `sysctl.conf`, `rules.v4`, `rules.v6`, `Caddyfile`
- Variables: `sysctl_src`, `iptables_rules_v4_src`, `iptables_rules_v6_src`

---

## [0.2.1] — 2026-03-27

### Fixed

- **Documentation:** removed the erroneous statement "Playbook is idempotent". Stage 1 (`site-init.yml`) is intended **only for initial deployment** — once the SSH port is changed and root is locked, re-running is not possible. Stage 2 (`site-configure.yml`) is idempotent. Corresponding warnings added to `README.md` and `INSTALL.md`.

---

## [0.2.0] — 2026-03-27

### Project generalization

Full parameterization — all hardcoded usernames, domain names, and paths removed.

### Changed

- **Users** — single `users` dictionary with flags (`sudo_nopasswd`, `has_authorized_keys`, `has_bashrc`). Tasks iterate over the dictionary instead of having separate blocks for each user.
- **Variable `deploy_user`** — username for stage 2 (previously hardcoded).
- **Variable `domain`** — domain for certificate paths (previously hardcoded).
- **Certificates** moved from `/home/ca/certs/<domain>/` to `/etc/ssl/<domain>/`. Path configured via `certs.base_dir`. Owner is root.
- **3x-ui and Caddy** — optional via `install_3xui` and `install_caddy` flags (default `true`).
- **`ansible.cfg`** — `interpreter_python` changed from `/usr/bin/python3.13` to `auto_silent`.
- **`site-init.yml` / `site-configure.yml`** — hardcoded values removed from comments.
- **Inventory** consolidated into a single `inventory.ini` file instead of two (`inventory-init.ini` + `inventory-configure.ini`). Port and user for each stage are now set in the playbooks themselves.
- **ipset** — instead of copying an `ipset.conf` file from the reference server, IP addresses are defined as a list `ipset_hosts` in the config. The set name is set via `ipset_set_name` (default `allowed_hosts`). Ansible creates the set and adds addresses automatically.

### Added

- `inventory.ini.example` — single inventory template.
- `group_vars/new_vps.yml.example` — variables template (committed to git).
- `CHANGELOG.md` — this file.

### Removed

- `inventory-init.ini.example` and `inventory-configure.ini.example` — replaced by a single `inventory.ini.example`.

### Security

- `.gitignore` expanded: `roles/bootstrap/files/`, `group_vars/new_vps.yml`, `inventory-*.ini` will no longer be included in commits.

---

## [0.1.0] — 2026-03-27

### Initial release

- Two-stage VPS deployment (init + configure).
- Package installation, user creation, SSH hardening, sysctl, iptables, ipset.
- 3x-ui installation with DB upload, Caddy with Caddyfile.
- sshd_config template via Jinja2.
