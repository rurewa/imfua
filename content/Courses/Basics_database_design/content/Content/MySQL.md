## Интерактивная работа с MySQL в изолированном окружении Docker. 

- запустить «чистый» контейнер
- подключиться к нему
- выполнять основные операции с базами данных

### 🚀 Шаг 1: Запуск контейнера MySQL

- Создайте отдельный каталог `my-mysql` и перейдите в него.
- Создайте в каталоге `my-mysql` файл `my.cnf` с содержимым:
```shell
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4
```
Контейнер с официального образа MySQL:
```bash
docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:latest
```

Разберем, что делает эта команда:
*   `--name my-mysql`: Присваивает контейнеру понятное имя `my-mysql`, чтобы не использовать его ID.
*   `-e MYSQL_ROOT_PASSWORD=123`: Устанавливает пароль для пользователя root. В учебном примере — 123, в реальных проектах используйте надёжный пароль **Обязательно замените `123` на свой надежный пароль!**
*   `-d`: Запускает контейнер в фоновом режиме (detached mode).
*   `mysql:latest`: Указывает, что нужно использовать последнюю версию официального образа MySQL.

Чтобы убедиться, что контейнер успешно запущен, выполните:
```bash
docker ps
```
Вы должны увидеть свой контейнер `my-mysql` в списке работающих.

### 🔌 Шаг 2: Подключение к MySQL в интерактивном режиме

Теперь, когда контейнер работает, к нему нужно подключиться. Для этого используется команда `docker exec`, которая позволяет выполнить команду внутри работающего контейнера.

Выполните следующую команду, чтобы открыть интерактивную оболочку MySQL:

```bash
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
*   `-it`: Флаги `-i` (interactive) и `-t` (tty) вместе создают интерактивный терминал.
*   `mysql -uroot -p`: Запускает клиент MySQL от имени пользователя `root` и запрашивает пароль.

После ввода пароля вы увидите приглашение `mysql>`. Теперь вы находитесь в интерактивной оболочке и можете выполнять любые SQL-команды.

### 🛠️ Шаг 3: Основные операции с базами данных и таблицами

Внутри оболочки MySQL вы можете выполнять все стандартные SQL-команды.

Проверить кодировку сразу после подключения:
```sql
SHOW VARIABLES LIKE 'character_set%';
```
Если `character_set_client`, `character_set_connection` и `character_set_results` равны `utf8mb4` — всё в порядке.

**1. Просмотр существующих баз данных**
```sql
SHOW DATABASES;
```
Эта команда покажет список всех баз данных, включая системные (`information_schema`, `mysql`, `performance_schema`, `sys`).

**2. Создание новой базы данных и таблицы в ней**
```sql
CREATE DATABASE my_database;
```
Создадим базу данных с именем `my_database`. Рекомендуется сразу указывать кодировку:
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

**3. Выбор базы данных для работы**
```sql
USE my_database;
```
Теперь все последующие команды будут применяться к этой базе данных.

Получить существующие таблицы
```sql
SHOW TABLES;
```

**4. Вставка данных**
```sql
INSERT INTO users (name, email) VALUES ('Павел Иванов', 'pavel@example.com');
INSERT INTO users (name, email) VALUES ('Марк Аврелий', 'mark@example.com');
```
Проверка кодировки после вставки:
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
или
```sql
UPDATE users SET name = 'Vasya' WHERE id = 1;
```

Посмотреть все строки таблицы
```sql
SELECT * FROM users;
```
Посмотреть одну конкретную строку
```sql
SELECT * FROM users WHERE id = 1;
```
Посмотреть только нужные столбцы
```sql
SELECT name, email FROM users WHERE id = 1;
```
Удобный вертикальный вывод
```sql
SELECT * FROM users WHERE id = 1\G
```
`\G` заменяет точку с запятой и не требует `;` в конце

**7. Удаление данных**
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
Просмотр существующих баз данных**
```sql
SHOW DATABASES;
```

### 🧹 Шаг 4: Управление контейнером

*   **Остановка контейнера:**
    ```bash
    docker stop my-mysql
    ```
*   **Запуск ранее остановленного контейнера:**
    ```bash
    docker start my-mysql
    ```
*   **Просмотр логов (может пригодиться для отладки):**
    ```bash
    docker logs my-mysql
    ```
*   **Полное удаление контейнера (вместе с volume, если он не нужен):**
    ```bash
    docker rm -v my-mysql
    ```
    Флаг `-v` также удалит связанный с контейнером volume. **Будьте осторожны: это приведет к безвозвратной потере данных.**

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


> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!
