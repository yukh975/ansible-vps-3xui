# Changelog

## [0.9.1] — 2026-03-28

### Исправлено

- **`init.yml`** — добавлен `DEBIAN_FRONTEND=noninteractive` в apt-задачу установки пакетов. Без этого `ipset-persistent` задаёт интерактивный вопрос при установке («сохранить текущие правила?»), что блокирует Ansible.

---

## [0.9.0] — 2026-03-27

### Исправлено

- **`caddy-override.conf.j2`** — ExecStartPost теперь запускается с правами root (префикс `+`). Caddy работает от пользователя `caddy`, без `+` cert-sync.sh не мог записывать в `/etc/ssl/`
- **`Caddyfile.j2`** — добавлен явный блок `http://{{ hostname }}` на порту 80 (обязателен для ACME HTTP-01 challenge). Без него Caddy не слушал порт 80
- **`Caddyfile.j2`** — убран глобальный `https_port`, вместо него явный порт в блоке сайта: `{{ hostname }}:{{ caddy_https_port }}`
- **`init.yml`** — ipset set теперь создаётся всегда (даже пустым), чтобы iptables правила не падали при отсутствии хостов в `ipset_hosts`

### Добавлено

- Переменная `caddy_https_port` (default: `8443`) — порт HTTPS для Caddy, не конфликтует с x-ray на 443; сертификат получается через HTTP-01 на порту 80
- Пакет `ipset-persistent` в `extra_packages` — обеспечивает корректное сохранение и восстановление ipset при загрузке (плагин `10-ipsets` запускается до `15-ip4tables`)
- Переводы `README_EN.md`, `INSTALL_EN.md`, `CHANGELOG_EN.md`

### Изменено

- Переменная `domain` удалена — везде используется `hostname` (FQDN сервера используется и для hostname, и для ACME-сертификата)
- Флаг `install_caddy` удалён — Caddy является обязательным компонентом
- **`configure.yml`** — после перезагрузки: останавливаем x-ui, ждём сертификат от Caddy (до 5 минут), запускаем cert-sync, запускаем x-ui. Плейбук завершается только когда всё гарантированно работает

---

## [0.8.0] — 2026-03-27

### Тестирование на чистой VM, исправление критических ошибок

### Исправлено

- **`configure.yml`** — убран `test_command: systemctl is-active caddy` из задачи финальной перезагрузки. При `install_caddy: false` или медленном ACME команда всегда завершалась ошибкой, Ansible висел 5 минут до таймаута.
- **`init.yml`** — применение iptables, сохранение netfilter-persistent и перезагрузка объединены в один `async: 30, poll: 0` shell-скрипт. Раньше применение правил разрывало SSH-соединение до того как Ansible успевал выполнить следующие задачи.
- **`init.yml`** — удаление `authorized_keys` root перенесено в тот же финальный async-скрипт, иначе задача выполнялась после разрыва соединения и падала с ошибкой.
- **`init.yml`** — убраны `{{ item.key }}` из имён задач (вызывало предупреждение `'item' is undefined`). Отображение имени пользователя перенесено в `loop_control.label`.
- **`init.yml`** — копирование `.bashrc` пропускается если файл не существует (`lookup('fileglob', ...)`), вместо ошибки.
- **`ansible.cfg`** — добавлен `bin_ansible_callbacks = True` для корректной работы счётчика задач `community.general.counter_enabled`.
- **`group_vars/new_vps.yml.example`** — убран лишний знак `=` в значениях BBR (`net.core.default_qdisc: = fq` → `net.core.default_qdisc: fq`).

### Добавлено

- **`configure.yml`** — задача `[3xui] Обновить пути к сертификатам в БД`: после заливки `x-ui.db` автоматически обновляет `webCertFile` и `webKeyFile` через `sqlite3`, используя переменные `certs_dest_dir` и `domain`. Устраняет необходимость вручную менять пути в x-ui.db на эталонном сервере при смене домена.
- **`configure.yml`** — рекомендация удалить `deploy_user` после завершения настройки отображается в финальном сообщении.
- **`INSTALL.md`** — шаг 2.7 «Удалить технического пользователя» с командой `userdel -r`.
- **`README.md`** — примечание об удалении `deploy_user` после этапа 2.
- **`init.yml`** — загрузка модуля `tcp_bbr` (`modprobe`) и постоянная активация через `/etc/modules-load.d/tcp_bbr.conf`.
- **`templates/sysctl.conf.j2`** — шаблон для `/etc/sysctl.d/99-custom.conf`, заменяет модуль `ansible.posix.sysctl`.
- **`.gitkeep`** — файл-заглушка в `roles/bootstrap/files/` чтобы каталог существовал после `git clone`.

