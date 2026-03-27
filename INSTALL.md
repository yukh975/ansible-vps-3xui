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
mkdir -p roles/bootstrap/files/{USER1,USER2,root,certs}
```

Пример для пользователей `yukh` и `ca`:
```bash
mkdir -p roles/bootstrap/files/{yukh,ca,root,certs}
```

---

### A4. Скопировать файлы с эталонного сервера

Подставьте IP эталона, имена пользователей и домен:

```bash
ETALON=root@<IP_ЭТАЛОНА>
DEPLOY_USER=<имя_deploy_user>     # например: yukh
CERT_USER=<имя_cert_user>         # например: ca (если есть)
DOMAIN=<ваш_домен>                # например: example.com
```

**SSH-ключи пользователей:**
```bash
scp $ETALON:/home/$DEPLOY_USER/.ssh/authorized_keys \
    roles/bootstrap/files/$DEPLOY_USER/authorized_keys

scp $ETALON:/home/$CERT_USER/.ssh/authorized_keys \
    roles/bootstrap/files/$CERT_USER/authorized_keys
```

**Добавить ключ Ansible к deploy_user:**
```bash
cat ~/.ssh/id_ed25519.pub >> roles/bootstrap/files/$DEPLOY_USER/authorized_keys
```

**Настройки оболочки (для пользователей с `has_bashrc: true`):**
```bash
scp $ETALON:/home/$DEPLOY_USER/.bashrc roles/bootstrap/files/$DEPLOY_USER/.bashrc
scp $ETALON:/root/.bashrc              roles/bootstrap/files/root/.bashrc
```

**Системные конфиги:**
```bash
scp $ETALON:/etc/sysctl.d/99-custom.conf   roles/bootstrap/files/sysctl.conf
ssh $ETALON "iptables-save"   > roles/bootstrap/files/rules.v4
ssh $ETALON "ip6tables-save"  > roles/bootstrap/files/rules.v6
# ipset — адреса задаются в group_vars/new_vps.yml (ipset_friends), файл не нужен
```

**3x-ui база данных:**
```bash
scp $ETALON:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db
```

**Caddy конфиг:**
```bash
scp $ETALON:/etc/caddy/Caddyfile roles/bootstrap/files/Caddyfile
```

**TLS-сертификаты:**
```bash
scp $ETALON:/etc/ssl/$DOMAIN/fullchain.crt   roles/bootstrap/files/certs/fullchain.crt
scp $ETALON:/etc/ssl/$DOMAIN/$DOMAIN.crt     roles/bootstrap/files/certs/$DOMAIN.crt
scp $ETALON:/etc/ssl/$DOMAIN/$DOMAIN.key     roles/bootstrap/files/certs/$DOMAIN.key
```

> **Важно о сертификатах:**
> Пути к сертификатам на целевом сервере задаются через `certs.base_dir` в конфиге
> и **должны совпадать** с настройками в `x-ui.db` на эталонном сервере.
> Если меняете пути здесь — обновите и настройки в 3X-UI на эталоне.

---

### A5. Настроить переменные

```bash
cp group_vars/new_vps.yml.example group_vars/new_vps.yml
```

Отредактировать `group_vars/new_vps.yml`:

| Что заполнить | Пример |
|---|---|
| `hostname` | `vps1.example.com` |
| `domain` | `example.com` |
| `ssh_port` | `275` |
| `deploy_user` | `yukh` |
| `users.*.password` | Хэш SHA-512 (см. ниже) |

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

### B1. Залить SSH-ключ Ansible на новый сервер

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP_НОВОГО_СЕРВЕРА>
```

---

### B2. Создать inventory файл

```bash
cp inventory.ini.example inventory.ini
```

Вписать IP нового сервера. Один файл используется для обоих этапов — порт и пользователь для каждого этапа задаются автоматически в playbook.

---

### B3. Запустить этап 1 (порт 22, root)

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
6. Копирование TLS-сертификатов в `/etc/ssl/<домен>/`
7. Деплой sshd_config (кастомный порт, root закрыт, только ключи)
8. Применение sysctl, ipset, iptables
9. Перезагрузка → SSH теперь на кастомном порту, root закрыт

---

### B4. Запустить этап 2 (кастомный порт, deploy_user)

```bash
ansible-playbook -i inventory.ini site-configure.yml
```

Что происходит:
1. Установка 3x-ui (если ещё не установлен)
2. Загрузка `x-ui.db` с эталонного сервера
3. Установка Caddy + деплой Caddyfile
4. Финальная перезагрузка
5. Проверка статуса x-ui и caddy

---

## Проверка

```bash
# Подключиться к серверу
ssh -p <ssh_port> <deploy_user>@<IP>

# Проверить сервисы
ssh -p <ssh_port> <deploy_user>@<IP> "sudo systemctl status x-ui caddy"
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
