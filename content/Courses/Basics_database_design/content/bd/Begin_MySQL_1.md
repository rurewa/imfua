# Интерактивная работа с MySQL в изолированном окружении Docker № 1

**Задачи:**
- запустить «чистый» контейнер
- подключиться к нему
- выполнять основные операции с базами данных
- сделать дамп базы данных
- выгрузить лог базы данных

---

### 🚀 Шаг 1: Запуск контейнера MySQL

**Подготовка Docker для выполнения этого учебного проекта:**

> ⚠️ Команды ниже удаляют **все** неиспользуемые контейнеры и volumes на хосте. Если у вас есть другие важные контейнеры — пропустите этот блок и удалите только `my-mysql` и `mysql_data` точечно (см. ниже).

1. Получить список текущих контейнеров:
```shell
docker ps -a
```
2. Остановить и удалить **только учебный** контейнер (если он уже существует):
```shell
docker stop имя_контейнера
docker rm имя_контейнера
```
3. Удалить все учебные **volume**
```shell
docker volume prune -a
```

> Если вы уверены, что на хосте нет других важных контейнеров и volumes, можно использовать более широкие команды:
```shell
docker stop $(docker ps -q)
docker container prune
docker volume prune -a
```
> Но **будьте осторожны** — они удаляют всё неиспользуемое, а не только учебный проект.

**Создание каталога проекта:**

- Создайте отдельный каталог `my-mysql` и перейдите в него (`cd my-mysql`).
- Создайте в каталоге `my-mysql` файл `my.cnf` с содержимым (не забудьте перейти в папку `my-mysql`):
```ini
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
Параметры `general_log = 1` и `general_log_file = /var/lib/mysql/general.log` нужны для того, чтобы MySQL вела файл журнала записей (лог).

Контейнер с официального образа **MySQL**:

Power Shell
```shell
docker run --name my-mysql -e MYSQL_ROOT_PASSWORD=123 -e LANG=C.UTF-8 -v mysql_data:/var/lib/mysql -v ${PWD}/my.cnf:/etc/mysql/conf.d/my.cnf -d mysql:8.0
```
Git-Bash:
```bash
docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:8.0
```
Разберём, что делает эта команда:
*   `--name my-mysql`: присваивает контейнеру понятное имя `my-mysql`, чтобы не использовать его **ID**.
*   `-e MYSQL_ROOT_PASSWORD=123`: устанавливает пароль для пользователя `root`. В учебном примере — `123`, в реальных проектах используйте надёжный пароль.
*   `-e LANG=C.UTF-8`: задаёт UTF-8 локаль внутри контейнера (нужна для корректной работы с кириллицей).
*   `-d`: запускает контейнер в фоновом режиме (detached mode).
*   `mysql:8.0`: **LTS-версия** официального образа MySQL. Тег `latest` может указывать на инновационную ветку (например, 26.x), которая не предназначена для стабильной учебной работы.

Чтобы убедиться, что контейнер успешно запущен, выполните:
```bash
docker ps
```
Вы должны увидеть свой контейнер **my-mysql** в списке работающих.

**Где «физически» хранятся файлы новой базы данных?**

Увидеть физические файлы базы в вашем Docker-контейнере (не обязательно):

PowerShell
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
```
> После создания базы `my_database` (см. Шаг 3) в этом каталоге появится подкаталог `my_database/` с файлами таблиц.

**InnoDB** — движок для MySQL

---

### 🔌 Шаг 2: Подключение к MySQL в интерактивном режиме

Теперь, когда контейнер работает, к нему нужно подключиться. Для этого используется команда `docker exec`, которая позволяет выполнить команду внутри работающего контейнера.