### Удалено

- **`requirements.yml`** — убран вместе с зависимостью от коллекции `ansible.posix`. Все модули заменены на `ansible.builtin`: `authorized_key` → `copy`, `sysctl` → `template + command`.

---

## [0.7.0] — 2026-03-27

### INSTALL.md — полная переработка

### Изменено

- **INSTALL.md** — переписан с нуля: добавлен шаг `git clone`, полная таблица всех параметров с разбивкой на обязательные/дополнительные, инструкции по SSH-ключам (несколько ключей на пользователя), хэшу пароля и опциональному `.bashrc`, vault-шифрованию.
- **`group_vars/new_vps.yml.example`** — убраны захардкоженные имена пользователей, заменены нейтральным `myuser`. Убран второй тестовый пользователь.
- **`CHANGELOG.md`** — убраны упоминания конкретных имён и доменов.

---

## [0.6.0] — 2026-03-27

### SSH-ключи в конфиге, убран .bashrc

Устранена последняя зависимость от файлов пользователей. Теперь для развёртывания нового сервера нужен только `x-ui.db`.

### Изменено

- **SSH authorized_keys** — вместо копирования файла `files/<user>/authorized_keys` ключи теперь задаются списком `ssh_public_keys` прямо в `group_vars/new_vps.yml`. Используется модуль `authorized_key` с `subelements`.
- **`.bashrc`** — поддержка копирования `.bashrc` для пользователей и root убрана (системный дефолт достаточен для Ansible-управляемых серверов).
- **`group_vars/new_vps.yml`** — убраны флаги `has_authorized_keys` и `has_bashrc`.
- **`INSTALL.md`** — упрощена Часть A: вместо `scp authorized_keys` + `mkdir` — одна команда `cat ~/.ssh/id_ed25519.pub` и вставка в конфиг.

### Удалено

- Таски `[users] Создать ~/.ssh`, `[users] authorized_keys` (файловый copy) из `init.yml`
- Таски `[users] .bashrc` и `[users] .bashrc для root` из `init.yml`
- Необходимость создавать каталоги `roles/bootstrap/files/<user>/`

### Итог: файлы с эталонного сервера

Единственный файл, который нужно скопировать с эталона: **`x-ui.db`**.

---

## [0.5.0] — 2026-03-27

### Переработка документации

Полностью переписаны README.md и INSTALL.md для ясности и удобства использования.

### Изменено

- **README.md** — добавлена схема «Быстрый старт», таблица портов, описание автоматического управления сертификатами. Убраны устаревшие ссылки на ручное копирование сертификатов.
- **INSTALL.md** — структура A/B (один раз / каждый сервер). Пошаговые инструкции B4/B5 с чётким описанием что происходит на каждом шаге. Добавлен раздел «Если что-то пошло не так».

---

## [0.4.0] — 2026-03-27

### Управление сертификатами через Caddy ACME

Ручное копирование TLS-сертификатов с эталонного сервера заменено на автоматическое получение через Let's Encrypt.

### Изменено

- **Сертификаты** — убран блок копирования из `init.yml`. Сертификаты теперь получаются автоматически через Caddy ACME (HTTP-01 challenge).
- **Caddyfile** — добавлены: `email {{ acme_email }}`, `https_port {{ caddy_https_port }}`. Caddy слушает на нестандартном HTTPS-порту (default: 8443), чтобы не конфликтовать с 3x-ui на 443. HTTP-01 challenge работает через порт 80.
- **iptables** — порты по умолчанию: `80` (ACME + HTTP), `443` (3x-ui), `8443` (Caddy HTTPS).
- **Документация** — убраны `scp`-команды для сертификатов, добавлено описание ACME и cert-sync.

### Добавлено

- `templates/cert-sync.sh.j2` — скрипт синхронизации сертификатов из Caddy в `certs_dest_dir`. Ждёт получения сертификата, копирует только если изменился, перезапускает x-ui.
- `templates/caddy-override.conf.j2` — systemd drop-in для Caddy: `ExecStartPost=/usr/local/bin/cert-sync.sh`. Обеспечивает синхронизацию и при первом запуске, и при автообновлении.
- Переменные: `acme_email`, `certs_dest_dir`, `caddy_https_port`.

### Удалено

- Блок `[certs]` из `init.yml` — больше не нужен (сертификаты выдаёт Caddy).
- Необходимость копировать сертификаты с эталона (`scp .crt/.key`).
- Переменная `certs` (словарь с `base_dir` и `files`) — заменена на `certs_dest_dir`.

