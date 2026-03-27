# VPS Bootstrap Ansible

Автоматическое развёртывание VPS с нуля: hardening, пользователи, firewall, 3x-ui.

Запускается один раз с управляющей машины на свежеустановленный Debian 12/13 с доступом root по паролю. После завершения сервер полностью готов к работе.

---

## Что делает

1. Полное обновление ОС (`apt full-upgrade`) и перезагрузка
2. Создание пользователей `yukh` и `ca` с SSH-ключами и паролями
3. SSH hardening — смена порта на 275, отключение root и парольного входа
4. Применение sysctl, ipset и iptables правил
5. Установка утилит: `mc`, `micro`, `telnet`, `traceroute`, `nslookup` и др.
6. Установка [3x-ui](https://github.com/MHSanaei/3x-ui), загрузка базы данных
7. Финальная перезагрузка — после неё всё работает

---

## Структура

```
├── site.yml                              # точка входа
├── inventory.ini                         # хосты
├── group_vars/
│   └── new_vps.yml                       # все переменные: порты, пакеты, пути
└── roles/bootstrap/
    ├── tasks/
    │   ├── main.yml                      # диспетчер фаз
    │   ├── upgrade.yml                   # apt upgrade + reboot
    │   └── configure.yml                 # основная конфигурация
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── sshd_config.j2                # шаблон SSH конфига
    └── files/
        ├── sysctl.conf                   # ваш sysctl конфиг
        ├── rules.v4                      # iptables IPv4 правила
        ├── rules.v6                      # iptables IPv6 правила
        ├── ipset.conf                    # ipset наборы
        ├── x-ui.db                       # база данных 3x-ui (не в репо)
        ├── yukh/
        │   ├── authorized_keys           # публичный SSH ключ
        │   └── .bashrc
        ├── ca/
        │   └── authorized_keys           # публичный SSH ключ
        └── root/
            └── .bashrc
```

---

## Подготовка

### 1. Заполнить файлы конфигурации

Скопировать с эталонного сервера:

```bash
ETALON=root@<IP>

scp $ETALON:/etc/x-ui/x-ui.db              roles/bootstrap/files/x-ui.db
scp $ETALON:/etc/sysctl.d/99-custom.conf   roles/bootstrap/files/sysctl.conf
ssh $ETALON "iptables-save"   > roles/bootstrap/files/rules.v4
ssh $ETALON "ip6tables-save"  > roles/bootstrap/files/rules.v6
ssh $ETALON "ipset save"      > roles/bootstrap/files/ipset.conf
```

Заполнить вручную:

```
roles/bootstrap/files/yukh/authorized_keys   # публичный ключ пользователя yukh
roles/bootstrap/files/ca/authorized_keys     # публичный ключ пользователя ca
roles/bootstrap/files/yukh/.bashrc           # .bashrc для yukh
roles/bootstrap/files/root/.bashrc           # .bashrc для root
```

### 2. Сгенерировать хэши паролей

```bash
openssl passwd -6 'пароль'
```

Вставить результат в `group_vars/new_vps.yml`:

```yaml
users:
  yukh:
    password: "$6$..."
  ca:
    password: "$6$..."
  root:
    password: "$6$..."
```

### 3. Заполнить inventory.ini

```ini
[new_vps]
vps1 ansible_host=<IP_СЕРВЕРА>

[new_vps:vars]
ansible_user=root
ansible_port=22
ansible_ssh_pass=<ПАРОЛЬ_ROOT>
```

### 4. Убедиться что в rules.v4 открыт порт 275

```
-A INPUT -p tcp --dport 275 -j ACCEPT
```

---

## Запуск

```bash
ansible-playbook -i inventory.ini site.yml
```

Если пароль не прописан в `inventory.ini`:

```bash
ansible-playbook -i inventory.ini site.yml --ask-pass
```

**Время выполнения: ~5–8 минут.**

---

## После развёртывания

```bash
# Подключение
ssh -p 275 yukh@<IP>

# Проверка x-ui
ssh -p 275 yukh@<IP> "sudo systemctl status x-ui"
```

---

## Файлы не входящие в репо

Добавьте в `.gitignore`:

```
roles/bootstrap/files/x-ui.db
roles/bootstrap/files/yukh/authorized_keys
roles/bootstrap/files/ca/authorized_keys
inventory.ini
```

---

## Зависимости

- Ansible 2.12+
- Python 3 на управляющей машине
- Целевой сервер: Debian 12 или 13, доступ root по паролю, порт 22
