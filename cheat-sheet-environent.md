## Шпаргалка по CRUD-операциям с конфигами с соблюдением иерархии приоритетов:

> 1. Переменные окружения (ОС/CLI)
> 2. Файл .env (общие настройки)
> 3. Значение по умолчанию прямо в коде

Все действия выполняются в Python с помощью `os`, `python-dotenv`.

### Read — чтение с учётом приоритетов

```python
import os
from dotenv import load_dotenv, find_dotenv

# Однократная загрузка .env (обычно в точке входа)
load_dotenv(find_dotenv())

# Классический паттерн: env → .env → default
DATABASE_URL = os.environ.get("DATABASE_URL") or "postgresql://localhost/mydb"
# или чуть длиннее, но явно:
DATABASE_URL = os.environ.get("DATABASE_URL", os.environ.get("DATABASE_URL", "postgresql://localhost/mydb"))
# На практике первый вариант достаточен, если .env уже загружен в os.environ.

# Типизированное чтение с дефолтом
DEBUG = os.environ.get("DEBUG", "false").lower() == "true"   # default False
PORT  = int(os.environ.get("PORT", "8000"))
```
#### Что происходит под капотом:

- Если переменная задана в системе или через CLI – она уже в os.environ.
- Иначе load_dotenv() добавляет в os.environ переменные из .env.
- Если ни там, ни там нет – срабатывает дефолт.

### Create / Update — создание и изменение
#### В текущем процессе (в памяти)
Изменения видны только самому процессу и его потомкам, не сохраняются в `.env` и не влияют на другие сессии.
```python
import os

# Создать / обновить переменную
os.environ["API_KEY"] = "new-secret-123"

# Проверить
print(os.environ.get("API_KEY"))  # 'new-secret-123'
```
#### Запись в файл .env (постоянное хранение)
Используем утилиты `python-dotenv`:
```python
from dotenv import set_key, find_dotenv

env_path = find_dotenv()  # или явно "путь/к/.env"

# Записать (создаст или обновит)
set_key(env_path, "API_KEY", "new-secret-123")

# Можно добавить/обновить сразу несколько, но set_key принимает только пару.
# Для массового обновления проще переписать весь словарь (см. ниже).
```
После записи в файл нужно заново загрузить его в процесс, если хотим видеть изменения:
```python
from dotenv import load_dotenv
load_dotenv(env_path, override=True)
```
Массовое обновление `.env` через словарь:
```python
from dotenv import dotenv_values, set_key

env_path = find_dotenv()
current = dotenv_values(env_path)  # читает .env без загрузки в os.environ
current.update({"KEY1": "val1", "KEY2": "val2"})

with open(env_path, "w") as f:
    for k, v in current.items():
        f.write(f"{k}={v}\n")
```
### Delete — удаление
Из текущего процесса
```python
del os.environ["API_KEY"]  # выбросит KeyError, если ключа нет

# Безопасное удаление
os.environ.pop("API_KEY", None)
```
Из файла `.env`
```python
from dotenv import unset_key, find_dotenv

env_path = find_dotenv()
unset_key(env_path, "API_KEY")  # удаляет запись из файла
```
После удаления тоже можно перезагрузить `load_dotenv(override=True)` для синхронизации с процессом.

### Типовые сценарии (CRUD-циклы)
#### 1. Установить значение поверх дефолта, записать в .env и перезагрузить
```python
import os
from dotenv import load_dotenv, set_key, find_dotenv

env_path = find_dotenv()
# Установка в процесс
os.environ["TIMEOUT"] = "45"
# Сохраняем в файл
set_key(env_path, "TIMEOUT", "45")
# (процесс уже видит, но для порядка)
load_dotenv(env_path, override=True)
```
#### 2. Прочитать с учётом всех уровней и вернуть значение + источник
```python
def get_config(key, default):
    # 1. Окружение
    if key in os.environ:
        return os.environ[key], "env"
    # 2. .env (уже в os.environ после load_dotenv, поэтому не отличим,
    #    но можем отдельно проверить dotenv_values)
    from dotenv import dotenv_values, find_dotenv
    env_vals = dotenv_values(find_dotenv())
    if key in env_vals:
        return env_vals[key], ".env"
    # 3. Дефолт
    return default, "default"
```
#### 3. Сбросить настройку к дефолту (удалить из окружения и .env)
```python
def reset_to_default(key):
    os.environ.pop(key, None)                # удалить из процесса
    from dotenv import unset_key, find_dotenv
    env_path = find_dotenv()
    unset_key(env_path, key)                 # удалить из файла
    # Теперь при следующем get_config сработает default
```
### ⚠️ Важные нюансы
- `os.environ` живёт только в рамках процесса. Изменения не видны родительскому процессу, не сохраняются в файлы. Для постоянного хранения – только `.env`.
- Не путайте `load_dotenv()` с чтением файла. По умолчанию `load_dotenv()` не перезаписывает уже установленные переменные окружения (т.е. системные имеют приоритет). Чтобы `.env` мог переопределить, используйте `override=True`.
- Безопасность: Не коммитьте `.env` с секретами. Для работы с секретами лучше использовать системные переменные в продакшене.
- Порядок загрузки: `load_dotenv(find_dotenv())` должен быть вызван до первого чтения конфигов, иначе значения из `.env` не попадут в `os.environ`.
- Функции `set_key` / `unset_key` модифицируют именно файл `.env`. Убедитесь, что файл не заблокирован и доступен для записи.
