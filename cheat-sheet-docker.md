## Шпаргалка (важные команды Docker)
### 1. Запуск сборки контейнеров
Собрать образ с тегом **myapp:latest** из **Dockerfile** в текущей директории и запустить контейнер
```bash
docker build -t myapp .
docker run -d --name myapp_container myapp
```
Собрать образы (если указан build) и запустить все сервисы в фоне
```bash
docker compose up -d --build
```
### 2. Пересборка контейнеров
Остановить и удалить старый контейнер, пересобрать образ, запустить заново
```bash
docker stop myapp_container && docker rm myapp_container
docker build -t myapp . --no-cache   # --no-cache — игнорировать кэш слоёв
docker run -d --name myapp_container myapp
```
Пересобрать образы (принудительно, без кэша) и пересоздать контейнеры
```bash
docker compose -f путь/к/файлу.yml build --no-cache
docker compose -f путь/к/файлу.yml up -d --force-recreate
# Или одной командой:
docker compose -f путь/к/файлу.yml up -d --build --force-recreate
```
### 3. Удаление контейнеров
Удаление конкретного контейнера
```bash
docker stop myapp_container     # остановить, если запущен
docker rm myapp_container       # удалить контейнер
# Или принудительно одной командой:
docker rm -f myapp_container
```
Удаление всех остановленных контейнеров
```bash
docker container prune -f
```
Остановить и удалить контейнеры, сети, созданные командой up
```bash
docker compose -f путь/к/файлу.yml down
```
Удалить также именованные volumes (данные), прикреплённые к сервисам
```bash
docker compose -f путь/к/файлу.yml down -v
```
### 4. Очистка памяти от временных файлов и артефактов
Общая очистка системы (неиспользуемые контейнеры, сети, образы, кэш сборки)
```bash
docker system prune -a --volumes
# -a — удалить все неиспользуемые образы (не только висячие)
# --volumes — также удалить неиспользуемые тома
```
Очистка только кэша сборки (build cache)
```bash
docker builder prune -a -f
```
Удаление "висячих" образов (<none>:<none>)
```bash
docker image prune -f
```
Удаление всех остановленных контейнеров, неиспользуемых сетей, висячих образов, кэша сборки (аналог system prune, но раздельно)
```bash
docker container prune -f
docker image prune -a -f
docker network prune -f
docker volume prune -f
docker builder prune -a -f
```
### 5. Вход в контейнеры
Запуск интерактивной оболочки в работающем контейнере
```bash
# bash (если доступен)
docker exec -it myapp_container /bin/bash
# sh (более универсальный вариант)
docker exec -it myapp_container /bin/sh
```
Запуск одноразовой команды
```bash
docker exec myapp_container ls -la /app
```
Подключение к контейнеру с правами root (если контейнер запущен под другим пользователем)
```bash
docker exec -u root -it myapp_container /bin/bash
```
### 6. Управление внутренней сетью
Просмотр существующих сетей
```bash
docker network ls
```
Создание пользовательской bridge-сети
```bash
docker network create my_network
```
Подключение контейнера к сети (на лету)
```bash
docker network connect my_network myapp_container
```
Отключение от сети
```bash
docker network disconnect my_network myapp_container
```
Просмотр деталей сети (подключенные контейнеры, IP)
```bash
docker network inspect my_network
```
Запуск контейнера сразу в нужной сети
```bash
docker run -d --name myapp_container --network my_network myapp
```
Удаление сети (перед удалением все контейнеры должны быть отключены)
```bash
docker network rm my_network
# или очистить все неиспользуемые сети разом:
docker network prune -f
```
Docker Compose автоматически создаёт отдельную сеть для проекта. Имя сети по умолчанию: **<каталог проекта>_default**.
### 7. Минимальная структура Dockerfile
```Dockerfile
# ---- Этап 1: сборка (builder) ----
# Используем официальный образ с нужным языком/инструментами
FROM golang:1.21-alpine AS builder

# Устанавливаем рабочую директорию внутри контейнера
WORKDIR /app

# Копируем файлы зависимостей (для кэширования слоёв)
COPY go.mod go.sum ./

# Загружаем зависимости
RUN go mod download

# Копируем исходный код
COPY . .

# Собираем статически слинкованный бинарник
RUN CGO_ENABLED=0 GOOS=linux go build -o /myapp ./cmd/main.go

# ---- Этап 2: финальный образ ----
# Используем минимальный образ (без лишних инструментов)
FROM alpine:3.19

# Устанавливаем сертификаты (нужны для HTTPS-запросов из приложения)
RUN apk --no-cache add ca-certificates

# Создаём непривилегированного пользователя для безопасности
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Копируем скомпилированный бинарник из этапа builder
COPY --from=builder /myapp /usr/local/bin/myapp

# Переключаемся на непривилегированного пользователя
USER appuser

# Указываем порт, который приложение будет слушать (чисто документация)
EXPOSE 8080

# Команда запуска приложения
CMD ["myapp"]
```
### 8. Минимальная структура docker-compose.yml
```yml
version: "3.9"  # Версия формата (опциональна в новых docker compose, но для совместимости оставляем)

services:
  # Сервис приложения
  app:
    # Сборка из Dockerfile в текущей директории (.)
    build:
      context: .
      args:
        - MY_VAR=${MY_VAR}   # значение из .env или shell
      dockerfile: Dockerfile
    # Тег для контейнера (чтобы не генерировалось автоматически)
    image: mayapp
    # Имя контейнера (чтобы не генерировалось автоматически)
    container_name: myapp_container
    # Подключаем entrypoint-script
    entrypoint: /entrypoint.sh
    # Проброс портов: "хост:контейнер"
    ports:
      - "8080:8080"
    env_file:
      - .env
    # Переменные окружения
    environment:
      environment:
      - DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}?sslmode=disable
    # Тома: монтируем локальную папку для разработки (hot-reload)
    volumes:
      - .:/app
    # Зависимости — этот сервис запустится только после старта db
    depends_on:
      db:
        condition: service_healthy  # дождаться healthcheck (см. ниже)
    # Сеть (если не указана, попадёт в default-сеть проекта)
    networks:
      - app-network

  # Сервис базы данных
  db:
    image: postgres:16-alpine  # Готовый образ из Docker Hub
    container_name: myapp_db
    env_file:
      - .env
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    # Постоянное хранилище данных
    volumes:
      - pgdata:/var/lib/postgresql/data
    # Healthcheck — проверка готовности базы
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-network

# Определение пользовательской сети (будет создана автоматически)
networks:
  app-network:
    driver: bridge

# Именованный том для данных БД (чтобы данные сохранялись между перезапусками)
volumes:
  pgdata:
```
Пример файла .env
```env
POSTGRES_USER=user
POSTGRES_PASSWORD=pass
POSTGRES_DB=mydb
DJANGO_SECRET_KEY=supersecret
DEBUG=False
```
> Примечание по версиям команд: В современных версиях Docker используется плагин Compose v2, команды выполняются как docker compose (без дефиса). Если у вас установлен старый docker-compose (v1), заменяйте docker compose на docker-compose.
### Эквивалент, через команды sh
1. Собрать образ для приложения (аналог build: .)
```bash
docker build -t myapp .
```
2. Создать том для данных БД (аналог volumes: pgdata)
```bash
docker volume create pgdata
```
3. Создать пользовательскую сеть (аналог networks: app-network)
```bash
docker network create app-network
```
4. Запустить базу данных (аналог сервиса db)
```bash
docker run -d \
  --name myapp_db \
  --network app-network \
  --restart unless-stopped \
  # -e POSTGRES_USER=user \
  # -e POSTGRES_PASSWORD=pass \
  # -e POSTGRES_DB=mydb \
  --env-file .env \
  -v pgdata:/var/lib/postgresql/data \
  --health-cmd "pg_isready -U \$(printenv POSTGRES_USER) -d \$(printenv POSTGRES_DB)" \
  --health-interval 5s \
  --health-timeout 5s \
  --health-retries 5 \
  postgres:16-alpine
```
5. Дождаться, пока БД станет healthy (замена depends_on с condition: service_healthy)
```bash
echo "Ожидание готовности PostgreSQL..."
until [ "$(docker inspect -f '{{.State.Health.Status}}' myapp_db)" = "healthy" ]; do
  sleep 1
done
echo "PostgreSQL готов"
```
6. Запустить приложение (аналог сервиса app)
```bash
docker run -d \
  --name myapp_container \
  --network app-network \
  --restart unless-stopped \
  -p 8080:8080 \
  # -e DATABASE_URL=postgres://user:pass@myapp_db:5432/mydb?sslmode=disable \
  --env-file .env \
  -v "$(pwd)":/app \
  myapp
```
## Entrypoint-скрипт: best practices, шаблоны и примеры

