## Как заставить работать свежую версию Playwright на старой Astra Linux (без изменения системного окружения)
### Описание окружения
- **ОС**: Astra Linux Special Edition (Смоленск), построена на пакетной базе Debian 9 (Stretch).
- **Ключевое ограничение**: системная glibc 2.24, которую категорически нельзя обновлять во избежание поломки ОС.
- **Цель**: использовать последнюю версию Playwright (Python) с браузерами Chromium, Firefox, WebKit, в том числе в визуальном режиме на рабочем столе Astra.
- **Установленные инструменты**: Python 3.14, uv, Docker.

### Проблемы, с которыми столкнулись, и почему прямые методы не сработали
1. **Ошибка загрузки встроенного Node.js в Playwright**
   `playwright install` выдавал ошибки о неудовлетворённых версиях GLIBC (2.25, 2.27, 2.28) и CXXABI. Playwright поставляется с собственным бинарником Node, скомпилированным под более новую glibc.
2. **Попытка подмены Node.js**
   Установили Node 18 через неофициальные сборки с поддержкой glibc 2.17. Playwright перестал ругаться на Node, но браузеры всё равно не запустились: им требовалась glibc ≥ 2.25 (ошибка `getrandom@GLIBC_2.25`).
3. **Попытка откатить Playwright до версии, совместимой с Node 16**
   Playwright 1.45 использовал Chromium 125+, который тоже требовал glibc 2.25. Даже с рабочей Node 16 браузер не стартовал.
4. **Попытка доустановить библиотеки через apt**
   Современный Playwright загружает браузеры для Ubuntu 24.04, которым нужны `libgtk-4`, `libicu74` и множество других библиотек, отсутствующих в репозиториях Astra. Пакеты не найдены.
5. **Подкладывание библиотек из Ubuntu 24.04 через LD_LIBRARY_PATH**
   Идея была рабочей, но возникали транзитивные зависимости, а главное — оставалась несовместимость с glibc 2.24.

Вывод: **запустить современные браузеры Playwright напрямую в старой системе невозможно без обновления glibc, что недопустимо**. Решение — контейнеризация.

### Окончательное решение: Docker с Ubuntu 24.04 и пробросом X11
Создаём Docker-образ на базе Ubuntu 24.04, в котором будут работать все браузеры. Проект монтируется с хоста, а для визуального режима пробрасывается сокет X-сервера Astra.

#### 1. Подготовка хоста (Astra Linux)
- Установите Docker:
```bash
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Выйдите и зайдите заново
```
- Для графического режима разрешите локальные подключения к X-серверу (выполнять перед запуском контейнера):
```bash
xhost +local:
```
#### 2. Dockerfile
Поместите в корень проекта (рядом с `pyproject.toml`) файл `Dockerfile`:
```dockerfile
FROM ubuntu:24.04

ENV PLAYWRIGHT_NODEJS_PATH=/usr/bin/node

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3-pip \
    python3-venv \
    nodejs \
    npm \
    curl \
    # Зависимости для браузеров Playwright
    libnss3 libnspr4 libdbus-1-3 libatk1.0-0 libatk-bridge2.0-0 libcups2 libdrm2 \
    libxkbcommon0 libxcomposite1 libxdamage1 libxfixes3 libxrandr2 libgbm1 \
    libpango-1.0-0 libcairo2 libasound2t64 libatspi2.0-0 \
    # Виртуальный X-сервер для headless-режима
    xvfb \
    && rm -rf /var/lib/apt/lists/*

# Установка uv (менеджер пакетов)
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.local/bin:$PATH"

# Установка Playwright с браузерами и всеми их зависимостями
RUN uv tool install playwright
RUN uv tool run playwright install --with-deps chromium firefox webkit

WORKDIR /tests
CMD ["/bin/bash"]
```
Обратите внимание на `libasound2t64` — в Ubuntu 24.04 пакет `libasound2` заменён на этот.

#### 3. Сборка образа
В корне проекта выполните:
```bash
docker build -t playwright-astra .
```
Если при предыдущих попытках остались кэши, используйте `--no-cache` для чистоты.

#### 4. Запуск контейнера
##### Визуальный режим (браузер будет открываться на рабочем столе Astra):
```bash
docker run --rm -it \
    -v "$(pwd)":/tests \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    -e DISPLAY="$DISPLAY" \
    --network host \
    playwright-astra \
    /bin/bash
```
Не забудьте предварительно выполнить `xhost +local:` на хосте.

##### Фоновый (headless) режим — например, для CI:
```bash
docker run --rm -it \
    -v "$(pwd)":/tests \
    playwright-astra \
    /bin/bash
```

#### 5. Запуск тестов внутри контейнера

Все команды выполняются из каталога `/tests`. Используйте `uv run` — он автоматически подтянет зависимости.
- Стандартный headless-запуск (Playwright по умолчанию headless):
```bash
uv run python test_browser.py
```
- Принудительный headless (если скрипт написан с headless=False):
```bash
PLAYWRIGHT_HEADLESS=true uv run python test_browser.py
```
- Визуальный запуск (при проброшенном X11):
```bash
uv run python test_browser.py   # в коде должно быть headless=False
```
- Запуск через виртуальный X-сервер (xvfb), если не хочется менять код и нет реального дисплея:
```bash
xvfb-run uv run python test_browser.py
```

### Возможные ошибки и их устранение
1. `E: Unable to locate package xvfb` при сборке
   Убедитесь, что `xvfb` находится в том же списке `apt-get install`, что и остальные пакеты, и предваряется `apt-get update`.
2. `externally-managed-environment` при установке uv через pip
   Используйте официальный скрипт установки `curl -LsSf ... | sh`, как в Dockerfile.
3. `Package libasound2 has no installation candidate`
   Замените `libasound2` на `libasound2t64`.
4. **При запуске браузера в визуальном режиме**: `Missing X server or $DISPLAY`
   Проверьте, что на хосте выполнен `xhost +local:` и что в контейнер проброшены `/tmp/.X11-unix` и переменная `DISPLAY`.
5. Недостаток места в `/var/lib/docker`
   Перенесите Docker-хранилище на другой раздел через параметр `data-root` в `/etc/docker/daemon.json` или символьную ссылку.

### Итог

С помощью Docker мы полностью изолировали современное окружение от старой системы, не изменив ни одного файла в Astra. Вы получили:
- Возможность запускать свежий Playwright с Chromium, Firefox, WebKit.
- Визуальную отладку на родном рабочем столе.
- Воспроизводимое и переносимое тестовое окружение.
