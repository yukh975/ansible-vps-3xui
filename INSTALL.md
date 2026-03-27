# Инструкция по развёртыванию

> Все команды выполняются из корня проекта (`~/ansible-vps` или где он у вас)

---

## Часть A. Первоначальная настройка (один раз)

Эти шаги выполняются однократно при первой установке проекта.
При развёртывании новых серверов переходите сразу к [Части B](#часть-b-развёртывание-нового-сервера).

---

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

### A2. Сгенерировать SSH-ключ Ansible (если нет)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ansible-control"
```

---

### A3. Создать каталоги для файлов с эталонного сервера

Для каждого пользователя, описанного в конфиге, нужен свой каталог:

```bash
mkdir -p roles/bootstrap/files/{USER1,root}
```

Пример для пользователей `myuser` и `myuser2`:
```bash
mkdir -p roles/bootstrap/files/{myuser,myuser2,root}
```

---

### A4. Скопировать файлы с эталонного сервера

Подставьте IP эталона, имена пользователей и домен:

```bash
ETALON=root@<IP_ЭТАЛОНА>
DEPLOY_USER=<имя_deploy_user>     # например: myuser
```

**SSH-ключи пользователей** (для пользователей с `has_authorized_keys: true`):
```bash
scp $ETALON:/home/$DEPLOY_USER/.ssh/authorized_keys \
    roles/bootstrap/files/$DEPLOY_USER/authorized_keys
```

**Добавить ключ Ansible к deploy_user:**
```bash
cat ~/.ssh/id_ed25519.pub >> roles/bootstrap/files/$DEPLOY_USER/authorized_keys
```

**Настройки оболочки** (для пользователей с `has_bashrc: true`):
```bash
scp $ETALON:/home/$DEPLOY_USER/.bashrc roles/bootstrap/files/$DEPLOY_USER/.bashrc
scp $ETALON:/root/.bashrc              roles/bootstrap/files/root/.bashrc
```

**3x-ui база данных:**
```bash
scp $ETALON:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db
```

> **Что копировать НЕ нужно:**
> - `sysctl.conf` — генерируется из `sysctl_settings` в конфиге
> - `rules.v4` / `rules.v6` — генерируются из шаблонов
> - `Caddyfile` — генерируется из шаблона
> - **TLS-сертификаты** — Caddy получает их автоматически через Let's Encrypt

> **Важно о путях сертификатов:**
> Переменная `certs_dest_dir` (куда cert-sync копирует сертификаты) **должна совпадать**
> с путями, прописанными в `x-ui.db` на эталонном сервере (раздел TLS в настройках инбаунда).
> Если меняете `certs_dest_dir` — обновите и в 3X-UI на эталоне.

---

### A5. Настроить переменные

```bash
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
```

Отредактировать `group_vars/new_vps.yml`:

| Что заполнить | Пример | Описание |
|---|---|---|
| `hostname` | `vps1.example.com` | FQDN сервера |
| `domain` | `example.com` | Домен (должен указывать на IP сервера) |
| `ssh_port` | `275` | SSH-порт после hardening |
| `deploy_user` | `myuser` | Пользователь для этапа 2 |
| `acme_email` | `admin@example.com` | Email для Let's Encrypt |
| `certs_dest_dir` | `/etc/ssl/example.com` | Путь к сертификатам (должен совпадать с x-ui.db) |
| `caddy_https_port` | `8443` | HTTPS-порт Caddy (не 443 — занят 3x-ui) |
| `caddy_redirect_url` | `127.0.0.1:2053` | Куда Caddy проксирует запросы |
| `users.*.password` | Хэш SHA-512 | (см. ниже) |
| `ipset_hosts` | `["1.2.3.4"]` | Список доверенных IP (полный доступ) |
| `allowed_tcp_ports` | `[80, 443, 8443]` | TCP-порты открытые для всех |
| `sysctl_settings` | dict | Параметры ядра (можно оставить по умолчанию) |

**Сгенерировать хэши паролей:**
```bash
openssl passwd -6 'ВАШ_ПАРОЛЬ'
```

Вставить результат в поле `password` каждого пользователя.

**Рекомендация — зашифровать файл:**
```bash
ansible-vault encrypt group_vars/new_vps.yml
```
При запуске playbook добавлять `--ask-vault-pass`.

---

> **Готово.** Файлы на месте, переменные заполнены.
> При развёртывании новых серверов часть A не повторяется.

---

## Часть B. Развёртывание нового сервера

---

### B1. Убедиться, что DNS настроен

Домен (`domain` в конфиге) должен указывать на IP нового сервера —
Caddy будет получать сертификат через HTTP-01 challenge на этом домене.

```bash
nslookup example.com
# или
dig example.com +short
```

---

### B2. Залить SSH-ключ Ansible на новый сервер

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP_НОВОГО_СЕРВЕРА>
```

---

### B3. Создать inventory файл

```bash
cp inventory.ini.example inventory.ini
```

Вписать IP нового сервера. Один файл используется для обоих этапов — порт и пользователь для каждого этапа задаются автоматически в playbook.

---

### B4. Запустить этап 1 (порт 22, root)

```bash
ansible-playbook -i inventory.ini site-init.yml
```

> Если используете vault: `ansible-playbook -i inventory.ini site-init.yml --ask-vault-pass`

Что происходит:
1. `apt full-upgrade` + перезагрузка (если были обновления)
2. Установка пакетов из списка `extra_packages`
3. Создание всех пользователей из `users` (кроме root)
4. Установка hostname
5. Копирование SSH-ключей, .bashrc, настройка sudoers
6. Деплой sshd_config (кастомный порт, root закрыт, только ключи)
7. Применение sysctl, ipset, iptables
8. Перезагрузка → SSH теперь на кастомном порту, root закрыт

---

### B5. Запустить этап 2 (кастомный порт, deploy_user)

```bash
ansible-playbook -i inventory.ini site-configure.yml
```

Что происходит:
1. Установка 3x-ui (если ещё не установлен)
2. Загрузка `x-ui.db` (настройки инбаундов и путей к сертификатам)
3. Установка Caddy из официального репозитория
4. Деплой Caddyfile (ACME email, домен, HTTPS-порт, reverse proxy)
5. Деплой `cert-sync.sh` + systemd drop-in (`ExecStartPost`)
6. Финальная перезагрузка — Caddy стартует и получает TLS-сертификат
7. `cert-sync.sh` копирует сертификаты в `certs_dest_dir`, перезапускает x-ui
8. Проверка статуса x-ui и caddy

---

### B6. Проверка

```bash
# Подключиться к серверу
ssh -p <ssh_port> <deploy_user>@<IP>

# Проверить сервисы
sudo systemctl status x-ui caddy

# Проверить сертификаты
ls -la /etc/ssl/<domain>/

# Проверить cert-sync лог
sudo journalctl -u caddy --no-pager | grep cert-sync
```

---

## Обновление конфигурации на работающем сервере

> **Важно:** этап 1 (`site-init.yml`) предназначен **только для первичного развёртывания**.
> Он подключается на порт 22 от root — после hardening этот доступ закрыт навсегда.
> Повторный запуск этапа 1 на уже настроенном сервере завершится ошибкой подключения.

Для обновления конфигурации на работающем сервере используйте **только этап 2**:

```bash
# Обновить x-ui.db, Caddyfile — перезалить и перезапустить сервисы
ansible-playbook -i inventory.ini site-configure.yml
```

Этап 2 идемпотентен: проверяет наличие бинарников перед установкой, перезагружает только если что-то изменилось.
