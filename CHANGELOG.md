# Changelog

Все значимые изменения проекта документируются в этом файле.

Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.1.0/).

---

## [0.3.0] — 2026-03-27

### Параметризация sysctl, iptables, Caddyfile

Полностью убрана зависимость от файлов с эталонного сервера для системных конфигов.

### Изменено

- **sysctl** — вместо копирования файла `sysctl.conf` используется словарь `sysctl_settings` в конфиге. Ansible применяет каждый параметр через модуль `sysctl` (идемпотентно).
- **iptables** — правила генерируются из Jinja2-шаблонов `iptables_v4.j2` / `iptables_v6.j2`. SSH-порт и ipset-сет подставляются автоматически. Настраиваются через `allowed_tcp_ports` / `allowed_udp_ports`.
- **Caddyfile** — генерируется из шаблона `Caddyfile.j2`. URL проксирования задаётся через `caddy_redirect_url` (default: `127.0.0.1:2053`). TLS-пути берутся из `certs.base_dir`.
- **ipset в iptables** — адреса из `ipset_hosts` получают полный доступ (`-j ACCEPT` на все протоколы).

### Добавлено

- `templates/iptables_v4.j2` — шаблон правил IPv4 с поддержкой ipset
- `templates/iptables_v6.j2` — шаблон правил IPv6
- `templates/Caddyfile.j2` — шаблон конфига Caddy
- Переменные: `sysctl_settings`, `allowed_tcp_ports`, `allowed_udp_ports`, `caddy_redirect_url`

### Удалено

- Необходимость копировать с эталона: `sysctl.conf`, `rules.v4`, `rules.v6`, `Caddyfile`
- Переменные: `sysctl_src`, `iptables_rules_v4_src`, `iptables_rules_v6_src`

---

## [0.2.1] — 2026-03-27

### Исправлено

- **Документация:** убрано ошибочное утверждение «Playbook идемпотентен». Этап 1 (`site-init.yml`) предназначен **только для первичного развёртывания** — после смены SSH-порта и закрытия root повторный запуск невозможен. Этап 2 (`site-configure.yml`) идемпотентен. В `README.md` и `INSTALL.md` добавлены соответствующие предупреждения.

---

## [0.2.0] — 2026-03-27

### Универсализация проекта

Полная параметризация — убраны все захардкоженные имена пользователей, домен и пути.

### Изменено

- **Пользователи** — единый словарь `users` с флагами (`sudo_nopasswd`, `has_authorized_keys`, `has_bashrc`). Таски итерируют по словарю вместо отдельных блоков для каждого пользователя.
- **Переменная `deploy_user`** — имя пользователя для этапа 2 (ранее захардкожено `yukh`).
- **Переменная `domain`** — домен для путей к сертификатам (ранее захардкожено `netadm.pro`).
- **Сертификаты** перенесены из `/home/ca/certs/<домен>/` в `/etc/ssl/<домен>/`. Путь настраивается через `certs.base_dir`. Владелец — root.
- **3x-ui и Caddy** — опциональные через флаги `install_3xui` и `install_caddy` (по умолчанию `true`).
- **`ansible.cfg`** — `interpreter_python` изменён с `/usr/bin/python3.13` на `auto_silent`.
- **`site-init.yml` / `site-configure.yml`** — убраны хардкоды из комментариев.
- **Inventory** объединён в один файл `inventory.ini` вместо двух (`inventory-init.ini` + `inventory-configure.ini`). Порт и пользователь для каждого этапа теперь задаются в самих playbook.
- **ipset** — вместо копирования файла `ipset.conf` с эталонного сервера, IP-адреса задаются списком `ipset_hosts` в конфиге. Имя set'а задаётся через `ipset_set_name` (по умолчанию `allowed_hosts`). Ansible создаёт set и добавляет адреса автоматически.

### Добавлено

- `inventory.ini.example` — единый шаблон inventory.
- `group_vars/new_vps.yml.example` — шаблон переменных (коммитится в git).
- `CHANGELOG.md` — этот файл.

### Удалено

- `inventory-init.ini.example` и `inventory-configure.ini.example` — заменены единым `inventory.ini.example`.

### Безопасность

- `.gitignore` расширен: `roles/bootstrap/files/`, `group_vars/new_vps.yml`, `inventory-*.ini` больше не попадут в коммит.

---

## [0.1.0] — 2026-03-27

### Первая версия

- Двухэтапное развёртывание VPS (init + configure).
- Установка пакетов, создание пользователей, SSH hardening, sysctl, iptables, ipset.
- Установка 3x-ui с загрузкой БД, Caddy с Caddyfile.
- Шаблон sshd_config через Jinja2.
