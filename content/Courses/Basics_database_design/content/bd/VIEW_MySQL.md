# Руководство по представлениям (VIEW) в MySQL

Представление — это **сохранённый SQL-запрос, который выглядит как таблица**. Вы обращаетесь к нему по имени, как к обычной таблице, но данных он не хранит — при каждом обращении запрос выполняется заново.

[Подробней о представлениях VIEW](/content/Courses/Basics_database_design/content/Docs/VIEW.md)


> **Задание выполнять в VS Code - Git-Bash!**

> **Работу можно выполнять в созданном ранее каталоге `my-mysql`, в котором уже есть файл `my.cnf`**

---

## 🔹 0. Настройка `my.cnf`

Замените содержимое файла `my.cnf` на это:

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
Переносы строк в `my.cnf`. Если файл был создан в **Блокноте** Windows, в нём могут оказаться **CRLF**-переносы. **MySQL** обычно это терпит, но если контейнер падает при старте — первым делом проверьте кодировку и переносы (лучше сохранить файл в **UTF-8** без BOM и с LF). Или создавать этот проект в Ubuntu WSL - в линуксах с этим проблем нет.

**Если контейнер `my-mysql` уже создан** — перезапустите его, чтобы настройки применились:

```shell
docker restart my-mysql
```

**Если контейнера нет** — переходите к разделу 1, где описано создание с нуля.

---

## 🔹 1. Подготовка: создание контейнера с нуля

Если вы начинаете «с чистого листа» — выполните эти шаги. Если контейнер уже есть и вы обновили `my.cnf` — переходите к разделу 2.

### 1.1. Получить список текущих контейнеров

```shell
docker ps -a
```

### 1.2. Остановить и удалить **только учебный** контейнер

```shell
docker stop my-mysql
docker rm my-mysql
```

### 1.3. Удалить volume учебных проектов

```shell
docker volume prune -a
```
> ⚠️ * — эта команда удаляет **все** неиспользуемые volumes на хосте, а не только `mysql_data`. Если у вас есть другие важные контейнеры и volumes — они пострадают.

### 1.4. Создать новый контейнер с MySQL

Git-Bash
```shell
docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -e MYSQL_INITDB_SKIP_TZINFO=1 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:8.0
```
Windows PowerShell
```shell
docker run --name my-mysql `
  -e MYSQL_ROOT_PASSWORD=123 `
  -e LANG=C.UTF-8 `
  -e MYSQL_INITDB_SKIP_TZINFO=1 `
  -v mysql_data:/var/lib/mysql `
  -v ${PWD}/my.cnf:/etc/mysql/conf.d/my.cnf `
  -d mysql:8.0
```
Разберём опции:

| Опция | Зачем |
|-------|-------|
| `--name my-mysql` | Понятное имя контейнера |
| `-e MYSQL_ROOT_PASSWORD=123` | Пароль пользователя `root` (в учебных целях — `123`) |
| `-e LANG=C.UTF-8` | UTF-8 локаль для корректной работы с кириллицей |
| `-e MYSQL_INITDB_SKIP_TZINFO=1` | Отключает автоматическую загрузку данных о часовых поясах. Ускоряет первый запуск и не засоряет лог тысячами `INSERT INTO time_zone_*` |
| `-v mysql_data:/var/lib/mysql` | Volume для хранения данных — сохраняются между перезапусками |
| `-v $(pwd)/my.cnf:...` | Монтирует ваш `my.cnf` с настройками кодировки и логирования |
| `-d mysql:8.0` | LTS-версия MySQL, стабильная и предсказуемая |

### 1.5. Подключиться к MySQL

Убедиться, что контейнер с БД существует и запущен (Up)
```shell
docker ps -a
```
Перед началом работы (сразу после создания контейнера) очистить лог:
```shell
docker exec my-mysql sh -c ': > /var/lib/mysql/general.log'
```
Подключиться к БД в контейнере
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

> ⚠️ Если сразу после запуска контейнера вы получите ошибку `ERROR 2002 (HY000): Can't connect to local MySQL server through socket...`, подождите 10–20 секунд (контейнер ещё инициализируется) и попробуйте подключиться снова.

---

## 🔹 2. Создание базы данных и таблиц

Все команды выполняются **внутри клиента MySQL** (после приглашения `mysql>`).

### 2.1. Создать учебную БД