### Что такое entrypoint-скрипт?

Это исполняемый файл (обычно `entrypoint.sh`), который вызывается первым при старте контейнера.  
Он заменяет собой встроенный `ENTRYPOINT` образа и позволяет выполнить служебные задачи **перед** основным процессом.

### Зачем он нужен? (Best-practices)

- **Ожидание зависимостей** (БД, кэш, брокер) перед запуском приложения.
- **Автоматические миграции** схемы БД.
- **Установка прав** на файлы и каталоги (если монтируются volume).
- **Генерация конфигов** из переменных окружения.
- **Обработка сигналов** (`SIGTERM`, `SIGINT`) для корректного завершения (через `exec`).
- **Логирование** окружения (без секретов) для отладки.

### Общая структура entrypoint-скрипта

```bash
#!/bin/sh
set -e               # Завершить при любой ошибке

# 1. Ожидание внешних сервисов
# 2. Проверки и подготовка (миграции, права, конфиги)
# 3. Исполнение переданной команды (CMD) с передачей управления через exec
exec "$@"
```
## Создание и использование непривилегированных пользователей в Dockerfile
По умолчанию Docker выполняет все инструкции (`RUN`, `CMD`, `ENTRYPOINT`) от имени `root`.  
Это несёт риски для безопасности и вызывает предупреждения (например, pip).  
Создание пользователя без прав root — **обязательная практика** для production-образов.
### 1. Шаблон для Debian/Ubuntu (образы на основе `debian`, `ubuntu`, `python:slim`)

