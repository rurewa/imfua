## Практическое руководство по основам SQL

### 🔌 Подключение к контейнеру

Если созданный ранее контейнер **my-mysql** ещё не запущен, запустите его:
```bash
docker start my-mysql
```
Подключитесь к MySQL:
```bash
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

---

#### 🗄️ Шаг 1: Создание учебной базы данных

```sql
CREATE DATABASE sql_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE sql_practice;
```
Проверить, что база создана:
```sql
SHOW DATABASES;
```

---

#### 📋 Шаг 2: Создание таблиц

Создадим две связанные таблицы: `customers` (клиенты) и `orders` (заказы):
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
Получить существующие таблицы
```sql
SHOW TABLES;
```
Проверить структуру таблицы:
```sql
DESCRIBE customers;
DESCRIBE orders;
```
Команда `CREATE TABLE customers` Создаёт файл customers.ibd и т.п.
```sql
exit
```
Внутри каталога sql_practice/ можно увидеть файлы
```shell
docker exec -it my-mysql ls -la /var/lib/mysql/sql_practice
```
Вы увидите:
```shell
customers.ibd
orders.ibd
```

---

#### ✍️ Шаг 3: Вставка данных (INSERT)

Вернуться в базу данных
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Использовать указанную БД
```sql
USE sql_practice;
```
Показать таблицы
```sql
SHOW TABLES;
```
Показать содержимое таблиц:
```sql
SELECT * FROM customers;
SELECT * FROM orders;
```
и выполнить эту инструкцию
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
`customers` — клиенты, `paid` — оплачено

Проверить, что данные вставились:
```sql
SHOW TABLES;
```
и
```sql
SELECT * FROM customers;
SELECT * FROM orders;
```

---

## 🔍 Шаг 4: Выборка данных (SELECT)

##### 4.1. Все столбцы и все строки
```sql
SELECT * FROM customers;
```
##### 4.2. Только нужные столбцы
```sql
SELECT name, city FROM customers;
```
##### 4.3. Условие `WHERE` (ГДЕ)
```sql
SELECT * FROM customers WHERE city = 'Москва';
SELECT * FROM orders WHERE amount > 1000;
SELECT * FROM orders WHERE status = 'paid' AND amount > 1500;
SELECT * FROM orders WHERE status = 'new' OR status = 'cancelled';
```
`amount` — количество, сумма

##### 4.4. Сортировка `ORDER BY`
```sql
SELECT * FROM customers ORDER BY name;
SELECT * FROM orders ORDER BY amount DESC;
SELECT * FROM orders ORDER BY order_date DESC, amount ASC;
```
Сортировка заказов по сумме (от большей к меньшей)

##### 4.5. Ограничение количества строк `LIMIT`
```sql
SELECT * FROM orders ORDER BY amount DESC LIMIT 3;
```
##### 4.6. Поиск по шаблону `LIKE`
```sql
SELECT * FROM customers WHERE name LIKE 'И%';      -- начинается на И
SELECT * FROM customers WHERE email LIKE '%@example.com';
```
##### 4.7. Диапазон `BETWEEN`, список `IN`
```sql
SELECT * FROM orders WHERE amount BETWEEN 1000 AND 3000;
SELECT * FROM customers WHERE city IN ('Москва', 'Казань');
```
Оператор **IN** — это логический оператор, который проверяет, входит ли значение в заданный список. Он является сокращённой формой цепочки OR (ИЛИ, дизъюнкция).

Это то же самое, что:
```sql
SELECT * FROM customers WHERE city = 'Москва' OR city = 'Казань';
```
Результат одинаковый — обе строки вернут клиентов из Москвы и Казани. Но **IN** короче, читабельнее и удобнее, особенно если значений много.

##### 4.8. Пустые значения `IS NULL`
```sql
SELECT * FROM customers WHERE city IS NULL;
```

---

#### 📊 Шаг 5: Агрегатные функции

Всего заказов
```sql
SELECT COUNT(*) AS total_orders FROM orders;
```
Общая сумма
```sql
SELECT SUM(amount) AS total_sum FROM orders;
```
Среднее
```sql
SELECT AVG(amount) AS avg_amount FROM orders;
```
Минимальная и максимальная сумма
```sql
SELECT MIN(amount) AS min_amount, MAX(amount) AS max_amount FROM orders;
```

---

#### 📈 Шаг 6: Группировка `GROUP BY` и `HAVING`

##### 6.1. Количество заказов по каждому клиенту
```sql
SELECT customer_id, COUNT(*) AS order_count, SUM(amount) AS total_amount
FROM orders
GROUP BY customer_id;
```

##### 6.2. Соединение с именами клиентов
```sql
SELECT c.name, COUNT(o.id) AS order_count, SUM(o.amount) AS total_amount
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_amount DESC;
```
Алиасы c и o — см. раздел 7.1.

##### 6.3. Фильтрация групп HAVING
```sql
SELECT customer_id, SUM(amount) AS total_amount
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 2000;
```

---

#### 🔗 Шаг 7: Соединения **JOIN**

##### 7.1. INNER JOIN — только совпадающие строки
```sql
SELECT c.name, o.order_date, o.amount, o.status
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;
```
**INNER JOIN** и просто **JOIN** — это одно и то же. Слово **INNER** можно опускать.

Чтобы узнать имя клиента по его **id**, нужно соединить таблицы через **JOIN**

**JOIN** — это оператор, который соединяет строки из двух или более таблиц на основе общего столбца (обычно это первичный ключ одной таблицы и внешний ключ другой). Без JOIN работать с реляционными базами данных практически невозможно: данные намеренно разнесены по таблицам, чтобы избежать дублирования

Алиасы в SQL
```sql
FROM customers c
JOIN orders o ON c.id = o.customer_id;
```
Здесь:
- `customers c` — таблица **customers** получает псевдоним `c`
- `orders o` — таблица **orders** получает псевдоним `o`

Зачем нужны?

1. Краткость. Вместо `customers.name` пишете `c.name`. В коротком запросе это мелочь, но в запросе с 5 таблицами — существенная экономия.
2. Разрешение неоднозначности. В обеих таблицах есть столбец `id`. Без алиаса запрос:
```sql
SELECT id FROM customers JOIN orders ON ...;
```
вернёт ошибку:
```sql
ERROR 1052 (23000): Column 'id' in field list is ambiguous
```
т.к. MySQL не знает, какой `id` имеется в виду — из `customers` или из `orders`. С алиасом всё однозначно:
```sql
SELECT c.id, o.id FROM customers c JOIN orders o ON c.id = o.customer_id;
```
[Подробней про алиасы в SQL тут >>>](/content/Courses/Basics_database_design/content/Docs/Aliases.md)


##### 7.2. LEFT JOIN — все клиенты, даже без заказов
```sql
SELECT c.name, o.id AS order_id, o.amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
```
**LEFT JOIN** (левое внешнее соединение) — возвращает все строки из левой таблицы, даже если нет совпадений в правой. Для несовпавших строк правая таблица даёт **NULL**.

Клиенты без заказов будут иметь `NULL` в полях заказа.

##### 7.3. RIGHT JOIN — все заказы, даже если клиент удалён (редко используется)
```sql
SELECT c.name, o.id AS order_id
FROM customers c
RIGHT JOIN orders o ON c.id = o.customer_id;
```
**RIGHT JOIN** (правое внешнее соединение) — возвращает все строки из правой таблицы, даже если нет совпадений в левой.

Результат: все строки из правой таблицы (orders) + совпадения из левой (customers). Если у заказа нет клиента — в поле c.name будет NULL.

На практике: RIGHT JOIN используется редко. Почти всегда его можно переписать как LEFT JOIN, поменяв таблицы местами:
```sql
-- Эквивалент RIGHT JOIN
SELECT c.name, o.id AS order_id
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
```

##### 7.4. JOIN и GROUP BY — соединение с агрегацией

Соединим таблицы и посчитаем агрегаты:
```sql
SELECT c.name,
       COUNT(o.id) AS order_count,
       SUM(o.amount) AS total_amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_amount DESC;