```sql
CREATE DATABASE sql_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

### 2.2. Переключиться на неё

```sql
USE sql_practice;
```

### 2.3. Создать таблицы `customers` и `orders`

```sql
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    city VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'new',
    FOREIGN KEY (customer_id) REFERENCES customers(id)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 2.4. Заполнить таблицы данными

```sql
INSERT INTO customers (name, email, city) VALUES
  ('Иван Иванов', 'ivan@example.com', 'Москва'),
  ('Пётр Петров', 'petr@example.com', 'Санкт-Петербург'),
  ('Мария Сидорова', 'maria@example.com', 'Москва'),
  ('Алексей Смирнов', 'alex@example.com', 'Казань'),
  ('Ольга Кузнецова', 'olga@example.com', 'Новосибирск');

INSERT INTO orders (customer_id, order_date, amount, status) VALUES
  (1, '2026-10-01', 1500.00, 'paid'),
  (1, '2026-10-05', 2300.50, 'paid'),
  (2, '2026-10-03', 800.00, 'new'),
  (3, '2026-10-07', 4200.00, 'paid'),
  (3, '2026-10-10', 1100.00, 'cancelled'),
  (4, '2026-10-12', 950.00, 'new');
```

### 2.5. Проверить

```sql
SHOW TABLES;
SELECT * FROM customers;
SELECT * FROM orders;
```
Должны быть две таблицы (`customers`, `orders`) и данные в них.

---

## 🔹 3. Зачем нужны представления
| Задача | Как помогает **VIEW** |
|--------|-------------------|
| Упростить сложные запросы | Пишете `SELECT * FROM my_view` вместо трёхэтажного `JOIN` |
| Скрыть структуру таблиц | Пользователь видит только нужные столбцы |
| Ограничить доступ | Даёте права на **VIEW**, а не на таблицу |
| Переиспользовать логику | Один раз определили — используете везде |
| Абстракция | При изменении таблиц приложение продолжает работать |


---

## 🔹 4. Простейшее представление

**Задача:** часто нужен список клиентов из Москвы.

Без VIEW:
```sql
SELECT * FROM customers WHERE city = 'Москва';
```
С VIEW:
```sql
CREATE VIEW moscow_customers AS
SELECT id, name, email
FROM customers
WHERE city = 'Москва';
```
Теперь обращайтесь к нему как к таблице:
```sql
SELECT * FROM moscow_customers;
```
**Результат:**

| id | name | email |
|----|------|-------|
| 1 | Иван Иванов | ivan@example.com |
| 3 | Мария Сидорова | maria@example.com |


**Ключевая мысль:** данные не скопированы. При каждом `SELECT * FROM moscow_customers` MySQL **заново выполняет** запрос `SELECT ... WHERE city = 'Москва'`. Если добавите нового клиента из Москвы в `customers` — он **сразу появится** в представлении.

---

## 🔹 5. VIEW с JOIN и агрегацией

**Задача:** сводка по клиентам — сколько заказов и на какую сумму.
```sql
CREATE VIEW customer_summary AS
SELECT c.id,
       c.name,
       c.city,
       COUNT(o.id)              AS order_count,
       COALESCE(SUM(o.amount), 0) AS total_amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name, c.city;
```
Использование:
```sql
SELECT * FROM customer_summary ORDER BY total_amount DESC;
```
**Результат:**

| id | name | city | order_count | total_amount |
|----|------|------|-------------|--------------|
| 3 | Мария Сидорова | Москва | 2 | 5300.00 |
| 1 | Иван Иванов | Москва | 2 | 3800.50 |
| 4 | Алексей Смирнов | Казань | 1 | 950.00 |
| 2 | Пётр Петров | Санкт-Петербург | 1 | 800.00 |
| 5 | Ольга Кузнецова | Новосибирск | 0 | 0.00 |


Теперь вместо сложного `JOIN` с `GROUP BY` достаточно одной строки:
```sql
SELECT name, total_amount FROM customer_summary WHERE total_amount > 2000;
```

---

## 🔹 6. VIEW с фильтрацией внутри

Можно комбинировать **VIEW** с дополнительным `WHERE` снаружи:
```sql
SELECT * FROM customer_summary WHERE city = 'Москва';
```
MySQL «завернёт» ваш `WHERE` внутрь запроса представления. Это работает, но **не всегда эффективно** — см. раздел 10.

---

## 🔹 7. Просмотр существующих VIEW

