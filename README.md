# ansible-vps

Ansible playbook для быстрого развёртывания VPS-ноды с 3x-ui и Caddy.

Берёт свежеустановленный Debian 12/13 с root-доступом по SSH и за два этапа превращает его в готовую к работе ноду: hardened SSH, firewall, пользователи, [3x-ui](https://github.com/MHSanaei/3x-ui) с базой данных от эталонного сервера, Caddy как reverse-proxy с автоматическим TLS-сертификатом через Let's Encrypt.

---

## Быстрый старт

Подробная пошаговая инструкция: **[INSTALL.md](INSTALL.md)**

```
Шаг 1 — подготовка (один раз, локально на control node):
         клонировать репо, настроить конфиг, скопировать x-ui.db

Шаг 2 — для каждого нового сервера:
         site-init.yml → site-configure.yml
```

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
- Деплой `cert-sync.sh` (копирует сертификат Caddy в `certs_dest_dir` для панели x-ui)
- Финальная перезагрузка → Caddy получает TLS-сертификат через Let's Encrypt

> **После успешного развёртывания** `deploy_user` больше не нужен. Рекомендуется удалить его вручную:
> ```bash
> sudo userdel -r <deploy_user>
> sudo rm -f /etc/sudoers.d/<deploy_user>
> ```

### Схема трафика

```
Клиент → 443 (Caddy, TLS)
           ├─ путь xray_ws_path/* → localhost:xray_port (xray, WebSocket)
           └─ всё остальное      → caddy_fallback_url (редирект, камуфляж)

Let's Encrypt → HTTP-01 (порт 80) → Caddy получает сертификат
→ cert-sync.sh копирует в certs_dest_dir → панель x-ui перезапускается
```

При автообновлении сертификата (раз в ~60 дней) Caddy обновляет его сам → cert-sync срабатывает через systemd `ExecStartPost`.

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
│   │   ├── cert-sync.sh.j2          # Скрипт синхронизации сертов для панели x-ui
│   │   └── caddy-override.conf.j2   # systemd drop-in (ExecStartPost)
│   └── files/                       # Файлы с эталона (в .gitignore!)
│       └── x-ui.db                  # единственный файл с эталона
├── INSTALL.md                       # Пошаговая инструкция
└── CHANGELOG.md
```

---

## Переменные

Все настройки в `group_vars/new_vps.yml`. Шаблон: `group_vars/new_vps.yml.example`.

### Сервер и доступ

| Переменная | Пример | Описание |
|---|---|---|
| `hostname` | `vps1.example.com` | FQDN сервера (используется для сертификата и hostname) |
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
    ssh_public_keys:             # публичные ключи: cat ~/.ssh/id_ed25519.pub
      - "ssh-ed25519 AAAA... ansible"
      - "ssh-ed25519 AAAA... laptop"   # можно несколько
  root:
    password: "$6$..."
```

### Сертификаты

```yaml
acme_email: "admin@example.com"          # email для Let's Encrypt
certs_dest_dir: "/etc/ssl/{{ hostname }}"  # куда cert-sync кладёт сертификат для панели x-ui
```

> Ansible автоматически обновляет пути к сертификатам панели x-ui в `x-ui.db` после заливки —
> трогать эталонный сервер не нужно.

> **Сертификаты в инбаундах 3X-UI** — если в настройках инбаундов используются отдельные
> TLS-сертификаты, они **не копируются автоматически**. После развёртывания необходимо вручную
> загрузить их на сервер и проверить пути и права доступа в настройках инбаунда.

### Caddy

Caddy слушает порт 443, получает TLS-сертификат через ACME и управляет трафиком:

- **VPN-трафик** по пути `xray_ws_path` проксируется на xray через WebSocket
- **Остальной трафик** редиректится на `caddy_fallback_url` (сайт-камуфляж)
- **Сертификат** синхронизируется в `certs_dest_dir` для панели x-ui

| Переменная | Пример | Описание |
|---|---|---|
| `acme_email` | `admin@example.com` | Email для Let's Encrypt (обязательно) |
| `caddy_fallback_url` | `https://example.com` | Сайт-камуфляж для не-VPN трафика |
| `xray_ws_path` | `/your-secret-path` | Секретный путь WebSocket (начинается с `/`) |
| `xray_port` | `10000` | Порт xray на localhost (WebSocket) |

> **debug в Caddyfile** — в шаблоне закомментирован. Раскомментировать для отладки,
> отключить в продакшне — влияет на производительность и объём логов.

### Firewall

```yaml
allowed_tcp_ports:
  - 80    # HTTP (ACME HTTP-01 challenge)
  - 443   # HTTPS (Caddy + xray через WebSocket)

ipset_set_name: "allowed_hosts"   # имя ipset-сета
ipset_hosts:                       # полный доступ для этих IP
  - "1.2.3.4"
```

Дополнительные правила iptables добавляются напрямую в шаблоны:
```
roles/bootstrap/templates/iptables_v4.j2
roles/bootstrap/templates/iptables_v6.j2
```
Дефолтные политики (`INPUT DROP`, `FORWARD DROP`, `OUTPUT ACCEPT`), loopback, ESTABLISHED и ICMP уже прописаны.

### Прочее

| Переменная | По умолчанию | Описание |
|---|---|---|
| `install_3xui` | `true` | Установка 3x-ui |
| `xui_version` | `""` (latest) | Версия 3x-ui |
| `sysctl_settings` | см. конфиг | Параметры ядра |
| `extra_packages` | см. конфиг | Дополнительные пакеты |

> **Следующие пакеты устанавливаются автоматически** — не нужно добавлять их в `extra_packages`:
> `iptables`, `iptables-persistent`, `ipset`, `ipset-persistent`, `netfilter-persistent`, `sqlite3`

---

## Порты

```
22          → только этап 1 (root, временно)
ssh_port    → SSH после hardening
80          → Caddy (ACME challenge)
443         → Caddy (TLS): xray_ws_path/* → xray, остальное → fallback
```

---

## Безопасность

- `group_vars/new_vps.yml` — хэши паролей, **не коммитится** (`.gitignore`)
- `roles/bootstrap/files/` — БД x-ui — **не коммитится**
- `inventory.ini` — реальные IP — **не коммитится**
- Рекомендуется: `ansible-vault encrypt group_vars/new_vps.yml`

---

## Требования

- **Ansible** 2.12+ (`brew install ansible` на macOS)
- **Целевой сервер:** Debian 12 или 13, root по SSH на порту 22
- **DNS:** домен должен указывать на IP сервера **до запуска этапа 2** (нужен для ACME)
