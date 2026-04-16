# Инструкция по развёртыванию

## Краткий обзор

```
Шаг 1 (один раз)  — клонировать репозиторий, установить Ansible, настроить конфиг
Шаг 2 (каждый сервер)  — залить ключ, создать inventory, запустить 2 playbook-а
```

**Два playbook — два этапа:**
- `site-init.yml` — порт 22, root. Настраивает ОС, пользователей, SSH, firewall.
  После завершения: root заблокирован, SSH на кастомном порту. **Запускается один раз.**
- `site-configure.yml` — кастомный порт, ваш пользователь. Устанавливает 3x-ui и Caddy.
  **Идемпотентен** — можно перезапускать для обновления конфигурации.

---

## Шаг 1. Первоначальная настройка (один раз)

### 1.1. Клонировать репозиторий

```bash
git clone https://github.com/yukh975/ansible-vps-3xui.git ansible-vps
cd ansible-vps
```

---

### 1.2. Установить Ansible

**macOS:**
```bash
brew install ansible
```

**Debian/Ubuntu:**
```bash
sudo apt install ansible
```

Проверить:
```bash
ansible --version
```

---

### 1.3. Сгенерировать SSH-ключ (если нет)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ansible"
```

Сразу посмотреть публичный ключ — он понадобится в конфиге:
```bash
cat ~/.ssh/id_ed25519.pub
```

---

### 1.4. Скопировать x-ui.db с эталонного сервера

Это **единственный файл**, который берётся с эталонного сервера.
В нём хранятся инбаунды и настройки подключений.

```bash
scp root@<IP_ЭТАЛОНА>:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db
```

> **Про пути к сертификатам:**
> Панель x-ui доступна через секретный URL в Caddy — сертификат хранится в Caddy,
> панель работает на localhost по HTTP. Пути `webCertFile` и `webKeyFile` в БД
> очищаются автоматически. Поле `webListen` выставляется в `127.0.0.1`.
>
> **⚠️ Пути к сертификатам в инбаундах:**
> Если в инбаундах эталонного сервера прописаны явные пути к сертификатам —
> после деплоя зайдите в панель x-ui и обновите эти пути вручную.

---

### 1.5. Создать файл переменных

```bash
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
nano group_vars/new_vps.yml
```

> 💡 **Разворачиваете сразу несколько серверов?**
> `group_vars/new_vps.yml` содержит **общие** переменные для всех хостов в группе (пользователи, ssh_port, пакеты, sysctl). Индивидуальные параметры (как минимум `hostname` — для ACME-сертификата) выносятся в `host_vars/<имя_хоста>.yml`. См. раздел «Развёртывание на несколько серверов» в [README.md](README.md) и шаблон `host_vars/server1.yml.example`.

#### Обязательные параметры

| Параметр | Описание |
|---|---|
| `hostname` | FQDN сервера, например `vps1.example.com` — используется как hostname и для ACME-сертификата |
| `ssh_port` | SSH-порт после hardening (не 22, например `275`) |
| `deploy_user` | Имя пользователя для этапа 2 — **должен быть в `users`** |
| `acme_email` | Email для регистрации в Let's Encrypt |
| `users.<name>.password` | SHA-512 хэш пароля |
| `users.<name>.ssh_public_keys` | Список публичных SSH-ключей (минимум один) |
| `caddy_fallback_url` | Сайт-камуфляж — куда Caddy редиректит обычный трафик |
| `xui_panel_path` | Секретный путь к панели x-ui, например `/my-secret-panel` |
| `xray_ws_path` | Секретный путь WebSocket для xray, например `/my-secret-ws` |

#### Параметры пользователей (опционально)

| Параметр | По умолчанию | Описание |
|---|---|---|
| `users.<name>.groups` | `[]` | Группы Linux, например `["sudo"]` |
| `users.<name>.sudo_nopasswd` | `false` | NOPASSWD в sudoers |
| `users.<name>.has_bashrc` | `false` | Копировать `.bashrc` из `files/<name>/.bashrc` |

#### Сервисы (опционально)

| Параметр | По умолчанию | Описание |
|---|---|---|
| `xui_panel_port` | `54321` | Порт панели x-ui (localhost) |
| `xray_port` | `10000` | Порт xray WebSocket (localhost) |
| `install_3xui` | `true` | Устанавливать 3x-ui |
| `xui_version` | `""` (latest) | Версия 3x-ui, например `2.3.11` |

#### Firewall (опционально)

| Параметр | По умолчанию | Описание |
|---|---|---|
| `ipset_hosts` | `[]` | Доверенные IP с полным доступом |
| `ipset_set_name` | `allowed_hosts` | Имя ipset-сета |
| `allowed_tcp_ports` | `[80, 443]` | Открытые TCP-порты |
| `allowed_udp_ports` | `[]` | Открытые UDP-порты |

---

#### Как заполнять

**SSH-ключи** — вставить прямо в конфиг, можно несколько:
```bash
cat ~/.ssh/id_ed25519.pub
```
```yaml
users:
  myuser:
    ssh_public_keys:
      - "ssh-ed25519 AAAA...ключ_ansible... ansible"
      - "ssh-ed25519 AAAA...личный_ключ... personal-laptop"   # опционально