```
Чтобы вместо NULL был 0:
```sql
SELECT c.name,
       COUNT(o.id) AS order_count,
       COALESCE(SUM(o.amount), 0) AS total_amount
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY total_amount DESC;
```

---

#### 🧩 Шаг 8: Подзапросы и **UNION**

##### 8.1. Подзапросы (Subqueries)

Подзапрос — это `SELECT`-запрос, вложенный внутрь другого SQL-запроса. Подзапрос выполняется первым, а его результат используется внешним запросом.

Подзапрос в WHERE (с IN)
```sql
SELECT name, city
FROM customers
WHERE id IN (
    SELECT customer_id
    FROM orders
    WHERE amount > 1000
);
```
Вернёт клиентов, у которых есть хотя бы один заказ на сумму больше 1000.

Подзапрос в WHERE (со сравнением)
```sql
SELECT name, city
FROM customers
WHERE id = (
    SELECT customer_id
    FROM orders
    ORDER BY amount DESC
    LIMIT 1
);
```
Вернёт клиента с самым крупным заказом.

Подзапрос в FROM (производная таблица)
```sql
SELECT c.name, o.total
FROM customers c
JOIN (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
) o ON c.id = o.customer_id
WHERE o.total > 2000;
```
Подзапрос в `FROM` формирует временную таблицу с агрегатами, которую затем соединяем с `customers`.

Подзапрос в SELECT (скалярный)
```sql
SELECT c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS order_count
FROM customers c;
```
Для каждого клиента считается количество его заказов.

EXISTS и NOT EXISTS
```sql
-- Клиенты, у которых есть хотя бы один заказ
SELECT name FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Клиенты без заказов
SELECT name FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```
`EXISTS` часто работает быстрее, чем `IN`, особенно если внешняя таблица большая, а внутренняя — маленькая.

#### 8.2. UNION и UNION ALL

`UNION` объединяет результаты двух `SELECT`-запросов в один набор строк.

UNION (удаляет дубликаты)
```sql
SELECT name, city FROM customers WHERE city = 'Москва'
UNION
SELECT name, city FROM customers WHERE city = 'Казань';
```
Вернёт клиентов из Москвы и Казани, при этом дубликаты строк удаляются.

UNION ALL (сохраняет дубликаты, работает быстрее)
```sql
SELECT city FROM customers
UNION ALL
SELECT city FROM customers;
```
Вернёт все города, включая повторы.

Требования к UNION
- Одинаковое количество столбцов в обоих запросах.
- Совместимые типы данных.
- Имена столбцов берутся из первого запроса.

#### 8.3. Сравнение JOIN и подзапросов

| Критерий | JOIN | Подзапрос |
|----------|------|-----------|
| Читабельность | Лучше для соединения данных | Лучше для фильтрации |
| Производительность | Обычно быстрее | Может быть медленнее |
| Когда использовать | Нужны данные из обеих таблиц | Нужна только фильтрация |
| **EXISTS** | — | Часто быстрее, чем **IN** |

---

#### ✏️ Шаг 9: Обновление данных (UPDATE)

```sql
UPDATE customers SET city = 'Екатеринбург' WHERE id = 4;
UPDATE orders SET status = 'paid' WHERE id = 3;
```
Проверить:

```sql
SELECT * FROM customers WHERE id = 4;
SELECT * FROM orders WHERE id = 3;
```

⚠️ **Важно:** всегда указывайте `WHERE`, иначе обновятся все строки!

#### 8.4. Создание лога и дампа

Как проверить, что лог пишется
```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```
Должен быть примерно такой вывод:
```shell
-rw-r----- 1 mysql mysql 3265 Nov  3 08:46 /var/lib/mysql/general.log
```
заглянуть в него:
```shell
docker exec my-mysql cat /var/lib/mysql/general.log
```
забрать лог на хост
```shell
docker cp my-mysql:/var/lib/mysql/general.log ./general.log
```
Очистка лога внутри контейнера
```shell
docker exec my-mysql sh -c ': > /var/lib/mysql/general.log'
```

Создание дампа БД (Экспорт в `.sql` файл)

Выйти из БД
```sql
\q
```
Делаем дамп:
```shell
docker exec my-mysql mysqldump -uroot -p123 sql_practice > dump.sql
```
или
```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot sql_practice > dump.sql
```
или
```shell
docker exec my-mysql mysqldump -uroot -p123 --set-gtid-purged=OFF sql_practice > dump_clean.sql
```
после чего в папке `my-mysql` появится текстовый файл `dump.sql`, содержимое которого можно увидеть командой
```shell
cat dump.sql
```
`.sql` файл является текстовым представлением вашей БД (текстовый SQL-скрипт). Его можно хранить как резервную копия вашей БД, из которой потом её можно восстановить

---

#### 🗑️ Шаг 10: Удаление и изменение данных (DELETE)


Снова зайти в конейнер с БД
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
выбрать БД
```sql
USE sql_practice
```
```sql
SHOW TABLES;
```
```sql
SELECT * FROM customers;
SELECT * FROM orders;
```
Удалить указанную запись из таблицы
```sql
DELETE FROM orders WHERE id = 6;
```
⚠️ Без `WHERE` удалятся все строки: `DELETE FROM orders;`

Проверить:
```sql
SELECT * FROM orders;
```

---

#### 🏗️ Шаг 11: Изменение структуры (ALTER TABLE)

##### Добавить столбец
```sql
ALTER TABLE customers ADD COLUMN phone VARCHAR(20);
```
проверить
```sql
SELECT * FROM customers;
```
Обновить данные в столбце phone
```sql
UPDATE customers SET phone = '123-36-77' WHERE id = 4;
```
Проверить
```sql
SELECT * FROM customers WHERE id = 4;
```
или все записи
```sql
SELECT * FROM customers;
```

##### Изменить тип столбца
```sql
ALTER TABLE customers MODIFY COLUMN phone VARCHAR(30);
```

##### Удалить столбец
```sql
ALTER TABLE customers DROP COLUMN phone;
```
проверить
```sql
SELECT * FROM customers;
```

---

## 🧨 Шаг 12: Удаление таблиц и базы

```sql
DROP TABLE orders;
SHOW TABLES;
DROP TABLE customers;
SHOW TABLES;
DROP DATABASE sql_practice;
```
Проверить
```sql
SHOW DATABASES;
```
выйти из контейнера
```sql
\q
```

#### 🧨 Шаг 13: Восстановление удалённой БД


Проверьте, что контейнер запущен
```shell
docker ps -a
```
если не запущен - запутите:
```shell
docker start my-mysql
```
Проверьте, что дамп на месте и не пуст
```shell
ls -lh dump.sql
```
Файл должен быть в текущей директории, где вы его создавали. Если размер 0 байт — дамп пустой, восстанавливать нечего. Убедитесь, что вы находитесь в той же папке, где запускали mysqldump, либо укажите полный путь.

Быстрая проверка содержимого:
```shell
head -n 30 dump.sql
```
зайти в контейнер
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

Затем выполните создание пустой БД:
```sql
CREATE DATABASE sql_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```
проверьте:
```sql
SHOW DATABASES;
```
Выйдите из MySQL:
```sql
\q
```
Команда импорта БД из дампа выполняется не внутри mysql, а в обычном терминале хоста (с игнорированием ошибок импорта):
```shell
docker exec -i my-mysql mysql -uroot -p123 --force sql_practice < dump.sql
```
Чтобы проверить результат, нужно снова зайти в контейнер с БД
```shell
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

Проверить содержимое восстановленной БД
```sql
SHOW DATABASES;
USE sql_practice;
SHOW TABLES;
SELECT * FROM customers;
SELECT * FROM orders;
```

---

#### 🧰 Полезные команды

| Команда | Что делает |
|---------|------------|
| `SHOW DATABASES;` | Список баз данных |
| `USE имя_базы;` | Переключиться на базу |
| `SHOW TABLES;` | Список таблиц в текущей базе |
| `DESCRIBE имя_таблицы;` | Структура таблицы |
| `SHOW CREATE TABLE имя_таблицы;` | SQL для создания таблицы |
| `EXIT;` или `\q` | Выйти из MySQL |
| `\G` вместо `;` | Вертикальный вывод строки |

---

#### 📝 Что дальше?

Когда освоите эти основы, можно переходить к:
- **Представлениям** (`CREATE VIEW`)
- **Индексам** (`CREATE INDEX`)
- **Транзакциям** (`START TRANSACTION`, `COMMIT`, `ROLLBACK`)
- **Хранимым процедурам и триггерам**
- **Оконным функциям** (`ROW_NUMBER()`, `RANK()`, `OVER`)

---

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!
