# ansible-vps

Ansible playbook для быстрого развёртывания VPS-ноды с 3x-ui и Caddy.

Берёт свежеустановленный Debian 12/13 с root-доступом по SSH и за два этапа превращает его в готовую к работе ноду: hardened SSH, firewall, пользователи, [3x-ui](https://github.com/MHSanaei/3x-ui) с базой данных от эталонного сервера, Caddy как reverse-proxy.

---

## Как это работает

Playbook разделён на два этапа, потому что первый этап меняет SSH-порт и закрывает root — после чего подключиться по-старому уже нельзя.

### Этап 1 — `site-init.yml` (порт 22, root)

Подключается к новому серверу стандартным способом и выполняет:

- `apt full-upgrade` + перезагрузка
- Установка пакетов (настраиваемый список)
- Создание пользователей из конфига (с SSH-ключами, паролями, опциональным NOPASSWD sudo)
- Установка hostname
- Копирование TLS-сертификатов в `/etc/ssl/<домен>/`
- SSH hardening — кастомный порт, root закрыт, только ключи
- Применение sysctl, iptables, ipset
- Перезагрузка → SSH поднимается на новом порту

### Этап 2 — `site-configure.yml` (кастомный порт, deploy_user)

Подключается уже по новому порту от имени deploy_user:

- Установка 3x-ui + загрузка x-ui.db с эталонного сервера
- Установка Caddy + копирование Caddyfile
- Финальная перезагрузка и проверка сервисов

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
│   │   ├── init.yml                      # Пользователи, SSH, firewall, certs
│   │   └── configure.yml                 # 3x-ui, Caddy
│   ├── handlers/main.yml                 # restart sshd, caddy, iptables
│   ├── templates/sshd_config.j2          # Шаблон sshd_config
│   └── files/                            # Файлы с эталона (в .gitignore)
│       ├── sysctl.conf
│       ├── rules.v4 / rules.v6
│       ├── Caddyfile
│       ├── x-ui.db
│       ├── certs/                        # TLS-сертификаты
│       ├── <user>/authorized_keys        # SSH-ключи пользователей
│       └── <user>/.bashrc               # Оболочка пользователей
├── INSTALL.md                            # Пошаговая инструкция
└── CHANGELOG.md
```

---

## Конфигурация

Все настройки хранятся в `group_vars/new_vps.yml`. Шаблон: `group_vars/new_vps.yml.example`.

### Основные переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `hostname` | `vps1.example.com` | FQDN сервера |
| `domain` | `example.com` | Домен (используется в путях к сертификатам) |
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

### Сертификаты

```yaml
certs:
  base_dir: "/etc/ssl/{{ domain }}"    # путь на целевом сервере
  files:
    - { src: "fullchain.crt", dest: "fullchain.crt", mode: "0644" }
    - { src: "{{ domain }}.crt", dest: "{{ domain }}.crt", mode: "0644" }
    - { src: "{{ domain }}.key", dest: "{{ domain }}.key", mode: "0600" }
```

> **Важно:** пути к сертификатам на сервере **должны совпадать** с настройками в `x-ui.db` на эталоне. Если меняете `base_dir` — обновите и 3X-UI на эталонном сервере.

### Опциональные компоненты

| Переменная | По умолчанию | Описание |
|---|---|---|
| `install_3xui` | `true` | Установка 3x-ui |
| `install_caddy` | `true` | Установка Caddy |
| `xui_version` | `""` (latest) | Версия 3x-ui (`""` = последняя) |
| `ipset_set_name` | `allowed_hosts` | Имя ipset-set'а |
| `ipset_hosts` | `[]` | Список IP для ipset-set'а |

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

## Безопасность

- `group_vars/new_vps.yml` содержит хэши паролей — **не коммитится** (в `.gitignore`)
- `roles/bootstrap/files/` содержит ключи, сертификаты, БД — **не коммитится**
- `inventory.ini` содержит реальные IP — **не коммитится**
- Рекомендуется использовать `ansible-vault encrypt group_vars/new_vps.yml`

---

## Зависимости

- **Ansible** 2.12+ (`brew install ansible` на macOS)
- **Управляющая машина:** Linux / macOS
- **Целевой сервер:** Debian 12 или 13, root по SSH на порту 22
