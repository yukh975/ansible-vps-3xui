# Инструкция по развёртыванию нового сервера

> Все команды выполняются из каталога `~/ansible-vps`

---

## Первоначальная настройка (один раз)

### 1. Распаковать архив

```bash
tar -xzf ansible-vps.tar.gz
cd ~/ansible-vps
```

### 2. Создать каталоги для файлов

```bash
mkdir -p roles/bootstrap/files/yukh
mkdir -p roles/bootstrap/files/ca
mkdir -p roles/bootstrap/files/root
mkdir -p roles/bootstrap/files/certs
```

### 3. Сгенерировать ключ Ansible (если нет)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ansible-control"
```

### 4. Скопировать все файлы с эталонного сервера

```bash
ETALON=root@<IP_ЭТАЛОНА>

# SSH ключи пользователей
scp $ETALON:/home/yukh/.ssh/authorized_keys roles/bootstrap/files/yukh/authorized_keys
scp $ETALON:/home/ca/.ssh/authorized_keys   roles/bootstrap/files/ca/authorized_keys

# Добавить ключ Ansible к пользователю yukh
cat ~/.ssh/id_ed25519.pub >> roles/bootstrap/files/yukh/authorized_keys

# .bashrc пользователей
scp $ETALON:/home/yukh/.bashrc roles/bootstrap/files/yukh/.bashrc
scp $ETALON:/root/.bashrc      roles/bootstrap/files/root/.bashrc

# Конфиги системы
scp $ETALON:/etc/sysctl.d/99-custom.conf   roles/bootstrap/files/sysctl.conf
ssh $ETALON "iptables-save"   > roles/bootstrap/files/rules.v4
ssh $ETALON "ip6tables-save"  > roles/bootstrap/files/rules.v6
ssh $ETALON "ipset save"      > roles/bootstrap/files/ipset.conf

# 3x-ui база данных
scp $ETALON:/etc/x-ui/x-ui.db roles/bootstrap/files/x-ui.db

# Caddy конфиг
scp $ETALON:/etc/caddy/Caddyfile roles/bootstrap/files/Caddyfile

# Сертификаты
scp $ETALON:/home/ca/certs/netadm.pro/fullchain.crt  roles/bootstrap/files/certs/fullchain.crt
scp $ETALON:/home/ca/certs/netadm.pro/netadm.pro.crt roles/bootstrap/files/certs/netadm.pro.crt
scp $ETALON:/home/ca/certs/netadm.pro/netadm.pro.key roles/bootstrap/files/certs/netadm.pro.key
```

### 5. Прописать пароли в group_vars/new_vps.yml

Сгенерировать хэши:
```bash
openssl passwd -6 'пароль_yukh'
openssl passwd -6 'пароль_ca'
openssl passwd -6 'пароль_root'
```

Вставить результаты в `group_vars/new_vps.yml`:
```yaml
users:
  yukh:
    password: "$6$..."
  ca:
    password: "$6$..."
  root:
    password: "$6$..."
```

> После этого проект готов к использованию. При развёртывании новых серверов
> шаги 1-5 не повторяются — файлы уже на месте.

---

## Развёртывание нового сервера

### 6. Залить SSH ключ Ansible на новый сервер

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP_НОВОГО_СЕРВЕРА>
```

### 7. Создать inventory файлы

```bash
cp inventory-init.ini.example inventory-init.ini
cp inventory-configure.ini.example inventory-configure.ini
```

Вписать IP нового сервера в оба файла.

### 8. Запустить этап 1 (порт 22, root)

```bash
ansible-playbook -i inventory-init.ini site-init.yml
```

Выполняет:
- apt full-upgrade + перезагрузка
- установка пакетов
- создание пользователей yukh и ca
- копирование сертификатов
- настройка SSH (порт 275, root закрыт, только ключи)
- применение sysctl, iptables, ipset
- перезагрузка

### 9. Запустить этап 2 (порт 275, yukh)

```bash
ansible-playbook -i inventory-configure.ini site-configure.yml
```

Выполняет:
- установка 3x-ui + загрузка x-ui.db
- установка Caddy + Caddyfile
- финальная перезагрузка

---

## Готово

```bash
ssh -p 275 yukh@<IP>
ssh -p 275 yukh@<IP> "sudo systemctl status x-ui caddy"
```

---

## Повторный запуск (обновление конфигурации)

```bash
# Этап 1 — пакеты, пользователи, сертификаты, firewall
ansible-playbook -i inventory-init.ini site-init.yml

# Этап 2 — 3x-ui, Caddy
ansible-playbook -i inventory-configure.ini site-configure.yml
```