```sql
SHOW FULL TABLES WHERE Table_type = 'VIEW';
```
Или в общем списке таблиц:
```sql
SHOW TABLES;
```
**MySQL** покажет **VIEW** рядом с обычными таблицами — визуально их не отличить.

Посмотреть **определение** (SQL-запрос, лежащий в основе):
```sql
SHOW CREATE VIEW customer_summary\G
```

Вы увидите что-то вроде:

```
*************************** 1. row ***************************
                View: customer_summary
         Create View: CREATE ALGORITHM=UNDEFINED DEFINER=`root`@`%`
                      SQL SECURITY DEFINER VIEW `customer_summary` AS
                      select ... from customers c left join orders o ...
        character_set_client: utf8mb4
        collation_connection: utf8mb4_unicode_ci
```

---

## 🔹 8. Изменение и удаление VIEW

### Изменить существующее представление

Получить таблицу для сравнения:
```sql
SELECT * FROM moscow_customers;
```
И выполнить запрос изменения:
```sql
CREATE OR REPLACE VIEW moscow_customers AS
SELECT id, name, email, created_at
FROM customers
WHERE city = 'Москва';
```
`CREATE OR REPLACE` перезаписывает старое определение новым, не удаляя **VIEW**.

Получить результат:
```sql
SELECT * FROM moscow_customers;
```

### Удалить

```sql
DROP VIEW moscow_customers;
```
Если **VIEW** не существует — ошибка. Чтобы не падало:
```sql
DROP VIEW IF EXISTS moscow_customers;
```
Получить результат:
```sql
SHOW TABLES;
```

---

## 🔹 9. Обновляемые VIEW (UPDATE, INSERT, DELETE)

**Не все VIEW можно обновлять.** Представление обновляемо, если MySQL может однозначно сопоставить строку VIEW со строкой базовой таблицы.

### ✅ Обновляемое VIEW

```sql
CREATE VIEW active_customers AS
SELECT id, name, email, city
FROM customers
WHERE city IS NOT NULL;
```
Можно:
```sql
UPDATE active_customers SET city = 'Сочи' WHERE id = 2;
SELECT * FROM customers WHERE id = 2;  -- city действительно изменился
```
и
```sql
INSERT INTO active_customers (name, email, city)
VALUES ('Новый Клиент', 'new@example.com', 'Уфа');
```
и
```sql
DELETE FROM active_customers WHERE id = 6;
```

### ❌ Необновляемое VIEW

```sql
CREATE VIEW customer_summary AS
SELECT c.id, c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;
```
Попытка обновления:
```sql
UPDATE customer_summary SET name = 'X' WHERE id = 1;
```
**Ошибка:**
```
ERROR 1288 (HY000): The target table customer_summary of the UPDATE is not updatable
```
**Почему:** `COUNT(o.id)` — это агрегат, его нельзя «разложить обратно» на строки таблицы. MySQL не знает, какую строку `customers` менять.

**Правило:** VIEW с `GROUP BY`, `DISTINCT`, `UNION`, агрегатами, `JOIN` (часто) — **необновляемы**.

---

## 🔹 10. VIEW и производительность

**Важно понимать:** VIEW — это **не материализованное представление**. Он не хранит данные и не имеет индексов. При каждом обращении MySQL выполняет **весь запрос заново**.

### Что это значит на практике

```sql
SELECT * FROM customer_summary WHERE city = 'Москва';
```
MySQL **не** сначала считает всю сводку, а потом фильтрует. Он «вставляет» ваш `WHERE` внутрь запроса и пытается оптимизировать. Иногда получается хорошо, иногда — нет.

**Проверить план:**
```sql
EXPLAIN SELECT * FROM customer_summary WHERE city = 'Москва';
```
Если план плохой — можно:

1. Упростить **VIEW**.
2. Использовать прямые запросы.
3. В PostgreSQL — `MATERIALIZED VIEW`, который хранит данные и имеет индексы. **В MySQL такого нет.**

---

## 🔹 11. VIEW с проверкой опций (WITH CHECK OPTION)

