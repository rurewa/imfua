## Интерактивная работа с MySQL в изолированном окружении Docker № 1

InnoDB - движок для MySQL

Задачи:
- запустить «чистый» контейнер
- подключиться к нему
- выполнять основные операции с базами данных
- сделать дамп базы данных

### 🚀 Шаг 1: Запуск контейнера MySQL

Подготовка Docker для выполнения этого учебного проекта:

1. Получить список текущих контейнеров
```shell
docker ps -a
```
2. Остановить все запущенные контейнеры:
```shell
docker stop $(docker ps -q)
```
3. Удалить все остановленные или не запущенные (Exited, Created) контейнеры:
```shell
docker container prune
```
4. Очистить все ненужные тома:
```shell
docker volume prune -a
```

Создание каталога проекта

- Создайте отдельный каталог `my-mysql` и перейдите в него (`cd my-mysql`).
- Создайте в каталоге `my-mysql` файл `my.cnf` с содержимым (на забудьте перейти в папку `my-mysql`):
```shell
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
general_log = 1
general_log_file = /var/lib/mysql/general.log
log_output = FILE

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
```
В каталоге `my-mysql` создайте папку `logs` для логов
```shell
mkdir -p logs
```
создать файл `general.log` для ведения лога (журнала) действий в БД
```shell
touch general.log
```
Контейнер с официального образа **MySQL**:
```bash
docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:latest
```
Разберем, что делает эта команда:
*   `--name my-mysql`: Присваивает контейнеру понятное имя `my-mysql`, чтобы не использовать его **ID**.
*   `-e MYSQL_ROOT_PASSWORD=123`: Устанавливает пароль для пользователя root. В учебном примере — `123`, в реальных проектах используйте надёжный пароль.
*   `-d`: Запускает контейнер в фоновом режиме (detached mode).
*   `mysql:latest`: Указывает, что нужно использовать последнюю версию официального образа MySQL.

Чтобы убедиться, что контейнер успешно запущен, выполните:
```bash
docker ps
```
Вы должны увидеть свой контейнер **my-mysql** в списке работающих.

**Где "физически" зранятся файлы новой базы данных?**

Увидеть физические файлы базы в вашем Docker-контейнере (не обязательно):
```shell
docker exec -it my-mysql ls -la /var/lib/mysql
```
Вы увидите примерно такую картину:
```shell
drwxr-xr-x  mysql mysql  ibdata1
drwxr-xr-x  mysql mysql  ib_logfile0
drwxr-xr-x  mysql mysql  ib_logfile1
drwxr-xr-x  mysql mysql  #innodb_temp
drwxr-xr-x  mysql mysql  mysql/
drwxr-xr-x  mysql mysql  performance_schema/
drwxr-xr-x  mysql mysql  sys/
drwxr-xr-x  mysql mysql  sql_practice/
```

### 🔌 Шаг 2: Подключение к MySQL в интерактивном режиме

Теперь, когда контейнер работает, к нему нужно подключиться. Для этого используется команда `docker exec`, которая позволяет выполнить команду внутри работающего контейнера.

Выполните следующую команду, чтобы открыть интерактивную оболочку **MySQL**:
```bash
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

*   `-it`: Флаги `-i` (interactive) и `-t` (tty) вместе создают интерактивный терминал.
*   `mysql -uroot -p`: Запускает клиент MySQL от имени пользователя `root` и запрашивает пароль.

После ввода пароля вы увидите приглашение `mysql>`. Теперь вы находитесь в интерактивной оболочке и можете выполнять любые SQL-команды.

### 🛠️ Шаг 3: Основные операции с базами данных и таблицами

Внутри оболочки **MySQL** вы можете выполнять все стандартные SQL-команды.

Проверить кодировку сразу после подключения (не обязательно):
```sql
SHOW VARIABLES LIKE 'character_set%';
```
Если `character_set_client`, `character_set_connection` и `character_set_results` равны `utf8mb4` — всё в порядке.

**1. Просмотр существующих баз данных**
```sql
SHOW DATABASES;
```
Эта команда покажет список всех баз данных, включая системные (`information_schema`, `mysql`, `performance_schema`, `sys`). Системные БД трогать не нужно - они необходимы для работы самой СУБД.

**2. Создание новой базы данных и таблицы в ней**

Получить краткую справку по командам СУБД **MySQL**
```sql
help
```
Фильтр таблиц для лога
```sql
SELECT event_time, command_type, argument
FROM mysql.general_log
WHERE user_host LIKE 'root@%'
ORDER BY event_time DESC;
```
и
```sql
SELECT event_time, command_type, argument
FROM mysql.general_log
WHERE user_host LIKE 'root@%'
  AND argument NOT LIKE 'SET NAMES%'
  AND argument NOT LIKE 'SHOW VARIABLES%'
  AND argument NOT LIKE 'SELECT @@%'
  AND argument NOT LIKE 'SET SESSION sql_log_off%'  -- на всякий случай
