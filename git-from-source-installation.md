```
# 1. Установка зависимостей, необходимых для сборки Git
sudo apt update
sudo apt install -y make gcc libssl-dev libcurl4-openssl-dev libexpat1-dev \
    gettext zlib1g-dev curl

# 2. Переход во временную директорию и загрузка исходников последней версии Git
cd /tmp
# Замените URL на нужную версию, если хотите другую. Актуальную версию смотрите на https://git-scm.com/
GIT_VERSION="2.47.0"
wget https://github.com/git/git/archive/refs/tags/v${GIT_VERSION}.tar.gz

# 3. Распаковка архива
tar -xzf v${GIT_VERSION}.tar.gz
cd git-${GIT_VERSION}

# 4. Компиляция (используем все ядра процессора, флаг -j$(nproc))
make prefix=/usr/local all -j$(nproc)

# 5. Установка собранного Git в систему (в /usr/local/bin)
sudo make prefix=/usr/local install

# 6. Проверка установленной версии
git --version

# 7. (Опционально) Очистка временных файлов
cd /tmp
rm -rf git-${GIT_VERSION} v${GIT_VERSION}.tar.gz
```

### Пояснения:

- Пакет `zlib1g-dev` исправляет ошибку zlib.h not found,
- `libcurl4-openssl-dev` и `libssl-dev` необходимы для работы с HTTPS и криптографии,
- `libexpat1-dev` — для поддержки формата патчей,
- `gettext` — для интернационализации сообщений Git.
- Установка выполняется в `/usr/local`, что не затрагивает системный Git (если он был установлен). Новый Git будет приоритетнее, если `/usr/local/bin` идёт в PATH перед `/usr/bin` (обычно так и есть).


Если вы хотите установить другую версию, измените GIT_VERSION="2.47.0" на нужную (например, 2.48.1). Актуальные версии можно найти на официальном сайте.