Без WITH CHECK OPTION — строка вставляется, но через VIEW не видна
```sql
-- Пересоздаём VIEW с email (обязательный столбец)
CREATE OR REPLACE VIEW moscow_customers AS
SELECT id, name, email, city
FROM customers
WHERE city = 'Москва';
```
```sql
-- Вставляем клиента из Казани — с уникальным email
INSERT INTO moscow_customers (name, email, city)
VALUES ('Пётр Второй', 'petr2@example.com', 'Казань');
```
```sql
-- Проверяем: строка есть в customers, но не видна через VIEW
SELECT * FROM customers WHERE email = 'petr2@example.com';
```
```sql
SELECT * FROM moscow_customers;
```
С WITH CHECK OPTION — вставка запрещена
```sql
CREATE OR REPLACE VIEW moscow_customers AS
SELECT id, name, email, city
FROM customers
WHERE city = 'Москва'
WITH CHECK OPTION;

INSERT INTO moscow_customers (name, email, city)
VALUES ('Пётр Третий', 'petr3@example.com', 'Казань');
```
-- ERROR 1369 (HY000): CHECK OPTION failed 'sql_practice.moscow_customers'

Теперь MySQL заранее проверяет, что вставляемая строка соответствует WHERE city = 'Москва'. Раз Казань не подходит — вставка отклоняется.

В таблице customers остались лишние строки (Пётр Второй, возможно Пётр Третий). Удалите их перед следующими разделами:
```sql
DELETE FROM customers WHERE email IN ('petr2@example.com', 'petr3@example.com');
```
Или полностью вернуть исходное состояние:
```sql
DELETE FROM customers WHERE id > 5;
```

---

## 🔹 12. Права доступа через VIEW

Частый сценарий: скрыть от пользователя столбец `email`.

```sql
CREATE VIEW public_customers AS
SELECT id, name, city FROM customers;  -- без email

-- Создать пользователя (пример)
CREATE USER 'analyst'@'%' IDENTIFIED BY 'pass';
GRANT SELECT ON sql_practice.public_customers TO 'analyst'@'%';
-- Доступ к самой таблице customers не даём
```
Теперь `analyst` видит `public_customers` без email, но не может обратиться к `customers` напрямую.

---

## 🔹 13. Создание дампа

Выйти из БД:
```sql
\q
```
Делаем дамп без предупреждения о пароле в командной строке:
```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot sql_practice > dump.sql
```
После чего в папке `my-mysql` появится текстовый файл `dump.sql`, содержимое которого можно увидеть командой (Git-Bash):
```shell
cat dump.sql
```
`.sql` файл является текстовым представлением вашей БД (текстовый SQL-скрипт). Его можно хранить как резервную копию вашей БД, из которой потом её можно восстановить.

По умолчанию **VIEW** включаются в дамп. Когда вы выполняете стандартную команду `mysqldump`, утилита по умолчанию включает в выходной файл не только таблицы, но и определения всех представлений.

---

## 🔹 14. Создание лога действий пользователя

MySQL может записывать все команды, которые выполняются в базе. Это полезно для отладки и анализа: видно, какие запросы отправлял пользователь.

**Самый простой способ — файловый лог.** Не нужно создавать дополнительные таблицы или представления.

### 14.1. Проверить, что лог включён

В `my.cnf` уже прописаны `general_log = 1` и `log_output = FILE`, поэтому после перезапуска контейнера лог должен быть активен. Проверьте в клиенте MySQL:

Вернуться в контейнер:
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
и выполнить это:
```sql
SHOW VARIABLES LIKE 'general_log%';
SHOW VARIABLES LIKE 'log_output';
```
Должно быть **general_log = ON**, **log_output = FILE**.

Если по какой-то причине лог выключен — включите вручную:
```sql
SET GLOBAL general_log = 'ON';
SET GLOBAL log_output = 'FILE';
```

### 14.2. Где находится лог

Файл лога лежит внутри контейнера по пути `/var/lib/mysql/general.log`.

Проверить, что файл существует:

Покинуть контейнер:
```sql
\q
```
```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```
Заглянуть в него:
```shell
docker exec my-mysql cat /var/lib/mysql/general.log
```
Забрать лог на хост (Git-Bash):
```shell
docker cp my-mysql:/var/lib/mysql/general.log ./general.log
```

### 14.3. Отфильтровать только свои действия (не обязательно)

В логе много служебных записей (`Connect`, `Quit`, `SET NAMES`, `SHOW VARIABLES` и т.п.). Отфильтровать их можно одной командой на хосте (Git-Bash):
```shell
docker exec my-mysql cat /var/lib/mysql/general.log \
  | grep "Query" \
  | grep -v -E "SET NAMES|SET autocommit|SET sql_mode|SET character_set_results|SHOW VARIABLES|SHOW STATUS|SHOW WARNINGS|SELECT @@|SELECT DATABASE\(\)|time_zone" \
  > my_actions.log
```
Теперь в файле `my_actions.log` — только ваши осмысленные SQL-команды.

