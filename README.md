# ansible-vps

Ansible playbook для быстрого развёртывания VPS-ноды с 3x-ui и Caddy.

Берёт свежеустановленный Debian 12/13 с root-доступом по SSH и за два этапа превращает его в готовую к работе ноду: hardened SSH, firewall, пользователи, [3x-ui](https://github.com/MHSanaei/3x-ui) с базой данных от эталонного сервера, Caddy как reverse-proxy с автоматическим TLS-сертификатом через Let's Encrypt.

---

## Быстрый старт

```
Шаг 1: Один раз подготовить файлы (INSTALL.md, Часть A)
Шаг 2: Для каждого нового сервера → INSTALL.md, Часть B
```

Подробная пошаговая инструкция: **[INSTALL.md](INSTALL.md)**

---

## Как это работает

### Этап 1 — `site-init.yml` (подключение: порт 22, пользователь root)

Выполняется **один раз** на свежем сервере. После выполнения root закрыт, SSH переходит на кастомный порт — повторный запуск невозможен.

Что делает:
- `apt full-upgrade`
- Установка пакетов
- Создание пользователей (SSH-ключи, пароли, sudo)
- Установка hostname
- SSH hardening (кастомный порт, root закрыт, только ключи)
- sysctl, ipset, iptables
- Перезагрузка

### Этап 2 — `site-configure.yml` (подключение: `ssh_port`, `deploy_user`)

Можно запускать повторно. Идемпотентен.

Что делает:
- Установка 3x-ui + загрузка `x-ui.db`
- Установка Caddy + деплой Caddyfile
- Деплой `cert-sync.sh` (автосинхронизация сертификатов)
- Финальная перезагрузка → Caddy получает TLS-сертификат через Let's Encrypt

### Управление сертификатами (автоматически)

```
Let's Encrypt → HTTP-01 challenge (порт 80) → Caddy получает cert
→ cert-sync.sh копирует в /etc/ssl/<домен>/ → x-ui перезапускается
```

При автообновлении (раз в ~60 дней) всё происходит само: Caddy обновляет сертификат → cert-sync срабатывает через systemd `ExecStartPost`.

---

## Структура проекта

```
ansible-vps/
├── site-init.yml                    # Этап 1 (порт 22, root)
├── site-configure.yml               # Этап 2 (кастомный порт, deploy_user)
├── inventory.ini.example            # Шаблон inventory
├── ansible.cfg
├── group_vars/
│   ├── new_vps.yml.example          # Шаблон переменных (коммитится в git)
│   └── new_vps.yml                  # Ваши переменные (в .gitignore — не коммитится!)
├── roles/bootstrap/
│   ├── tasks/
│   │   ├── main.yml                 # Диспетчер фаз
│   │   ├── upgrade.yml              # apt full-upgrade
│   │   ├── init.yml                 # Пользователи, SSH, firewall
│   │   └── configure.yml            # 3x-ui, Caddy, cert-sync
│   ├── handlers/main.yml
│   ├── templates/
│   │   ├── sshd_config.j2
│   │   ├── iptables_v4.j2
│   │   ├── iptables_v6.j2
│   │   ├── Caddyfile.j2
│   │   ├── cert-sync.sh.j2          # Скрипт синхронизации сертов
│   │   └── caddy-override.conf.j2   # systemd drop-in (ExecStartPost)
│   └── files/                       # Файлы с эталона (в .gitignore!)
│       ├── x-ui.db
│       ├── <user>/authorized_keys
│       └── <user>/.bashrc
├── INSTALL.md                       # Пошаговая инструкция
└── CHANGELOG.md
```

---

## Переменные

Все настройки в `group_vars/new_vps.yml`. Шаблон: `group_vars/new_vps.yml.example`.

### Сервер и доступ

| Переменная | Пример | Описание |
|---|---|---|
| `hostname` | `vps1.example.com` | FQDN сервера |
| `domain` | `example.com` | Домен (нужен для ACME и сертификатов) |
| `ssh_port` | `275` | SSH-порт после hardening |
| `deploy_user` | `myuser` | Пользователь для этапа 2 (должен быть в `users`) |

### Пользователи

```yaml
users:
  myuser:
    groups: ["sudo"]
    shell: /bin/bash
    password: "$6$..."           # openssl passwd -6 'ПАРОЛЬ'
    sudo_nopasswd: true
    has_authorized_keys: true    # нужен файл files/myuser/authorized_keys
    has_bashrc: true             # нужен файл files/myuser/.bashrc
  root:
    password: "$6$..."
    has_bashrc: true             # нужен файл files/root/.bashrc
```

### Сертификаты

```yaml
acme_email: "admin@example.com"          # email для Let's Encrypt
certs_dest_dir: "/etc/ssl/{{ domain }}"  # куда cert-sync кладёт сертификаты
```

> Путь `certs_dest_dir` **должен совпадать** с настройками TLS в `x-ui.db`.
> Если меняете — обновите и в 3X-UI на эталонном сервере.

### Caddy

| Переменная | По умолчанию | Описание |
|---|---|---|
| `install_caddy` | `true` | Установка Caddy |
| `acme_email` | — | Email для Let's Encrypt (обязательно) |
| `caddy_redirect_url` | `127.0.0.1:2053` | Куда Caddy проксирует HTTPS |
| `caddy_https_port` | `8443` | HTTPS-порт Caddy (8443 чтобы не конфликтовать с 3x-ui на 443) |

### Firewall

```yaml
allowed_tcp_ports:
  - 80    # HTTP (ACME HTTP-01 challenge)
  - 443   # HTTPS (3x-ui инбаунды)
  - 8443  # HTTPS (Caddy panel)

ipset_set_name: "allowed_hosts"   # имя ipset-сета
ipset_hosts:                       # полный доступ для этих IP
  - "1.2.3.4"
```

### Прочее

| Переменная | По умолчанию | Описание |
|---|---|---|
| `install_3xui` | `true` | Установка 3x-ui |
| `xui_version` | `""` (latest) | Версия 3x-ui |
| `sysctl_settings` | см. конфиг | Параметры ядра |
| `extra_packages` | см. конфиг | Дополнительные пакеты |

---

## Порты

```
22   → только этап 1 (root, временно)
SSH  → ssh_port (275 по умолчанию)
80   → Caddy (ACME challenge)
443  → 3x-ui (VPN инбаунды)
8443 → Caddy (HTTPS reverse proxy)
```

---

## Безопасность

- `group_vars/new_vps.yml` — хэши паролей, **не коммитится** (`.gitignore`)
- `roles/bootstrap/files/` — ключи, БД — **не коммитится**
- `inventory.ini` — реальные IP — **не коммитится**
- Рекомендуется: `ansible-vault encrypt group_vars/new_vps.yml`

---

## Требования

- **Ansible** 2.12+ (`brew install ansible` на macOS)
- **Целевой сервер:** Debian 12 или 13, root по SSH на порту 22
- **DNS:** домен должен указывать на IP сервера **до запуска этапа 2** (нужен для ACME)
