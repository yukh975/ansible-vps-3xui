# Инструкция по развёртыванию нового сервера

> Все команды выполняются из каталога `~/ansible-vps`

---

## Подготовка (один раз)

### 1. Распаковать архив

```bash
tar -xzf ansible-vps.tar.gz
cd ~/ansible-vps
```

### 2. Сгенерировать ключ Ansible на управляющей машине (если нет)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "ansible-control"
```

### 3. Скопировать файлы с эталонного сервера

```bash
ETALON=root@<IP_ЭТАЛОНА>

# authorized_keys пользователей
scp $ETALON:/home/yukh/.ssh/authorized_keys roles/bootstrap/files/yukh/authorized_keys
scp $ETALON:/home/ca/.ssh/authorized_keys   roles/bootstrap/files/ca/authorized_keys

# .bashrc пользователей
scp $ETALON:/home/yukh/.bashrc roles/bootstrap/files/yukh/.bashrc
scp $ETALON:/root/.bashrc      roles/bootstrap/files/root/.bashrc

# Конфиги
scp $ETALON:/etc/x-ui/x-ui.db             roles/bootstrap/files/x-ui.db
scp $ETALON:/etc/sysctl.d/99-custom.conf   roles/bootstrap/files/sysctl.conf
ssh $ETALON "iptables-save"   > roles/bootstrap/files/rules.v4
ssh $ETALON "ip6tables-save"  > roles/bootstrap/files/rules.v6
ssh $ETALON "ipset save"      > roles/bootstrap/files/ipset.conf
```

### 4. Добавить ключ Ansible к пользователю yukh

```bash
cat ~/.ssh/id_ed25519.pub >> roles/bootstrap/files/yukh/authorized_keys
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

---

## Развёртывание нового сервера

### 6. Залить SSH ключ Ansible на новый сервер

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@<IP_НОВОГО_СЕРВЕРА>
```

Введёте пароль root один раз.

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
- настройка SSH (порт 275, root закрыт, только ключи)
- применение sysctl, iptables, ipset

### 9. Запустить этап 2 (порт 275, yukh)

```bash
ansible-playbook -i inventory-configure.ini site-configure.yml
```

Выполняет:
- установка 3x-ui
- загрузка базы данных x-ui.db
- финальная перезагрузка

---

## Готово

```bash
# Подключение после развёртывания
ssh -p 275 yukh@<IP>

# Проверка x-ui
ssh -p 275 yukh@<IP> "sudo systemctl status x-ui"
```

---

## Повторный запуск (обновление конфигурации)

```bash
# Этап 1 — если менялись пакеты, пользователи, firewall
ansible-playbook -i inventory-init.ini site-init.yml

# Этап 2 — если менялась конфигурация 3x-ui
ansible-playbook -i inventory-configure.ini site-configure.yml
```