```dockerfile
# Создаём группу и пользователя
RUN groupadd --system --gid 1000 appgroup && \
    useradd --system --uid 1000 --gid appgroup --create-home appuser

# Опционально: создаём рабочую директорию и меняем владельца
WORKDIR /app
RUN chown -R appuser:appgroup /app

# Переключаемся на непривилегированного пользователя
USER appuser
```
Все последующие команды (RUN, CMD) выполняются от appuser
### 2. Шаблон для Alpine (образы на основе alpine, python:alpine)
```dockerfile
# В Alpine утилита adduser/addgroup (без useradd/groupadd)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
RUN chown -R appuser:appgroup /app

USER appuser
```
### 3. Создание пользователя с конкретным UID/GID
Полезно, когда нужно совпадение с UID/GID хоста для проброшенных томов (volume mounts).
```dockerfile
# Debian/Ubuntu
RUN groupadd --gid 1001 mygroup && \
    useradd --uid 1001 --gid mygroup --create-home myuser

# Alpine
RUN addgroup -g 1001 mygroup && adduser -u 1001 -G mygroup -D myuser
```
### 4. Копирование файлов с правильными правами
При копировании файлов до смены пользователя — владельцем будет root.
Лучше копировать файлы, затем менять владельца.
```dockerfile
COPY --chown=appuser:appgroup . /app
# или
COPY . /app
RUN chown -R appuser:appgroup /app
```
### 5. Использование пользователя в multi-stage сборках
В финальном образе обычно нет компиляторов, поэтому пользователя создаём именно там.
```dockerfile
# Стадия сборки (builder) — работает от root, это нормально
FROM golang:1.21-alpine AS builder
WORKDIR /build
COPY . .
RUN go build -o myapp .

# Финальный образ
FROM alpine:3.19
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
COPY --from=builder --chown=appuser:appgroup /build/myapp /usr/local/bin/myapp
USER appuser
CMD ["myapp"]
```
### 6. Как проверить, под каким пользователем работает контейнер
```bash
docker exec myapp_container whoami
# или
docker exec myapp_container id
```
### 7. Смена пользователя на root внутри работающего контейнера (только для отладки)
```bash
docker exec -u root -it myapp_container /bin/sh
```
> ### Важные замечания
> - Инструкция USER влияет только на команды, идущие после неё.
> - Если вашему приложению нужны привилегированные порты (например, 80), не запускайте его под root. Используйте `EXPOSE 8080` и проброс портов на хосте: `docker run -p 80:8080` ....
> - Для временного повышения прав (установка пакетов) в одном `RUN` слое используйте `sudo` или временный `USER root`, но лучше выполнять установку ДО переключения пользователя.
## Управление отдельными сервисами в Docker Compose
Часто нужно взаимодействовать не со всеми сервисами сразу, а только с одним (запустить, остановить, собрать).  
Все команды выполняются из директории с файлом `compose.yml` (или с указанием `-f`).

