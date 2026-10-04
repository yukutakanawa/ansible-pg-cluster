# ansible-pg-cluster

> Кластер высокой доступности **PostgreSQL 16** на базе **etcd + Patroni + Ansible**

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue?logo=postgresql&logoColor=white)
![Patroni](https://img.shields.io/badge/Patroni-4.x-green)
![etcd](https://img.shields.io/badge/etcd-3.5-orange)
![Ansible](https://img.shields.io/badge/Ansible-2.21+-red?logo=ansible&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-yellow)

Проект автоматически разворачивает отказоустойчивый кластер PostgreSQL из трёх нод:

- **etcd** — обеспечивает консенсус и хранение метаданных кластера;
- **Patroni** — управляет переключением лидера (failover) и репликацией;
- **PostgreSQL 16** — хранит данные.

Всё развёртывание автоматизировано Ansible-ролями.

---

## 📋 Содержание

- [Состав кластера](#-состав-кластера)
- [Компоненты](#-компоненты)
- [Требования](#-требования)
- [Структура проекта](#-структура-проекта)
- [Запуск](#-запуск)
- [Проверка](#-проверка)
- [Тест репликации](#-тест-репликации)
- [Тест failover](#-тест-failover)
- [Теги](#-теги)
- [Линтинг](#-линтинг)
- [Проверка после перезагрузки](#-проверка-после-перезагрузки)
- [Лицензия](#-лицензия)

---

## 🖥 Состав кластера

| Нода          | IP               | ОС            | Роль              |
| :------------ | :--------------- | :------------ | :---------------- |
| 🟢 **ubuntu** | `192.168.100.10` | Ubuntu 24.04  | **Leader** + etcd |
| 🔵 **debian** | `192.168.100.11` | Debian 12     | Replica + etcd    |
| 🔵 **rocky**  | `192.168.100.20` | Rocky Linux 9 | Replica + etcd    |

---

## 🧩 Компоненты

### etcd 3.5

- Распределённое key-value хранилище, кворум **2 из 3**.
- Устанавливается из бинарника.
- Конфигурируется через YAML-шаблон.
- Запускается как **systemd-сервис**.

### PostgreSQL 16

- Установка из официального **PGDG**-репозитория.
- Systemd-сервис PostgreSQL **отключается** — управление полностью передаётся Patroni.

### Patroni 4.x

- Устанавливается в отдельный **venv** через `pip`.
- Управляет PostgreSQL: выбирает лидера через etcd, поднимает реплики, выполняет failover при отказе лидера.

---

## ⚙️ Требования

- **Ansible 2.21+** на управляющей машине
- Три Linux-ВМ (Ubuntu, Debian, Rocky) с SSH-доступом
- **Python 3** на целевых нодах
- Доступ в интернет для установки пакетов (PGDG, pip)

---

## 📁 Структура проекта

```
ansible-pg-cluster/
├── ansible.cfg
├── .ansible-lint
├── inventory.ini
├── group_vars/
│   └── all.yml
├── host_vars/
│   ├── ubuntu.yml
│   ├── debian.yml
│   └── rocky.yml
├── playbooks/
│   └── site.yml
└── roles/
    ├── etcd/
    ├── postgresql/
    └── patroni/
```

---

## 🚀 Запуск

### Полное развёртывание кластера

```bash
ansible-playbook playbooks/site.yml
```

### Развёртывание только одной роли (через теги)

```bash
ansible-playbook playbooks/site.yml --tags etcd_setup
ansible-playbook playbooks/site.yml --tags db_setup
ansible-playbook playbooks/site.yml --tags patroni_setup
```

### Применение к одной ноде

```bash
ansible-playbook playbooks/site.yml --limit ubuntu
```

---

## ✅ Проверка

### Статус кластера Patroni

```bash
sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```

Ожидаемый вывод — **1 Leader, 2 Replica**:

```
+ Cluster: pg-cluster ---+----+-----------+
| Member | Host           | Role    | State     |
+--------+----------------+---------+-----------+
| ubuntu | 192.168.100.10 | Leader  | running   |
| debian | 192.168.100.11 | Replica | streaming |
| rocky  | 192.168.100.20 | Replica | streaming |
+--------+----------------+---------+-----------+
```

### Статус кластера etcd

```bash
sudo /usr/local/bin/etcdctl \
  --endpoints=http://192.168.100.10:2379,http://192.168.100.11:2379,http://192.168.100.20:2379 \
  member list

sudo /usr/local/bin/etcdctl \
  --endpoints=http://192.168.100.10:2379,http://192.168.100.11:2379,http://192.168.100.20:2379 \
  endpoint health
```

---

## 🔄 Тест репликации

**На лидере (Ubuntu)** — создать таблицу и записать данные:

```bash
sudo -u postgres psql -c "CREATE TABLE test (id int, name text);"
sudo -u postgres psql -c "INSERT INTO test VALUES (1, 'hello');"
```

**На репликах (Debian, Rocky)** — проверить наличие данных:

```bash
sudo -u postgres psql -c "SELECT * FROM test;"
```

Ожидаемый результат — `1 | hello` на **обеих репликах**. ✅

---

## 🔥 Тест failover

**1. Остановить Patroni на текущем лидере:**

```bash
sudo systemctl stop patroni     # на Ubuntu
```

**2. Через 15–30 секунд проверить статус кластера с любой реплики:**

```bash
sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```

Patroni автоматически выберет **нового лидера** из реплик. 🎯

**3. Вернуть бывшего лидера в кластер:**

```bash
sudo systemctl start patroni    # на Ubuntu
```

Ubuntu вернётся в кластер как **реплика** (Patroni выполнит `pg_rewind`).

---

## 🏷 Теги

В плейбуке `site.yml` определены теги для выборочного запуска:

| Тег             | Что делает                    |
| :-------------- | :---------------------------- |
| `etcd_setup`    | Установка и настройка etcd    |
| `db_setup`      | Установка PostgreSQL из PGDG  |
| `patroni_setup` | Установка и настройка Patroni |

---

## 🧹 Линтинг

```bash
ansible-lint roles/ playbooks/
```

Все проверки проходят **без ошибок**. Стилевое правило
`var-naming[no-role-prefix]` отключено осознанно в `.ansible-lint`:
переменные `pg_*` используются совместно ролями `postgresql` и `patroni`
(одинаковые значения, разные роли) — общий префикс `pg_` обеспечивает
читаемость без дублирования.

---

## 🔁 Проверка после перезагрузки

После `reboot` всех трёх нод кластер собирается **автоматически**:

1. ⚙️ **etcd** поднимается как systemd-сервис, кворум восстанавливается.
2. 🎛 **Patroni** поднимается как systemd-сервис, выбирает лидера через etcd.
3. 🐘 **PostgreSQL** запускается Patroni — итог: **1 Leader, 2 Replica**.

---

## 📜 Лицензия

Проект распространяется под лицензией **MIT**.

---