Проверить (Git-Bash):
```shell
cat my_actions.log
```

### 14.4. Очистить лог

Если хотите начать с чистого листа (не обязательно):
```shell
docker exec my-mysql sh -c ': > /var/lib/mysql/general.log'
```
Файл обнулится, но останется на месте. MySQL продолжит в него писать.

Результат оформить в виде отдельного `README.md` и дампом и логом (лог-файл должен содержать минимум служебных записей) в этой же папке

---

## 🔹 15. Шпаргалка по командам

| Действие | Команда |
|----------|---------|
| Создать | `CREATE VIEW name AS SELECT ...;` |
| Создать или заменить | `CREATE OR REPLACE VIEW name AS SELECT ...;` |
| Удалить | `DROP VIEW name;` |
| Удалить безопасно | `DROP VIEW IF EXISTS name;` |
| Показать все VIEW | `SHOW FULL TABLES WHERE Table_type = 'VIEW';` |
| Показать определение | `SHOW CREATE VIEW name\G` |
| Использовать | `SELECT * FROM name;` |
| Обновить через VIEW | `UPDATE name SET ... WHERE ...;` |
| Вставить через VIEW | `INSERT INTO name ... VALUES ...;` |
| Удалить через VIEW | `DELETE FROM name WHERE ...;` |

---

## 🔹 16. Типичные ошибки

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `View 'x' already exists` | VIEW с таким именем уже есть | `DROP VIEW x;` или `CREATE OR REPLACE` |
| `The target table x of the UPDATE is not updatable` | VIEW содержит `GROUP BY` / агрегаты / `UNION` | Не обновлять через VIEW |
| `CHECK OPTION failed` | Вставка нарушает `WHERE` в VIEW | Не вставлять или убрать `WITH CHECK OPTION` |
| `Unknown table 'x'` | VIEW не существует или не выбрана база | `SHOW FULL TABLES WHERE Table_type = 'VIEW';` |
| VIEW работает медленно | Сложный запрос внутри | `EXPLAIN`, упростить VIEW |
| `ERROR 2002 ... Can't connect ... socket` | Контейнер ещё инициализируется | Подождать 10–20 секунд и подключиться снова |

---

## 🔹 17. Полный пример «с нуля» (для наглядности - делать не обязательно)

```sql
USE sql_practice;

-- 1. Создать VIEW
CREATE OR REPLACE VIEW moscow_customers AS
SELECT id, name, email
FROM customers
WHERE city = 'Москва';

-- 2. Использовать
SELECT * FROM moscow_customers;

-- 3. Проверить, что данные «живые»
INSERT INTO customers (name, email, city)
VALUES ('Тест Тестов', 'test@example.com', 'Москва');

SELECT * FROM moscow_customers;  -- новый клиент уже виден

-- 4. Посмотреть определение
SHOW CREATE VIEW moscow_customers\G

-- 5. Удалить
DROP VIEW moscow_customers;

-- 6. Убедиться, что удалено
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- 7. Убрать тестового клиента
DELETE FROM customers WHERE email = 'test@example.com';
```

---

## 📌 Итог

| Что важно запомнить о VIEW |
|----------------------------|
| VIEW — это **сохранённый запрос**, а не таблица |
| Данные **не копируются**, запрос выполняется при каждом обращении |
| VIEW **автоматически** показывает изменения в базовых таблицах |
| VIEW может быть **обновляемым** или нет (зависит от запроса) |
| VIEW **не ускоряет** запросы — иногда замедляет |
| `WITH CHECK OPTION` защищает от «невидимых» вставок |
| VIEW — удобный инструмент **абстракции** и **контроля доступа** |

**Когда использовать VIEW:**
- Часто повторяющийся сложный запрос → сохранить как VIEW.
- Нужно скрыть часть столбцов от пользователя → VIEW + права.
- Хочется «стабилизировать» интерфейс для приложения → VIEW переживёт изменение таблиц.

**Когда не использовать:**
- Ради ускорения — VIEW не материализован.
- Для очень больших агрегатов с частыми запросами — лучше хранить результат в отдельной таблице и обновлять по расписанию.

---

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!