### Запуск и остановка

Запустить один сервис (в фоне) / Остановить и удалить только один сервис (контейнер останавливается, но не удаляется) / Запустить ранее остановленный сервис / Перезапустить сервис / Остановить и удалить контейнеры одного сервиса (без удаления томов).
```bash
docker compose up -d app
docker compose stop app
docker compose start app
docker compose restart app
docker compose down app
```
### Сборка и обновление образов
Собрать образ только для конкретного сервиса / Загрузить свежий образ из registry только для одного сервиса / Пересоздать контейнер только одного сервиса (если изменился образ или конфигурация)
```bash
docker compose build app
docker compose build --no-cache app
docker compose pull app
docker compose up -d --force-recreate app
```
### Выполнение команд и просмотр логов
Выполнить команду внутри контейнера сервиса / Просмотр логов конкретного сервиса (с отслеживанием) / Просмотр логов нескольких конкретных сервисов / Просмотр запущенных процессов в сервисе
```bash
docker compose exec app bash
docker compose exec app ls -la /app
docker compose logs -f app
docker compose logs -f app db
docker compose top app
```
### Информация о портах
Посмотреть, на какой порт хоста проброшен внутренний порт сервиса
```bash
docker compose port app 8080
```
### Удаление контейнера
Удалить остановленный контейнер одного сервиса / Удалить вместе с анонимными томами
```bash
docker compose rm app
docker compose rm -v app
```
## Популярные сценарии запуска контейнеров
### 1. Интерактивная разработка (shell внутри контейнера)

**Сценарий:** отладка, ручная работа с файлами, проверка окружения. Контейнер запускается с терминалом, часто с монтированием текущей папки.

```bash
# Docker
docker run -it --rm -v $(pwd):/app -w /app python:3.12-alpine /bin/sh

# Docker Compose (сервис command: /bin/sh, запуск без -d)
docker compose run --rm app /bin/sh
```
`-it` — интерактивный режим + TTY
`--rm` — удалить контейнер после выхода
`-v $(pwd):/app` — смонтировать текущую папку хоста в /app контейнера
`-w /app` — рабочая директория внутри контейнера