---

## [0.3.0] — 2026-03-27

### Параметризация sysctl, iptables, Caddyfile

Полностью убрана зависимость от файлов с эталонного сервера для системных конфигов.

### Изменено

- **sysctl** — вместо копирования файла `sysctl.conf` используется словарь `sysctl_settings` в конфиге. Ansible применяет каждый параметр через модуль `sysctl` (идемпотентно).
- **iptables** — правила генерируются из Jinja2-шаблонов `iptables_v4.j2` / `iptables_v6.j2`. SSH-порт и ipset-сет подставляются автоматически. Настраиваются через `allowed_tcp_ports` / `allowed_udp_ports`.
- **Caddyfile** — генерируется из шаблона `Caddyfile.j2`. URL проксирования задаётся через `caddy_redirect_url` (default: `127.0.0.1:2053`). TLS-пути берутся из `certs.base_dir`.
- **ipset в iptables** — адреса из `ipset_hosts` получают полный доступ (`-j ACCEPT` на все протоколы).

### Добавлено

- `templates/iptables_v4.j2` — шаблон правил IPv4 с поддержкой ipset
- `templates/iptables_v6.j2` — шаблон правил IPv6
- `templates/Caddyfile.j2` — шаблон конфига Caddy
- Переменные: `sysctl_settings`, `allowed_tcp_ports`, `allowed_udp_ports`, `caddy_redirect_url`

### Удалено

- Необходимость копировать с эталона: `sysctl.conf`, `rules.v4`, `rules.v6`, `Caddyfile`
- Переменные: `sysctl_src`, `iptables_rules_v4_src`, `iptables_rules_v6_src`

---

## [0.2.1] — 2026-03-27

### Исправлено

- **Документация:** убрано ошибочное утверждение «Playbook идемпотентен». Этап 1 (`site-init.yml`) предназначен **только для первичного развёртывания** — после смены SSH-порта и закрытия root повторный запуск невозможен. Этап 2 (`site-configure.yml`) идемпотентен. В `README.md` и `INSTALL.md` добавлены соответствующие предупреждения.

---

## [0.2.0] — 2026-03-27

### Универсализация проекта

Полная параметризация — убраны все захардкоженные имена пользователей, домен и пути.

### Изменено

- **Пользователи** — единый словарь `users` с флагами (`sudo_nopasswd`, `has_authorized_keys`, `has_bashrc`). Таски итерируют по словарю вместо отдельных блоков для каждого пользователя.
- **Переменная `deploy_user`** — имя пользователя для этапа 2 (ранее захардкожено).
- **Переменная `domain`** — домен для путей к сертификатам (ранее захардкожено).
- **Сертификаты** перенесены из `/home/ca/certs/<домен>/` в `/etc/ssl/<домен>/`. Путь настраивается через `certs.base_dir`. Владелец — root.
- **3x-ui и Caddy** — опциональные через флаги `install_3xui` и `install_caddy` (по умолчанию `true`).
- **`ansible.cfg`** — `interpreter_python` изменён с `/usr/bin/python3.13` на `auto_silent`.
- **`site-init.yml` / `site-configure.yml`** — убраны хардкоды из комментариев.
- **Inventory** объединён в один файл `inventory.ini` вместо двух (`inventory-init.ini` + `inventory-configure.ini`). Порт и пользователь для каждого этапа теперь задаются в самих playbook.
- **ipset** — вместо копирования файла `ipset.conf` с эталонного сервера, IP-адреса задаются списком `ipset_hosts` в конфиге. Имя set'а задаётся через `ipset_set_name` (по умолчанию `allowed_hosts`). Ansible создаёт set и добавляет адреса автоматически.

### Добавлено

- `inventory.ini.example` — единый шаблон inventory.
- `group_vars/new_vps.yml.example` — шаблон переменных (коммитится в git).
- `CHANGELOG.md` — этот файл.

### Удалено

- `inventory-init.ini.example` и `inventory-configure.ini.example` — заменены единым `inventory.ini.example`.

### Безопасность

- `.gitignore` расширен: `roles/bootstrap/files/`, `group_vars/new_vps.yml`, `inventory-*.ini` больше не попадут в коммит.

---

## [0.1.0] — 2026-03-27

### Первая версия

- Двухэтапное развёртывание VPS (init + configure).
- Установка пакетов, создание пользователей, SSH hardening, sysctl, iptables, ipset.
- Установка 3x-ui с загрузкой БД, Caddy с Caddyfile.
- Шаблон sshd_config через Jinja2.
