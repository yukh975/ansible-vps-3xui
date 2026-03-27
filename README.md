# ansible-vps

Ansible playbook для быстрого развёртывания VPS-ноды с 3x-ui и Caddy.

Берёт свежеустановленный Debian 12/13 с root-доступом по SSH и за два этапа превращает его в готовую к работе ноду: hardened SSH, firewall, пользователи, [3x-ui](https://github.com/MHSanaei/3x-ui) с базой данных от эталонного сервера, Caddy как reverse-proxy с автоматическим TLS-сертификатом через Let's Encrypt.

---

## Как это работает

Playbook разделён на два этапа, потому что первый этап меняет SSH-порт и закрывает root — после чего подключиться по-старому уже нельзя.

### Этап 1 — `site-init.yml` (порт 22, root)

Подключается к новому серверу стандартным способом и выполняет:

- `apt full-upgrade` + перезагрузка
- Установка пакетов (настраиваемый список)
- Создание пользователей из конфига (с SSH-ключами, паролями, опциональным NOPASSWD sudo)
- Установка hostname
- SSH hardening — кастомный порт, root закрыт, только ключи
- Применение sysctl, iptables, ipset
- Перезагрузка → SSH поднимается на новом порту

### Этап 2 — `site-configure.yml` (кастомный порт, deploy_user)

Подключается уже по новому порту от имени deploy_user:

- Установка 3x-ui + загрузка x-ui.db с эталонного сервера
- Установка Caddy + деплой Caddyfile
- Деплой cert-sync (systemd ExecStartPost: автокопирование сертификатов из Caddy в `/etc/ssl/<домен>/`)
- Финальная перезагрузка — Caddy стартует и получает TLS-сертификат через ACME (HTTP-01)
- Проверка статуса x-ui и caddy

### Управление сертификатами

После перезагрузки Caddy автоматически получает TLS-сертификат через Let's Encrypt (HTTP-01 challenge, порт 80). Скрипт `cert-sync.sh` запускается при каждом старте Caddy через `ExecStartPost` и копирует сертификат из хранилища Caddy в `/etc/ssl/<домен>/`, затем перезапускает x-ui — без ручного вмешательства.

При автообновлении сертификата (раз в ~60 дней) Caddy перезапускает себя → cert-sync срабатывает автоматически.

---

## Структура проекта

```
ansible-vps/
├── site-init.yml                         # Этап 1 (порт 22, root)
├── site-configure.yml                    # Этап 2 (кастомный порт, deploy_user)
├── inventory.ini.example                 # Шаблон inventory (один на оба этапа)
├── ansible.cfg
├── group_vars/
│   ├── new_vps.yml.example               # Шаблон переменных (коммитится)
│   └── new_vps.yml                       # Ваши переменные (в .gitignore)
├── roles/bootstrap/
│   ├── tasks/
│   │   ├── main.yml                      # Диспетчер: выбирает фазу
│   │   ├── upgrade.yml                   # apt full-upgrade + reboot
│   │   ├── init.yml                      # Пользователи, SSH, firewall
│   │   └── configure.yml                 # 3x-ui, Caddy, cert-sync
│   ├── handlers/main.yml                 # restart sshd, caddy, iptables
│   ├── templates/
│   │   ├── sshd_config.j2               # Шаблон sshd_config
│   │   ├── iptables_v4.j2               # Шаблон правил IPv4
│   │   ├── iptables_v6.j2               # Шаблон правил IPv6
│   │   ├── Caddyfile.j2                 # Шаблон Caddyfile
│   │   ├── cert-sync.sh.j2              # Скрипт синхронизации сертификатов
│   │   └── caddy-override.conf.j2       # systemd drop-in для cert-sync
│   └── files/                           # Файлы с эталона (в .gitignore)
│       ├── x-ui.db
│       ├── <user>/authorized_keys       # SSH-ключи пользователей
│       └── <user>/.bashrc              # Настройки оболочки
├── INSTALL.md                           # Пошаговая инструкция
└── CHANGELOG.md
```

---

## Конфигурация

Все настройки хранятся в `group_vars/new_vps.yml`. Шаблон: `group_vars/new_vps.yml.example`.

### Основные переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `hostname` | `vps1.example.com` | FQDN сервера |
| `domain` | `example.com` | Домен для ACME-сертификата |
| `ssh_port` | `275` | SSH-порт после hardening |
| `deploy_user` | `yukh` | Пользователь для этапа 2 (должен быть в `users`) |

### Пользователи

Единый словарь `users`. Каждый пользователь (кроме root) создаётся автоматически.

```yaml
users:
  myuser:
    groups: ["sudo"]
    shell: /bin/bash
    password: "$6$..."           # openssl passwd -6 'ПАРОЛЬ'
    sudo_nopasswd: true          # NOPASSWD sudo (default: false)
    has_authorized_keys: true    # копировать files/myuser/authorized_keys (default: false)
    has_bashrc: true             # копировать files/myuser/.bashrc (default: false)
  root:
    password: "$6$..."
    has_bashrc: true
```

### Сертификаты (ACME)

```yaml
acme_email: "admin@example.com"     # email для регистрации в Let's Encrypt
certs_dest_dir: "/etc/ssl/{{ domain }}"  # куда cert-sync копирует сертификаты для x-ui
```

> **Важно:** путь `certs_dest_dir` **должен совпадать** с настройками путей в `x-ui.db` на эталонном сервере (раздел TLS/SSL в настройках инбаунда). Если меняете путь — обновите и в 3X-UI на эталоне.

Сертификаты получаются автоматически при первом старте Caddy. Перезапуск x-ui происходит автоматически после получения или обновления сертификата.

### Caddy

| Переменная | По умолчанию | Описание |
|---|---|---|
| `install_caddy` | `true` | Установка Caddy |
| `caddy_redirect_url` | `127.0.0.1:2053` | Куда Caddy проксирует HTTPS-запросы |
| `caddy_https_port` | `8443` | HTTPS-порт Caddy (не 443, чтобы не конфликтовать с 3x-ui) |
| `acme_email` | `admin@example.com` | Email для Let's Encrypt |

> **Порты:** 3x-ui занимает порт 443 для VPN-инбаундов. Caddy использует порт 80 для ACME HTTP-01 challenge и `caddy_https_port` (8443 по умолчанию) для HTTPS. Оба порта должны быть открыты в `allowed_tcp_ports`.

### Опциональные компоненты

| Переменная | По умолчанию | Описание |
|---|---|---|
| `install_3xui` | `true` | Установка 3x-ui |
| `xui_version` | `""` (latest) | Версия 3x-ui (`""` = последняя) |
| `ipset_set_name` | `allowed_hosts` | Имя ipset-set'а |
| `ipset_hosts` | `[]` | Список IP — полный доступ через ipset |
| `allowed_tcp_ports` | `[80, 443, 8443]` | TCP-порты открытые для всех |
| `allowed_udp_ports` | `[]` | UDP-порты открытые для всех |
| `sysctl_settings` | см. конфиг | Параметры ядра (dict) |

---

## Быстрый старт

```bash
# 1. Настроить переменные
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
# Отредактировать group_vars/new_vps.yml

# 2. Создать inventory
cp inventory.ini.example inventory.ini
# Вписать IP сервера

# 3. Залить SSH-ключ на новый сервер
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP>

# 4. Этап 1
ansible-playbook -i inventory.ini site-init.yml

# 5. Этап 2
ansible-playbook -i inventory.ini site-configure.yml
```

Подробнее: [INSTALL.md](INSTALL.md)

---

## Важно: этап 1 — одноразовый

**Этап 1 (`site-init.yml`) запускается один раз** на свежем сервере с доступом root:22.
После его выполнения SSH переходит на кастомный порт, root закрывается — повторный запуск этапа 1 завершится ошибкой подключения.

Для обновления конфигурации на работающем сервере используйте **только этап 2**:
```bash
ansible-playbook -i inventory.ini site-configure.yml
```

---

## Безопасность

- `group_vars/new_vps.yml` содержит хэши паролей — **не коммитится** (в `.gitignore`)
- `roles/bootstrap/files/` содержит ключи и БД — **не коммитится**
- `inventory.ini` содержит реальные IP — **не коммитится**
- Рекомендуется использовать `ansible-vault encrypt group_vars/new_vps.yml`

---

## Зависимости

- **Ansible** 2.12+ (`brew install ansible` на macOS)
- **Управляющая машина:** Linux / macOS
- **Целевой сервер:** Debian 12 или 13, root по SSH на порту 22
- **DNS:** домен должен указывать на IP сервера до запуска этапа 2 (нужен для ACME)
