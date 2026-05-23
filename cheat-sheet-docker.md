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
      dockerfile: Dockerfile
    # Имя контейнера (чтобы не генерировалось автоматически)
    container_name: myapp_container
    # Проброс портов: "хост:контейнер"
    ports:
      - "8080:8080"
    # Переменные окружения
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb?sslmode=disable
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
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
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
> Примечание по версиям команд: В современных версиях Docker используется плагин Compose v2, команды выполняются как docker compose (без дефиса). Если у вас установлен старый docker-compose (v1), заменяйте docker compose на docker-compose.
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
