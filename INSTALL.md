# Инструкция по развёртыванию

```
Шаг 1  — один раз, при первой установке проекта
Шаг 2  — для каждого нового сервера
```

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
В нём хранятся инбаунды и настройки TLS.

```bash
scp root@<IP_ЭТАЛОНА>:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db
```

> **Про `certs_dest_dir` и пути в x-ui.db:**
> Ansible автоматически обновляет пути к сертификатам в `x-ui.db` после заливки —
> прописывает туда `certs_dest_dir/hostname.crt` и `certs_dest_dir/hostname.key`.
> Трогать эталонный сервер не нужно.
>
> **⚠️ Пути к сертификатам в инбаундах:**
> Ansible обновляет только пути к сертификатам **панели** x-ui (`webCertFile`, `webKeyFile`).
> Если в инбаундах эталонного сервера настроен TLS с явными путями к сертификатам —
> эти пути **не обновляются автоматически**. После деплоя зайдите в панель x-ui,
> проверьте настройки TLS каждого инбаунда и при необходимости обновите пути вручную.

---

### 1.5. Создать файл переменных

```bash
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
```

Открыть `group_vars/new_vps.yml` и заполнить параметры:

#### Обязательные

| Параметр | Описание |
|---|---|
| `hostname` | FQDN сервера, например `vps1.example.com` — используется как hostname и для ACME-сертификата |
| `ssh_port` | SSH-порт после hardening (не 22) |
| `deploy_user` | Имя пользователя для этапа 2 — **должен быть в `users`** |
| `acme_email` | Email для регистрации в Let's Encrypt |
| `certs_dest_dir` | Путь к сертификатам (пути в x-ui.db обновляются автоматически) |
| `users.<name>.password` | SHA-512 хэш пароля пользователя |
| `users.<name>.ssh_public_keys` | Список публичных SSH-ключей (минимум один) |

#### Параметры пользователей (опционально)

| Параметр | По умолчанию | Описание |
|---|---|---|
| `users.<name>.groups` | `[]` | Группы Linux, например `["sudo"]` |
| `users.<name>.sudo_nopasswd` | `false` | NOPASSWD в sudoers |
| `users.<name>.has_bashrc` | `false` | Копировать `.bashrc` из `files/<name>/.bashrc` |

#### Сервисы (опционально)

| Параметр | По умолчанию | Описание |
|---|---|---|
| `caddy_https_port` | `8443` | HTTPS-порт Caddy — не 443 (занят x-ray), закрыт снаружи; сертификат получается через HTTP-01 на порту 80 |
| `caddy_listen_port` | `4443` | Порт, на который x-ray отправляет fallback-трафик |
| `caddy_fallback_url` | — | Внешний сайт-камуфляж, куда Caddy редиректит трафик |
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

Для внесения изменений — расшифровать, отредактировать, зашифровать обратно:
```bash
ansible-vault decrypt group_vars/new_vps.yml
nano group_vars/new_vps.yml
ansible-vault encrypt group_vars/new_vps.yml
```

Или редактировать не расшифровывая (откроет в $EDITOR):
```bash
ansible-vault edit group_vars/new_vps.yml
```

---

> ✅ **Шаг 1 выполнен.** При развёртывании следующих серверов начинайте с Шага 2.

---

## Шаг 2. Развёртывание нового сервера

### 2.1. Убедиться что DNS настроен

DNS нужен для получения TLS-сертификата через Caddy ACME (HTTP-01 challenge — Let's Encrypt обращается на порт 80 домена). Всё остальное — SSH, пользователи, firewall, 3x-ui — работает без DNS.

**Что именно должно быть настроено:** в DNS-зоне домена должна существовать **A-запись**, указывающая на IP нового сервера. То, что у вас работает интернет и резолвится `google.com` — не считается.

```bash
dig +short example.com
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
```

Вписать IP сервера вместо `YOUR_SERVER_IP`. Один файл работает для обоих этапов.

---

### 2.4. Запустить этап 1 (порт 22, root)

```bash
ansible-playbook -i inventory.ini site-init.yml
# с vault:
ansible-playbook -i inventory.ini site-init.yml --ask-vault-pass
```

Что происходит:
1. `apt full-upgrade` (если есть обновления — перезагрузка)
2. Установка пакетов из `extra_packages`
3. Создание пользователей, SSH-ключи из конфига, sudoers
4. Установка hostname
5. SSH hardening: кастомный порт, root закрыт, только ключи
6. sysctl, ipset, iptables
7. Перезагрузка → SSH поднимается на `ssh_port`

⚠️ **После этого шага этап 1 нельзя запустить повторно** — SSH на 22 закрыт, root закрыт.

---

### 2.5. Запустить этап 2 (ssh_port, deploy_user)

```bash
ansible-playbook -i inventory.ini site-configure.yml
# с vault:
ansible-playbook -i inventory.ini site-configure.yml --ask-vault-pass
```

Что происходит:
1. Установка 3x-ui (если не установлен)
2. Загрузка `x-ui.db` (инбаунды, настройки TLS)
3. Установка Caddy из официального репозитория
4. Деплой Caddyfile (ACME, домен, порты)
5. Деплой `cert-sync.sh` + systemd drop-in (`ExecStartPost`)
6. Финальная перезагрузка:
   - Caddy стартует → получает TLS-сертификат (HTTP-01)
   - `cert-sync.sh` копирует сертификаты в `certs_dest_dir`
   - x-ui перезапускается с новым сертификатом

Этот шаг **идемпотентен** — можно запускать повторно для обновления конфигурации.

---

### 2.6. Проверить результат

```bash
# Подключиться к серверу
ssh -p <ssh_port> <deploy_user>@<IP>

# Статус сервисов
sudo systemctl status x-ui caddy

# Сертификаты на месте
ls -la /etc/ssl/<hostname>/

# Лог cert-sync
sudo journalctl -u caddy --no-pager | grep cert-sync
```

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

**Caddy не получил сертификат:**
```bash
sudo systemctl status caddy
sudo journalctl -u caddy -n 50
```
Убедитесь что DNS настроен и порт 80 открыт (`allowed_tcp_ports` содержит `80`).

**x-ui не видит сертификат:**
```bash
ls -la /etc/ssl/<hostname>/
sudo journalctl -u caddy --no-pager | grep cert-sync
```
Убедитесь что `certs_dest_dir` совпадает с путём в настройках инбаунда в x-ui.
