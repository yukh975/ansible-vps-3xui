# ansible-vps

Ansible playbook для быстрого развёртывания VPS-ноды с 3x-ui и Caddy.

Берёт свежеустановленный Debian 12/13 с root-доступом по SSH и за два этапа превращает его в готовую к работе ноду: hardened SSH, firewall, пользователи, [3x-ui](https://github.com/MHSanaei/3x-ui) с базой данных от эталонного сервера, Caddy как reverse-proxy с автоматическим TLS-сертификатом через Let's Encrypt.

> ⚠️ **Сценарий заточен под xray VLESS + WebSocket.**
> Caddy на порту 443 проксирует WebSocket-трафик по секретному пути на xray (`localhost:xray_port`). Это работает для конфигураций VLESS/VMess/Trojan с транспортом `ws` (WebSocket). Другие транспорты (TCP, Reality, gRPC, mKCP, QUIC, xHTTP и т.д.) через этот reverse-proxy не пройдут — для них нужна другая архитектура (например, xray слушает 443 напрямую, без Caddy).
>
> Поддержка других типов подключений появится позже — либо в этом же репозитории, либо в отдельных сценариях Ansible.

> ❗ **ВАЖНО: после получения ссылки из 3x-ui замените `security=none` на `security=tls`**
>
> TLS терминируется в Caddy (порт 443, сертификат от Let's Encrypt), а между Caddy и xray трафик идёт по localhost без шифрования — поэтому в инбаунде 3x-ui TLS **умышленно выключен**.
>
> Следствие: ссылка, которую панель генерирует для клиента (`vless://...?security=none&type=ws&...`), из коробки **не будет работать**. Клиент увидит `security=none` и не будет ожидать TLS от сервера — а сервер (Caddy) отдаёт именно TLS.
>
> **Что делать:** в ссылке для клиента (или в шаблоне подписки) руками меняем `security=none` → `security=tls`. После этого подключение работает.
>
> Этот шаг нужно делать **при каждой генерации ссылки** из панели. Альтернатива — настроить подписочный сервис, который подменяет параметр автоматически.

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
- Установка 3x-ui + загрузка `x-ui.db` (автоматически: `webListen=127.0.0.1`, `webBasePath=xui_panel_path`, пути к сертификатам очищаются)
- Установка Caddy + деплой Caddyfile (прокси на панель x-ui + WebSocket на xray + fallback на сайт-камуфляж)
- Финальная перезагрузка → Caddy получает TLS-сертификат через Let's Encrypt → запуск x-ui

> **После успешного развёртывания** `deploy_user` больше не нужен. Рекомендуется удалить его вручную:
> ```bash
> sudo userdel -r <deploy_user>
> sudo rm -f /etc/sudoers.d/<deploy_user>
> ```

### Схема трафика

```
Клиент → 443 (Caddy, TLS)
           ├─ путь xui_panel_path/* → localhost:xui_panel_port (панель x-ui)
           ├─ путь xray_ws_path/*   → localhost:xray_port (xray, WebSocket)
           └─ всё остальное         → caddy_fallback_url (редирект, камуфляж)

Let's Encrypt → HTTP-01 (порт 80) → Caddy получает и автообновляет сертификат
```

Панель x-ui работает на localhost по HTTP, TLS терминируется в Caddy. Сертификат обновляется автоматически раз в ~60 дней — никаких скриптов синхронизации или рестартов не требуется.

---

## Структура проекта

```
ansible-vps/
├── site-init.yml                    # Этап 1 (порт 22, root)
├── site-configure.yml               # Этап 2 (кастомный порт, deploy_user)
├── inventory.ini.example            # Шаблон inventory
├── ansible.cfg
├── group_vars/
│   ├── new_vps.yml.example          # Шаблон общих переменных группы (коммитится)
│   └── new_vps.yml                  # Общие переменные (в .gitignore — не коммитится!)
├── host_vars/                       # Переменные конкретных хостов (для мульти-сервера)
│   ├── server1.yml.example          # Шаблон per-host переменных (коммитится)
│   └── server1.yml                  # Ваши переменные для server1 (в .gitignore!)
├── roles/bootstrap/
│   ├── tasks/
│   │   ├── main.yml                 # Диспетчер фаз
│   │   ├── upgrade.yml              # apt full-upgrade
│   │   ├── init.yml                 # Пользователи, SSH, firewall
│   │   └── configure.yml            # 3x-ui, Caddy
│   ├── handlers/main.yml
│   ├── templates/
│   │   ├── sshd_config.j2
│   │   ├── iptables_v4.j2
│   │   ├── iptables_v6.j2
│   │   └── Caddyfile.j2
│   └── files/                       # Файлы с эталона (в .gitignore!)
│       └── x-ui.db                  # единственный файл с эталона
├── INSTALL.md                       # Пошаговая инструкция
└── CHANGELOG.md
```

> ⚠️ **Имя `new_vps` используется в трёх местах и должно совпадать:**
> 1. `inventory.ini` → секция `[new_vps]`
> 2. `group_vars/new_vps.yml` → имя файла
> 3. `site-init.yml` / `site-configure.yml` → `hosts: new_vps`
>
> Если хочется переименовать группу (например, в `vpn_nodes`) — менять нужно **во всех трёх местах**, иначе Ansible не увидит переменные или плейбук не найдёт хосты.

---

## Развёртывание на несколько серверов

Плейбук поддерживает одновременное развёртывание на несколько серверов. Ansible по умолчанию работает с хостами параллельно (5 форков, настраивается через `forks` в `ansible.cfg` или флагом `-f N`).

### Разделение переменных

```
group_vars/new_vps.yml   → общие: users, ssh_port, acme_email, extra_packages, sysctl, ...
host_vars/server1.yml    → индивидуальные для server1: hostname, (опционально) пути, порты
host_vars/server2.yml    → индивидуальные для server2
```

**`hostname` обязательно должен быть уникальным для каждого сервера** — Caddy получает отдельный TLS-сертификат на каждый FQDN через Let's Encrypt.

### Пример на 3 сервера

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

> ⚠️ **Имя файла в `host_vars/` должно совпадать с именем хоста из `inventory.ini`** (левая колонка, не IP). То есть для `server1 ansible_host=1.2.3.4` → `host_vars/server1.yml`.

### Предусловия

- На каждый сервер заранее залит один и тот же SSH-ключ (`ssh-copy-id root@<IP>` на каждый IP)
- Для каждого FQDN в DNS настроена A-запись, указывающая на IP соответствующего сервера

> **Про `x-ui.db`:** в проекте всегда ровно один файл `roles/bootstrap/files/x-ui.db`, и он раскатывается на все серверы группы — это by design, ферма получает одинаковые инбаунды. Если нужны разные инбаунды на разных серверах, их проще настроить вручную в панели после развёртывания (или разнести серверы по разным группам с отдельными БД).

### Если серверы совсем разные

Создайте отдельные группы:

```ini
[node_fr]
fr1 ansible_host=1.2.3.4

[node_de]
de1 ansible_host=5.6.7.8

[new_vps:children]
node_fr
node_de
```

И разложите переменные: `group_vars/node_fr.yml`, `group_vars/node_de.yml` для специфики каждой группы, `group_vars/new_vps.yml` для общего.

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

Caddy получает и автообновляет TLS-сертификат через Let's Encrypt (HTTP-01 challenge на порту 80). Сертификат хранится в каталоге Caddy, сам x-ui работает на `127.0.0.1` по HTTP — доступ к панели через Caddy по секретному URL.

```yaml
acme_email: "admin@example.com"   # email для Let's Encrypt
```

> Ansible автоматически очищает `webCertFile`/`webKeyFile` в `x-ui.db` и
> выставляет `webListen=127.0.0.1`, `webBasePath=xui_panel_path` — трогать
> эталонный сервер не нужно.

> **Сертификаты в инбаундах 3X-UI** — если в настройках инбаундов используются отдельные
> TLS-сертификаты, они **не копируются автоматически**. После развёртывания необходимо вручную
> загрузить их на сервер и проверить пути и права доступа в настройках инбаунда.

### Caddy

Caddy слушает порт 443, получает TLS-сертификат через ACME и управляет трафиком:

- **Панель x-ui** доступна по пути `xui_panel_path` (proxy на `localhost:xui_panel_port`)
- **VPN-трафик** по пути `xray_ws_path` проксируется на xray через WebSocket
- **Остальной трафик** редиректится на `caddy_fallback_url` (сайт-камуфляж)

| Переменная | Пример | Описание |
|---|---|---|
| `acme_email` | `admin@example.com` | Email для Let's Encrypt (обязательно) |
| `caddy_fallback_url` | `https://example.com` | Сайт-камуфляж для не-VPN трафика |
| `xui_panel_path` | `/my-secret-panel` | Секретный путь к панели x-ui (начинается с `/`) |
| `xui_panel_port` | `54321` | Порт панели x-ui на localhost |
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
