# ansible-vps

Автоматическое развёртывание VPS с нуля: hardening, пользователи, firewall, 3x-ui, Caddy.

Запускается в два этапа с управляющей машины на свежеустановленный Debian 12/13.

---

## Что делает

**Этап 1** (порт 22, root):
- Полное обновление ОС (`apt full-upgrade`) + перезагрузка
- Установка пакетов: `sudo`, `mc`, `micro`, `telnet`, `traceroute`, `nslookup` и др.
- Создание пользователей `yukh` и `ca` с SSH-ключами, паролями и NOPASSWD sudo
- Копирование сертификатов в `/home/ca/certs/netadm.pro/`
- SSH hardening — порт 275, root закрыт, только ключи
- Применение sysctl, iptables, ipset

**Этап 2** (порт 275, yukh):
- Установка [3x-ui](https://github.com/MHSanaei/3x-ui) + загрузка базы данных
- Установка Caddy + копирование Caddyfile
- Финальная перезагрузка

---

## Структура

```
├── site-init.yml                         # Этап 1 — порт 22, root
├── site-configure.yml                    # Этап 2 — порт 275, yukh
├── inventory-init.ini.example            # Пример inventory для этапа 1
├── inventory-configure.ini.example       # Пример inventory для этапа 2
├── ansible.cfg                           # Настройки Ansible
├── group_vars/
│   └── new_vps.yml                       # Переменные: порты, пакеты, пароли
└── roles/bootstrap/
    ├── tasks/
    │   ├── main.yml                      # Диспетчер фаз
    │   ├── upgrade.yml                   # apt upgrade + reboot
    │   ├── init.yml                      # Этап 1: пользователи, SSH, firewall
    │   └── configure.yml                 # Этап 2: 3x-ui, Caddy
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── sshd_config.j2
    └── files/
        ├── sysctl.conf                   # ваш sysctl конфиг
        ├── rules.v4                      # iptables IPv4
        ├── rules.v6                      # iptables IPv6
        ├── ipset.conf                    # ipset наборы
        ├── Caddyfile                     # конфиг Caddy
        ├── x-ui.db                       # база данных 3x-ui (не коммитить)
        ├── certs/
        │   ├── fullchain.crt             # сертификаты (не коммитить)
        │   ├── netadm.pro.crt
        │   └── netadm.pro.key
        ├── yukh/
        │   ├── authorized_keys
        │   └── .bashrc
        ├── ca/
        │   └── authorized_keys
        └── root/
            └── .bashrc
```

---

## Быстрый старт

См. [INSTALL.md](INSTALL.md) для полной инструкции.

```bash
# Этап 1
ansible-playbook -i inventory-init.ini site-init.yml

# Этап 2
ansible-playbook -i inventory-configure.ini site-configure.yml
```

---

## Зависимости

- Ansible 2.12+
- Управляющая машина: Linux
- Целевой сервер: Debian 12 или 13, root по SSH, порт 22