### 2. Долгоживущий сервис (веб-приложение)
**Сценарий**: запуск API, веб-сервера, базы данных. Контейнер работает в фоне и перезапускается при падениях.
```bash
# Docker
docker run -d --name my_app --restart unless-stopped -p 8080:8080 myapp:latest

# Docker Compose
docker compose up -d app
```
`-d` — детач-режим (фон)
`--restart unless-stopped` — автоматический перезапуск (кроме случаев ручной остановки)
`-p хост:контейнер` — проброс порта

### 3. Одноразовая задача (скрипт, миграция, cron)
**Сценарий**: выполнение команды и завершение контейнера. Часто используется с --rm, чтобы не оставлять мусора.
```bash
# Docker
docker run --rm -v $(pwd)/data:/data alpine:latest sh -c "cat /data/file.txt"

# Docker Compose (сервис должен быть настроен с командой по умолчанию)
docker compose run --rm app python manage.py migrate
```

### 4. Запуск с пробросом портов
**Сценарий**: сделать внутренний порт приложения доступным на хосте.
```bash
# Docker
docker run -d -p 80:8080 -p 443:8443 myapp

# Проброс на конкретный IP хоста
docker run -d -p 127.0.0.1:3306:3306 mysql:8
```

### 5. Передача переменных окружения
**Сценарий**: конфигурирование приложения без правки образа.
```bash
# Одна переменная
docker run -d -e DATABASE_URL=postgres://... myapp

# Из файла .env
docker run -d --env-file ./.env myapp
```
```yml
# Docker Compose (в файле compose.yml)
environment:
  - NODE_ENV=production
# или файл
env_file:
  - .env
```

### 6. Переопределение команды или точки входа
**Сценарий**: временно запустить другой процесс, не меняя Dockerfile.
```bash
# Переопределить команду (CMD)
docker run -it --rm alpine echo "Hello"

# Переопределить точку входа (ENTRYPOINT) и команду
docker run -it --rm --entrypoint /bin/sh myapp -c "ls /app"
```

### 7. Запуск с ограничением ресурсов (CPU, память)
**Сценарий**: предотвратить захват всех ресурсов хоста одним контейнером.
```bash
docker run -d --memory="256m" --cpus="1.5" myapp
```
`--memory` — максимальный объём ОЗУ
`--cpus` — доля ядер процессора (1.5 = полтора ядра)

### 8. Запуск с использованием GPU (nvidia)
**Сценарий**: машинное обучение, обработка видео.
```bash
docker run -d --gpus all nvidia/cuda:12.0-runtime-ubuntu22.04 nvidia-smi
```

### 9. Запуск стека сервисов (Docker Compose)
**Сценарий**: микросервисы, приложение с базой данных и кэшем.
```bash
docker compose up -d              # весь стек
docker compose up -d app db       # только выбранные
docker compose up -d --scale worker=3   # масштабирование (если нет привязки к портам)
```

### 10. Запуск контейнера в той же сети, что и другой (связь между контейнерами)
**Сценарий**: временно присоединить контейнер к существующей сети для отладки.
```bash
# Создать сеть
docker network create mynet
# Запустить первый контейнер в этой сети
docker run -d --net mynet --name db postgres
# Запустить временный контейнер в той же сети
docker run -it --rm --net mynet alpine ping db
```

### 11. Запуск контейнера с инициализацией базы (seed)
**Сценарий**: контейнер выполняет подготовительные операции и завершается.
```yml
# В compose.yml
services:
  init:
    image: alpine
    volumes:
      - ./init.sh:/init.sh
    command: /init.sh
    depends_on:
      - db
```
Запуск
```bash
docker compose up init
```

### 12. Запуск с монтированием Docker-сокета (Docker-in-Docker)
**Сценарий**: управлять хост-демоном Docker из контейнера (CI/CD, администрирование).
```bash
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock docker:latest sh
```
Внутри можно выполнять команды `docker` от имени хоста.
