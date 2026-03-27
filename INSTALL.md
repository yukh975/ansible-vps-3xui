# Инструкция по развёртыванию

## Общая схема

```
Часть A  — один раз, при первой установке проекта
Часть B  — для каждого нового сервера
```

---

## Часть A. Первоначальная настройка (один раз)

### A1. Установить Ansible

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

### A2. Сгенерировать SSH-ключ для Ansible (если нет)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ansible"
```

---

### A3. Создать файл переменных

```bash
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
```

Открыть `group_vars/new_vps.yml` и заполнить:

| Параметр | Что вписать |
|---|---|
| `hostname` | FQDN сервера, например `vps1.example.com` |
| `domain` | Домен, например `example.com` |
| `ssh_port` | SSH-порт после hardening (например `275`) |
| `deploy_user` | Имя пользователя, от которого работает этап 2 |
| `acme_email` | Email для Let's Encrypt |
| `certs_dest_dir` | Путь к сертификатам (должен совпадать с x-ui.db!) |
| `caddy_https_port` | HTTPS-порт Caddy (8443 по умолчанию, т.к. 443 занят 3x-ui) |
| `caddy_redirect_url` | Куда Caddy проксирует запросы (обычно `127.0.0.1:2053`) |
| `users.*.password` | Хэши паролей SHA-512 (см. ниже) |
| `ipset_hosts` | Список доверенных IP с полным доступом |
| `allowed_tcp_ports` | Открытые TCP-порты (80, 443, 8443 по умолчанию) |

**Сгенерировать хэш пароля:**
```bash
openssl passwd -6 'ВАШ_ПАРОЛЬ'
```

**Опционально — зашифровать файл vault'ом:**
```bash
ansible-vault encrypt group_vars/new_vps.yml
# При каждом запуске playbook добавлять: --ask-vault-pass
```

---

### A4. Подготовить файлы с эталонного сервера

Создать каталоги для каждого пользователя из конфига:
```bash
mkdir -p roles/bootstrap/files/{ИМЯ_ПОЛЬЗОВАТЕЛЯ,root}
# Пример: mkdir -p roles/bootstrap/files/{myuser,root}
```

Скопировать файлы:
```bash
ETALON=root@<IP_ЭТАЛОНА>
USER=<deploy_user>

# SSH-ключи
scp $ETALON:/home/$USER/.ssh/authorized_keys \
    roles/bootstrap/files/$USER/authorized_keys

# Добавить ключ Ansible к пользователю
cat ~/.ssh/id_ed25519.pub >> roles/bootstrap/files/$USER/authorized_keys

# .bashrc (если has_bashrc: true)
scp $ETALON:/home/$USER/.bashrc  roles/bootstrap/files/$USER/.bashrc
scp $ETALON:/root/.bashrc        roles/bootstrap/files/root/.bashrc

# 3x-ui база данных
scp $ETALON:/etc/x-ui/x-ui.db   roles/bootstrap/files/x-ui.db
```

> **Что копировать НЕ нужно:**
> `sysctl.conf`, `rules.v4/v6`, `Caddyfile` — генерируются из переменных.
> TLS-сертификаты — Caddy получает их сам через Let's Encrypt.

> **Важно про `certs_dest_dir`:**
> Путь, куда cert-sync кладёт сертификаты, **должен совпадать** с настройками TLS
> в `x-ui.db` (раздел TLS в настройках инбаунда).
> Если меняете `certs_dest_dir` — обновите и в 3X-UI на эталоне.

---

> ✅ **Часть A выполнена.** При развёртывании следующих серверов начинайте с Части B.

---

## Часть B. Развёртывание нового сервера

### B1. Убедиться что DNS настроен

Домен из конфига должен указывать на IP нового сервера.
Caddy получает TLS-сертификат через HTTP-01 challenge — для этого нужен рабочий DNS.

```bash
dig +short example.com
# должен вернуть IP нового сервера
```

---

### B2. Залить SSH-ключ на сервер

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP>
```

---

### B3. Создать inventory

```bash
cp inventory.ini.example inventory.ini
```

Вписать IP сервера вместо `YOUR_SERVER_IP`. Один файл работает для обоих этапов.

---

### B4. Этап 1 — базовая настройка (порт 22, root)

```bash
ansible-playbook -i inventory.ini site-init.yml
```

С vault: `ansible-playbook -i inventory.ini site-init.yml --ask-vault-pass`

Что происходит:
1. `apt full-upgrade` (если есть обновления — перезагрузка)
2. Установка пакетов из `extra_packages`
3. Создание пользователей, SSH-ключи, .bashrc, sudoers
4. Hostname
5. SSH hardening: кастомный порт, root закрыт, только ключи
6. sysctl, ipset, iptables
7. Перезагрузка → SSH поднимается на `ssh_port`

⚠️ **После этого шага этап 1 больше нельзя запустить повторно** — SSH на 22 закрыт, root закрыт.

---

### B5. Этап 2 — установка сервисов (ssh_port, deploy_user)

```bash
ansible-playbook -i inventory.ini site-configure.yml
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

### B6. Проверка

```bash
# Подключиться к серверу
ssh -p <ssh_port> <deploy_user>@<IP>

# Статус сервисов
sudo systemctl status x-ui caddy

# Сертификаты на месте
ls -la /etc/ssl/<domain>/

# Лог cert-sync
sudo journalctl -u caddy --no-pager | grep cert-sync
```

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
ls -la /etc/ssl/<domain>/
sudo journalctl -u caddy --no-pager | grep cert-sync
```
Убедитесь что `certs_dest_dir` совпадает с путём в настройках инбаунда в x-ui.