```

**Хэш пароля** — сгенерировать локально:
```bash
openssl passwd -6 'ВАШ_ПАРОЛЬ'
```

**`.bashrc`** — если нужны алиасы и кастомный промпт:
```bash
mkdir -p roles/bootstrap/files/myuser/
nano roles/bootstrap/files/myuser/.bashrc
```
Затем в конфиге: `has_bashrc: true`.

---

### 1.6. (Опционально) Настроить правила iptables

Порты и ipset задаются в конфиге (см. шаг 1.5). Если нужны дополнительные правила — отредактируй шаблоны напрямую **до запуска playbook**:

```
roles/bootstrap/templates/iptables_v4.j2   — правила IPv4
roles/bootstrap/templates/iptables_v6.j2   — правила IPv6
```

Уже прописаны: политики `INPUT DROP / FORWARD DROP / OUTPUT ACCEPT`, loopback, ESTABLISHED, ICMP, SSH-порт, порты из `allowed_tcp_ports` / `allowed_udp_ports`, ipset.

Добавляй свои правила между существующими блоками, например:

```
# Пример: разрешить входящий WireGuard
-A INPUT -p udp --dport 51820 -j ACCEPT
```

---

### 1.7. (Опционально) Зашифровать конфиг через ansible-vault

```bash
ansible-vault encrypt group_vars/new_vps.yml
```

При каждом запуске playbook добавлять флаг:
```bash
ansible-playbook ... --ask-vault-pass
```

Для внесения изменений — редактировать напрямую (откроет в $EDITOR):
```bash
ansible-vault edit group_vars/new_vps.yml
```

---

> ✅ **Шаг 1 выполнен.** При развёртывании следующих серверов начинайте с Шага 2.

---

## Шаг 2. Развёртывание нового сервера

### 2.1. Убедиться что DNS настроен

DNS нужен для получения TLS-сертификата через Caddy ACME (HTTP-01 challenge — Let's Encrypt обращается на порт 80 домена). Всё остальное — SSH, пользователи, firewall, 3x-ui — работает без DNS.

**Что именно должно быть настроено:** в DNS-зоне домена должна существовать **A-запись**, указывающая на IP нового сервера.

```bash
dig +short vps1.example.com
# должен вернуть IP нового сервера, например: 1.2.3.4
```

---

### 2.2. Залить SSH-ключ на сервер

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP>
```

> Команда запросит пароль root — используйте пароль, выданный хостингом.
> Это **единственный раз**, когда он понадобится: после этапа 1 пароль root будет изменён,
> вход по паролю через SSH будет запрещён, доступ — только по ключу.

---

### 2.3. Создать inventory

```bash
cp inventory.ini.example inventory.ini
nano inventory.ini   # вписать IP сервера вместо YOUR_SERVER_IP
```

Один `inventory.ini` работает для обоих этапов.

> 💡 **Несколько серверов сразу?** Добавьте строки в секцию `[new_vps]` и создайте `host_vars/<host>.yml` для каждого (с уникальным `hostname`). Ansible развернёт параллельно. Подробности — в [README.md](README.md).

---

### 2.4. Запустить этап 1 (порт 22, root)

