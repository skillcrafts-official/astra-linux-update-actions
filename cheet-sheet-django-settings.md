## Шпаргалка по настройке Django SETTINGS.PY
Ориентирована на production-ready конфигурацию, но с пояснениями для разработки.
### 1. Настройки базы данных
```python
# Пример для PostgreSQL (рекомендуется)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'mydb',
        'USER': 'mydbuser',
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': 'localhost',          # или IP / имя контейнера
        'PORT': '5432',
        'CONN_MAX_AGE': 600,         # постоянное соединение (сек)
        'OPTIONS': {
            'sslmode': 'require',    # если требуется SSL
        },
    }
}

# Для SQLite (разработка)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Для MySQL
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mydb',
        'USER': 'root',
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'OPTIONS': {
            'charset': 'utf8mb4',
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'",
        },
    }
}
```
**Важно**: для PostgreSQL и MySQL требуется установить драйвер (`psycopg2-binary`, `mysqlclient`).

### 2. Настройки static (статические файлы: CSS, JS, изображения для вёрстки)
```python
# URL, по которому статика будет отдаваться в браузере
STATIC_URL = '/static/'

# В production (когда DEBUG=False) – путь, куда collectstatic соберёт все файлы
STATIC_ROOT = BASE_DIR / 'staticfiles'

# Дополнительные папки, где Django будет искать статику помимо static/ внутри приложений
STATICFILES_DIRS = [
    BASE_DIR / 'static',            # общая статика уровня проекта
    # '/var/www/static/',
]

# Хранилище (по умолчанию FileSystemStorage). Для S3/CDN потребуется django-storages.
# STATICFILES_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
```
Команды:
- `python manage.py collectstatic` – сбор всей статики в `STATIC_ROOT`.
- В разработке статика раздаётся автоматически при `DEBUG=True`.

### 3. Настройки media (загружаемые пользователем файлы)