ORDER BY event_time DESC;
```
и
```sql
SET SESSION sql_log_off = ON;
```
Создадим базу данных с именем `my_database` с таблицей `users`. Рекомендуется сразу указывать кодировку:
```sql
CREATE DATABASE my_database
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE my_database;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

INSERT INTO users (name, email) VALUES
  ('Иван Иванов', 'ivan@example.com'),
  ('Пётр Петров', 'petr@example.com'),
  ('Мария Сидорова', 'maria@example.com');

SELECT * FROM users;
```
Это инструкции, которые указывают СУБД: **«создай базу»**, **«создай таблицу»**, **«вставь строку»**

Получить существующие БД
```sql
SHOW DATABASES;
```

**3. Выбор базы данных для работы**
```sql
USE my_database;
```
Теперь все последующие команды будут применяться к этой базе данных!

Получить существующие таблицы
```sql
SHOW TABLES;
```

**4. Вставка данных**

Сначала получим текущие данные указанной таблицы
```sql
SELECT * FROM users;
```
затем добавим новые:
```sql
INSERT INTO users (name, email) VALUES ('Павел Иванов', 'pavel@example.com');
INSERT INTO users (name, email) VALUES ('Марк Аврелий', 'mark@example.com');
```
Проверка кодировки после вставки (опционально):
```sql
SELECT name, HEX(name) FROM users WHERE id = 1;
```

**5. Просмотр данных**
```sql
SELECT * FROM users;
```

**6. Обновление данных**
```sql
UPDATE users SET name = 'Иван Петров' WHERE id = 1;
```
просто меняем **id** в указанной записи

или
```sql
UPDATE users SET name = 'Vasya' WHERE id = 1;
```
Посмотреть все строки таблицы
```sql
SELECT * FROM users;
```
Посмотреть одну конкретную запись (строку)
```sql
SELECT * FROM users WHERE id = 1;
```
Получить указанные поля
```sql
SELECT name, email FROM users;
```
или получить указанное поле
```sql
SELECT name FROM users;
```
Посмотреть только нужные поля (столбцы)
```sql
SELECT name, email FROM users WHERE id = 1;
```
Удобный вертикальный вывод
```sql
SELECT * FROM users WHERE id = 1\G
```
`\G` заменяет точку с запятой и не требует `;` в конце

**7. Удаление данных**

Сначала получим содержимое таблицы для последующего сравнения
```sql
SELECT * FROM users;
```
затем удалим указанную запись
```sql
DELETE FROM users WHERE id = 2;
```
Если удалить `WHERE`, удалятся все строки

**8. Удаление таблицы**
```sql
DROP TABLE users;
```
Получить существующие таблицы
```sql
SHOW TABLES;
```

**9. Удаление базы данных**
```sql
DROP DATABASE my_database;
```
Просмотр существующих баз данных
```sql
SHOW DATABASES;
```
```sql
exit
```
или
```sql
quit
```
ещё
```sql
\q
```

### 🧹 Шаг 4: Управление Docker-контейнером БД

**Остановка контейнера:**
```bash
docker stop my-mysql
```
**Запуск ранее остановленного контейнера:**
```bash
docker start my-mysql
```
**Просмотр лога контейнера (может пригодиться для отладки):**
```bash
docker logs my-mysql
```
**Просмотр логов контейнера с перенаправлением в файл**
```shell
docker logs my-mysql > log.txt
```
Если нужно пересоздать контейнер с сохранением данных:
```shell
docker stop my-mysql
docker rm my-mysql
docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:latest
```

***

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!