```bash
ansible-playbook -i inventory.ini site-init.yml
# с vault:
ansible-playbook -i inventory.ini site-init.yml --ask-vault-pass
```

Что происходит:
1. `apt full-upgrade` (если есть обновления — автоматическая перезагрузка и продолжение)
2. Ожидание окончания cloud-init и освобождения блокировки apt (до 30 минут — некоторые хостеры накатывают обновления сразу после provisioning)
3. Установка пакетов из `extra_packages`
4. Создание пользователей, SSH-ключи из конфига, sudoers
5. Установка hostname
6. SSH hardening: кастомный порт, root заблокирован, только ключи
7. sysctl, ipset, iptables
8. Перезагрузка → SSH поднимается на `ssh_port`

⚠️ **После этого шага этап 1 нельзя запустить повторно** — SSH на 22 закрыт, root заблокирован.

---

### 2.5. Запустить этап 2 (ssh_port, deploy_user)

```bash
ansible-playbook -i inventory.ini site-configure.yml
# с vault:
ansible-playbook -i inventory.ini site-configure.yml --ask-vault-pass
```

Что происходит:
1. Установка 3x-ui (если не установлен)
2. Загрузка `x-ui.db`, обновление настроек в БД (webListen, webBasePath, очистка путей к сертификатам)
3. Установка Caddy из официального репозитория
4. Деплой Caddyfile (Caddy на порту 443: ACME + прокси на панель x-ui + WebSocket → xray + fallback)
5. Финальная перезагрузка → ожидание ACME-сертификата → запуск x-ui
   Плейбук завершается только когда Caddy получил сертификат и x-ui активен

Этот шаг **идемпотентен** — можно запускать повторно для обновления конфигурации.

---

### 2.6. Проверить результат

```bash
# Подключиться к серверу
ssh -p <ssh_port> <deploy_user>@<IP>

# Статус сервисов
sudo systemctl status x-ui caddy

# Панель x-ui доступна по адресу:
# https://<hostname><xui_panel_path>
```

> ❗ **При экспорте ссылки (или QR-кода) клиенту из панели** замените в ней `security=none` на `security=tls`.
> TLS в инбаунде выключен (его делает Caddy), и без этой правки клиент не подключится.
> QR-код из панели содержит ту же ссылку, поэтому тоже нерабочий — отдавайте клиенту ссылку/QR уже с `security=tls`.
> Подробнее — в [README.md](README.md), callout в самом начале.

---

### 2.7. Удалить технического пользователя (рекомендуется)

`deploy_user` — технический пользователь, нужен только Ansible для этапа 2. После успешного развёртывания он больше не нужен.

Подключитесь под основным пользователем (или root) и удалите:

```bash
sudo userdel -r <deploy_user>
sudo rm -f /etc/sudoers.d/<deploy_user>
```

> **Важно:** убедитесь, что у вас есть другой пользователь с SSH-доступом, прежде чем удалять `deploy_user`.

---

## Обновление конфигурации на работающем сервере

Только этап 2 — он идемпотентен и работает через `ssh_port`:

```bash
ansible-playbook -i inventory.ini site-configure.yml
```

Используйте для: обновления `x-ui.db`, изменения Caddyfile, изменения версии 3x-ui.

---

## Если что-то пошло не так

**Этап 1 завис или упал до конца:**
Сервер мог перезагрузиться со старым SSH-конфигом. Попробуйте подключиться на порту 22 от root — если получается, значит этап 1 не завершился. Исправьте проблему и запустите заново.

**apt завис надолго:**
Это нормально — некоторые хостеры запускают обновления сразу после provisioning.
Playbook ждёт до 30 минут (30 попыток × 60 секунд). В логе будут видны RETRYING-сообщения.

**Caddy не получил сертификат:**
```bash
sudo systemctl status caddy
sudo journalctl -u caddy -n 50
```
Убедитесь что DNS настроен (`dig +short <hostname>` возвращает IP сервера) и порт 80 открыт (`allowed_tcp_ports` содержит `80`).

**Панель x-ui недоступна:**
Проверьте что Caddy запущен и `xui_panel_path` указан корректно:
```bash
sudo systemctl status caddy
curl -s http://localhost:<xui_panel_port>  # должен ответить
```
