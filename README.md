# Итоговое домашнее задание

Это проект для итогового задания по основам виртуализации и контейнеризации.

Я подняла виртуальную машину с Ubuntu в VMware, настроила доступ по SSH, включила firewall и запустила несколько сервисов через Docker Compose.

В проекте используется 3 контейнера:

* nginx как reverse proxy;
* простое web-приложение на Python;
* PostgreSQL как база данных.

Приложение открывается через nginx по адресу:

```text
http://172.16.24.130/
```

## Виртуальная машина

VM была создана в VMware.

Параметры виртуальной машины:

* ОС: Ubuntu 25.10
* виртуализация: VMware
* архитектура: arm64
* CPU: 2 vCPU
* RAM: 4 GB
* диск: 40 GB
* пользователь: student
* IP-адрес: 172.16.24.130

Подключение к VM с хоста выполнялось по SSH:

```bash
ssh student@172.16.24.130
```

Проверка пользователя:

```bash
whoami
```

Вывод:

```text
student
```

Пользователь `student` не root, но добавлен в группу `sudo`.

Проверка групп:

```bash
groups
```

Вывод:

```text
student sudo users docker
```

## Информация о системе

Команда:

```bash
hostnamectl
```

Основной вывод:

```text
Static hostname: Rose
Virtualization: vmware
Operating System: Ubuntu 25.10
Kernel: Linux 6.17.0-6-generic
Architecture: arm64
Hardware Vendor: VMware, Inc.
Hardware Model: VMware20,1
```

## Диск

Команда:

```bash
lsblk
```

Вывод:

```text
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0    40G  0 disk
├─nvme0n1p1 259:1    0   953M  0 part /boot/efi
└─nvme0n1p2 259:2    0  39,1G  0 part /
```

Команда:

```bash
df -h /
```

Вывод:

```text
Файл.система   Размер Использовано  Дост Использовано% Cмонтировано в
/dev/nvme0n1p2    39G          12G   26G           31% /
```

Диск был увеличен до 40 GB, чтобы было больше места для Docker и контейнеров.

## Сеть

Команда:

```bash
ip -br a
```

Вывод:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp2s0           UP             172.16.24.130/24 fe80::20c:29ff:fedb:d6a3/64
```

## Firewall

Для firewall использовался `ufw`.

Были разрешены только SSH и порт приложения:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw --force enable
```

Проверка:

```bash
sudo ufw status verbose
```

Вывод:

```text
Состояние: активен
Журналирование: on (low)
По умолчанию: deny (входящие), allow (исходящие), disabled (маршрутизированные)
Новые профили: skip

В                          Действие    Из
-                          --------    --
22/tcp (OpenSSH)           ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
```

## Docker

Docker был установлен внутри виртуальной машины.

Проверка версии Docker:

```bash
docker --version
```

Вывод:

```text
Docker version 29.1.3, build 29.1.3-0ubuntu3~25.10.1
```

Проверка версии Docker Compose:

```bash
docker compose version
```

Вывод:

```text
Docker Compose version 2.40.3+ds1-0ubuntu1~25.10.1
```

## Файлы проекта

В проекте есть такие файлы:

```text
.
├── .env
├── README.md
├── app
│   └── index.html
├── docker-compose.yml
└── nginx.conf
```

Файл `.env` используется для переменных окружения базы данных. Пароль от PostgreSQL не прописан напрямую в `docker-compose.yml`.

Состав сервисов:

* `proxy` — контейнер с nginx;
* `app` — контейнер с простым HTTP-приложением;
* `db` — контейнер с PostgreSQL.

Также в `docker-compose.yml` настроены:

* отдельная bridge-сеть `final_network`;
* named volume `postgres_data` для данных PostgreSQL;
* restart policy для `proxy` и `db`;
* healthcheck для базы данных.

## Запуск

Запуск проекта:

```bash
docker compose up -d
```

Проверка контейнеров:

```bash
docker compose ps
```

Вывод:

```text
NAME          IMAGE                 COMMAND                  SERVICE   CREATED              STATUS                        PORTS
final_app     python:3.12-alpine    "python -m http.serv…"   app       About a minute ago   Up About a minute             8000/tcp
final_db      postgres:16-alpine    "docker-entrypoint.s…"   db        About a minute ago   Up About a minute (healthy)   5432/tcp
final_proxy   nginx:stable-alpine   "/docker-entrypoint.…"   proxy     About a minute ago   Up About a minute             0.0.0.0:80->80/tcp, [::]:80->80/tcp
```

## Проверка приложения

Проверка через curl:

```bash
curl -I http://172.16.24.130/
```

Вывод:

```text
HTTP/1.1 200 OK
Server: nginx/1.30.2
Date: Mon, 08 Jun 2026 15:55:05 GMT
Content-Type: text/html
Content-Length: 1177
Connection: keep-alive
Last-Modified: Mon, 08 Jun 2026 18:30:44 GMT
```

Также приложение открывается в браузере на хостовой системе:

```text
http://172.16.24.130/
```

Через браузер открывается простая страница на русском языке с описанием проекта.

## Проверка сохранения данных в PostgreSQL

Для проверки персистентности была создана таблица `persistence_check` и добавлена тестовая запись.

Команда для входа в PostgreSQL внутри контейнера:

```bash
docker compose exec -T db psql -U app_user -d app_db
```

SQL-запросы:

```sql
CREATE TABLE IF NOT EXISTS persistence_check (
    id SERIAL PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO persistence_check (message)
VALUES ('Данные сохранились после перезапуска контейнеров');

SELECT id, message, created_at FROM persistence_check;
```

После этого контейнеры были остановлены и запущены заново:

```bash
docker compose down
docker compose up -d
```

Потом я снова проверила данные:

```bash
docker compose exec -T db psql -U app_user -d app_db -c "SELECT id, message, created_at FROM persistence_check;"
```

Вывод:

```text
 id |                     message                      |         created_at
----+--------------------------------------------------+----------------------------
  1 | Данные сохранились после перезапуска контейнеров | 2026-06-08 18:31:51.741912
(1 row)
```

Данные остались на месте после перезапуска контейнеров. Значит named volume `postgres_data` работает правильно.

## Остановка проекта

Остановить контейнеры:

```bash
docker compose down
```

Если нужно удалить контейнеры вместе с данными БД:

```bash
docker compose down -v
```

В обычной проверке `docker compose down -v` лучше не использовать, потому что он удаляет volume с данными PostgreSQL.
