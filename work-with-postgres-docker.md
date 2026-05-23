# Подключение к базе данных в Docker-контейнере

## 1. Подключиться к удалённому серверу по SSH
```
#bash
ssh username@server-ip
```

## 2. Найти контейнер с базой данных
```
#bash
sudo find / -type f -name ".env" 2>/dev/null
docker ps | grep -E 'postgres|mysql|mariadb|mongo'
```
Или посмотреть все запущенные контейнеры:
```
#bash

docker ps
```
Запомните CONTAINER ID или NAME нужного контейнера.
## 3. Подключиться к контейнеру и войти в СУБД
```
#Для PostgreSQL
#bash
docker exec -it <container_name_or_id> /bin/bash
docker exec -it <container_id> psql -U <username> -d <database_name>

#Пример:
#bash

docker exec -it postgres_db psql -U postgres -D mydb

#Для MySQL / MariaDB
#bash

docker exec -it <container_id> mysql -u <username> -p

#После нажатия Enter введите пароль.
#Для MongoDB
#bash

docker exec -it <container_id> mongosh
```
## 4. Если база запущена через docker-compose
```
#bash

docker-compose exec <service_name> psql -U <username>   # для PostgreSQL
docker-compose exec <service_name> mysql -u root -p     # для MySQL
```
(Сначала перейдите в каталог с docker-compose.yml)
## 5. Полезные команды для дальнейшей работы
```
    Посмотреть логи контейнера:
    docker logs -f <container_id>

    Перезапустить контейнер:
    docker restart <container_id>

    Остановить контейнер:
    docker stop <container_id>

    Зайти в контейнер в обычный shell (без СУБД):
    docker exec -it <container_id> bash (или sh)
```
### Примечание: Замените <container_id>, <username>, <database_name> на свои значения.

text


---

### 📝 Краткий алгоритм подключения к БД (текстом):
1. `ssh user@server`
2. `docker ps` → найти ID контейнера с БД
3. `docker exec -it <ID> psql -U postgres` (или mysql, mongosh)
4. Работайте с базой.

Если нужен именно выход на консоль СУБД без предварительного входа в контейнер — используйте команду из п.3. Все шаги уже включены в инструкцию выше.