Выполните следующую команду, чтобы открыть интерактивную оболочку **MySQL** (Git-Bash, Power Shell):
```bash
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

*   `-it`: флаги `-i` (interactive) и `-t` (tty) вместе создают интерактивный терминал.
*   `mysql -uroot -p`: запускает клиент MySQL от имени пользователя `root` и запрашивает пароль.

После ввода пароля вы увидите приглашение `mysql>`. Теперь вы находитесь в интерактивной оболочке и можете выполнять любые SQL-команды.

---

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
Эта команда покажет список всех баз данных, включая системные (`information_schema`, `mysql`, `performance_schema`, `sys`). Системные БД трогать не нужно — они необходимы для работы самой СУБД.

**2. Создание новой базы данных и таблицы в ней**

Получить краткую справку по командам клиента `mysql` (не SQL):
```sql
help
```
> `help` — это команда клиента `mysql`, а не SQL. Она показывает справку по клиентским командам.

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
Это инструкции, которые указывают СУБД: **«создай базу»**, **«создай таблицу»**, **«вставь строку»**.

Получить существующие БД:
```sql
SHOW DATABASES;
```

**3. Выбор базы данных для работы**
```sql
USE my_database;
```
Теперь все последующие команды будут применяться к этой базе данных.

Получить существующие таблицы:
```sql
SHOW TABLES;
```

**4. Вставка данных**

Сначала получим текущие данные указанной таблицы:
```sql
SELECT * FROM users;
```
Затем добавим новые:
```sql
INSERT INTO users (name, email) VALUES ('Павел Иванов', 'pavel@example.com');
INSERT INTO users (name, email) VALUES ('Марк Аврелий', 'mark@example.com');
```
Проверка кодировки после вставки (опционально):
```sql
SELECT name, HEX(name) FROM users WHERE id = 1;
```
> `HEX(name)` покажет байты строки в hex. Для `utf8mb4` кириллица занимает 2 байта на символ — это нормально.

**5. Просмотр данных**
```sql
SELECT * FROM users;
```

**6. Обновление данных**
```sql
UPDATE users SET name = 'Иван Петров' WHERE id = 1;
```
Просто меняем **name** в указанной записи (по `id`).

Или:
```sql
UPDATE users SET name = 'Vasya' WHERE id = 1;
```
Посмотреть все строки таблицы:
```sql
SELECT * FROM users;
```
Посмотреть одну конкретную запись (строку):
```sql
SELECT * FROM users WHERE id = 1;
```
Получить указанные поля:
```sql
SELECT name, email FROM users;
```
Или получить одно поле:
```sql
SELECT name FROM users;
```
Посмотреть только нужные поля (столбцы) для конкретной записи:
```sql
SELECT name, email FROM users WHERE id = 1;
```
Удобный вертикальный вывод:
```sql
SELECT * FROM users WHERE id = 1\G
```
`\G` заменяет точку с запятой и не требует `;` в конце.

**7. Создание дампа БД (экспорт в `.sql`-файл на хост для последующего восстановления):**

Выйти из клиента MySQL (на всякий случай):
```sql
\q
```
Делаем дамп - рекомендуемый вариант, без GTID и с согласованным снимком:

Power Shell
```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot `
  --set-gtid-purged=OFF `
  --single-transaction `
  my_database > dump_clean.sql
```
Git-Bash
```shell
MSYS_NO_PATHCONV=1 docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot \
  --set-gtid-purged=OFF \
  --single-transaction \
  my_database > dump_clean.sql
```
Linux WSL
```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot \
  --set-gtid-purged=OFF \
  --single-transaction \
  my_database > dump_clean.sql
```
После этого в папке `my-mysql` появится текстовый файл `dump.sql` (или `dump_clean.sql`), содержимое которого можно увидеть командой:

Git-Bash
```shell
cat dump.sql
```
`.sql`-файл является текстовым представлением вашей БД (SQL-скрипт). Его можно хранить как резервную копию и использовать для восстановления.

**8. Удаление данных**

Снова войдём в контейнер (Git-Bash, Power Shell, Linux WSL):
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Переключимся на нужную нам БД
```sql
USE my_database;
```
Теперь все последующие команды будут применяться к этой базе данных.

Получить существующие таблицы:
```sql
SHOW TABLES;
```
Сначала получим содержимое таблицы для последующего сравнения:
```sql
SELECT * FROM users;
```
Затем удалим указанную запись:
```sql
DELETE FROM users WHERE id = 2;
```
⚠️ Если удалить `WHERE`, удалятся **все** строки.

**9. Удаление таблицы**
```sql
DROP TABLE users;
```
Получить существующие таблицы:
```sql
SHOW TABLES;
```

**10. Удаление базы данных**
```sql
DROP DATABASE my_database;
```
Просмотр существующих баз данных:
```sql
SHOW DATABASES;
```
Выйти из клиента MySQL:
```sql
\q
```
> `exit` и `quit` — синонимы `\q`, работают так же.

---

### 📝 Шаг 4: Создание лога

Все команды записываются в файл `general.log` внутри контейнера (при `log_output = FILE` в `my.cnf`).

Проверить, что файл существует:

PowerShell, Linux WSL
```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```
> Если файл не создан — проверьте, что в `my.cnf` стоит `log_output = FILE` и `general_log = 1`, и контейнер перезапущен (`docker restart my-mysql`).

Заглянуть в лог:

PowerShell, Linux WSL
```shell
docker exec my-mysql cat /var/lib/mysql/general.log
```
Забрать лог на хост:

PowerShell
```shell
docker exec my-mysql cat /var/lib/mysql/general.log `
  | Where-Object { $_ -cmatch "Query" -and $_ -cnotmatch "SET NAMES|SET autocommit|SET sql_mode|SET character_set_results|SHOW VARIABLES|SHOW STATUS|SHOW WARNINGS|SELECT @@|SELECT DATABASE\(\)|time_zone" } `
  > my_actions.log
```
Git-Bash
```shell
docker exec my-mysql cat /var/lib/mysql/general.log \
  | grep "Query" \
  | grep -v -E "SET NAMES|SET autocommit|SET sql_mode|SET character_set_results|SHOW VARIABLES|SHOW STATUS|SHOW WARNINGS|SELECT @@|SELECT DATABASE\(\)|time_zone" \
  > my_actions.log
```
Очистка лога внутри контейнера (PowerShell, Linux WSL):
```shell
docker exec my-mysql sh -c ': > /var/lib/mysql/general.log'
```

---

### 🧹 Шаг 5: Управление Docker-контейнером БД

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
**Просмотр логов контейнера с перенаправлением в файл (Git-Bash, PowerShell, Linux WSL):**
```shell
docker logs my-mysql > log.txt
```

Результат оформить в виде отдельного `README.md` и дампом и логом (лог-файл должен содержать минимум служебных записей) в этой же папке

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!