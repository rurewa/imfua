## Нормализация данных. Практическое руководство

1. Создайте базу `shop_norm` (SQL).
2. Создайте одну большую «плохую» таблицу `orders_bad` и наполните её.
3. Попробуйте выполнить три операции:
   - Обновите город Ивана в одной строке и посмотрите, что стало с другой.
   - Удалите последний заказ Марии и проверьте, осталась ли информация о ней.
   - Попробуйте вставить клиента без заказа — увидите, что не получается.
4. Создайте четыре таблицы в 3NF.
5. Повторите те же операции — убедитесь, что аномалии исчезли.

Контейнер с официального образа **MySQL**:
```bash
docker run --name shop_norm \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:latest
```
Чтобы убедиться, что контейнер успешно запущен, выполните:
```bash
docker ps
```
Войтите в контейнер
```sql
docker exec -it shop_norm mysql --default-character-set=utf8mb4 -uroot -p
```

### Шаг 1. Создание новой БД с ненормализованной таблицей

1. Создание БД `shop_norm` с таблицей `orders_bad`
```sql
CREATE DATABASE shop_norm
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE shop_norm;

CREATE TABLE orders_bad (
    order_id      INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_city VARCHAR(100),
    product_name  VARCHAR(100),
    product_price INT,
    quantity      INT,
    order_date    DATE
);
```
Проверить результат
```sql
SHOW DATABASES;
```
2. Наполним данными:
```sql
INSERT INTO orders_bad (customer_name, customer_city, product_name, product_price, quantity, order_date) VALUES
  ('Иван',  'Москва',          'Яблоки',   120, 2, '2026-10-01'),
  ('Иван',  'Москва',          'Лимоны',   200, 1, '2026-10-01'),
  ('Мария', 'Казань',          'Яблоки',   120, 3, '2026-10-02'),
  ('Мария', 'Казань',          'Хурма',    350, 1, '2026-10-03'),
  ('Пётр',  'Санкт-Петербург', 'Каштаны',  200, 2, '2026-10-04');
```
Проверить результат
```sql
SHOW TABLES;
```
Проверим содержимое таблицы:
```sql
SELECT * FROM orders_bad;
```
Сколько потратил Иван?
```sql
SELECT SUM(product_price * quantity)
FROM orders_bad
WHERE customer_name = 'Иван';
```
Работает, но данные дублируются, и любое изменение — риск аномалии.

3. Работа с "плохой" таблицей

Обновите город Ивана в одной строке и посмотрите, что стало с другой
```sql
UPDATE orders_bad
SET customer_city = 'Сочи'
WHERE order_id = 1;
```
Проверка:
```sql
SELECT * FROM orders_bad WHERE customer_name = 'Иван';
```
Вывод: город Ивана теперь хранится в двух разных значениях. Это аномалия обновления: один и тот же факт (место жительства) продублирован, и при изменении легко забыть обновить все строки. В нормализованной базе (с отдельной таблицей customers) такое невозможно — город хранился бы в одной строке.

Удалите последний заказ Марии и проверьте, осталась ли информация о ней
```sql
DELETE FROM orders_bad WHERE order_id = 4;
```
Проверка:
```sql
SELECT * FROM orders_bad WHERE customer_name = 'Мария';
```
Вывод: информация о Марии пока осталась, потому что у неё есть ещё один заказ (order_id = 3). Однако если удалить и его:
```sql
DELETE FROM orders_bad WHERE order_id = 3;
```
то Мария полностью исчезнет из базы:
```sql
SELECT * FROM orders_bad WHERE customer_name = 'Мария';
```
Попробуйте вставить клиента без заказа — увидите, что не получается.
```sql
INSERT INTO orders_bad (customer_name, customer_city)
VALUES ('Новый клиент', 'Тверь');
```
Проверка:
```sql
SELECT * FROM orders_bad WHERE customer_name = 'Новый клиент';
```
Вывод: мы не можем сохранить клиента, не создавая фиктивный заказ. Все поля, относящиеся к заказу, оказались NULL. Это аномалия вставки: таблица не позволяет хранить сущность «клиент» отдельно от сущности «заказ».

Итог:

| Аномалия | Что делает запрос | Что показывает |
|----------|-------------------|----------------|
| **Обновления** | Меняет город только в одной строке | Тот же клиент имеет разные города в разных заказах |
| **Удаления** | Удаляет последний заказ Марии | При удалении всех заказов клиент исчезает из базы |
| **Вставки** | Пытается добавить клиента без заказа | Появляется строка с `NULL` в полях заказа |

### Шаг 2. Создание дампа ненормализованной БД

Создание дампа БД (Экспорт в `.sql` файл)

Выйти из БД
```sql
\q
```
Делаем дамп:
```shell
docker exec shop_norm mysqldump -uroot -p123 shop > dump.sql
```
или
```shell
docker exec -e MYSQL_PWD=123 shop_norm mysqldump -uroot shop > dump.sql
```
после чего в папке `shop_norm` появится текстовый файл `dump.sql`, содержимое которого можно увидеть командой
```shell
cat dump.sql
```
`.sql` файл является текстовым представлением вашей БД (текстовый SQL-скрипт). Его можно хранить как резервную копия вашей БД, из которой потом её можно восстановить

### Шаг 3  ✅ Нормализация БД. Приведение к 3NF

В нормализованной базе клиент добавляется в новую таблицу `customers` без каких-либо заказов.

Разделяем на **четыре таблицы**:

```sql
-- Клиенты
CREATE TABLE customers_3nf (
    customer_id   INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    customer_city VARCHAR(100) NOT NULL
);

-- Товары
CREATE TABLE products_3nf (
    product_id    INT AUTO_INCREMENT PRIMARY KEY,
    product_name  VARCHAR(100) NOT NULL,
    product_price INT NOT NULL
);

-- Заказы (шапка)
CREATE TABLE orders_3nf (
    order_id    INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date  DATE NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers_3nf(customer_id)
);

-- Позиции заказа (строки)
CREATE TABLE order_items_3nf (
    order_id   INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id)   REFERENCES orders_3nf(order_id),
    FOREIGN KEY (product_id) REFERENCES products_3nf(product_id)
);
```
Наполним:
```sql
INSERT INTO customers_3nf (customer_name, customer_city) VALUES
  ('Иван',  'Москва'),
  ('Мария', 'Казань'),
  ('Пётр',  'Санкт-Петербург');

INSERT INTO products_3nf (product_name, product_price) VALUES
  ('Яблоки',  120),
  ('Лимоны',  200),
  ('Хурма',   350),
  ('Каштаны', 200);

INSERT INTO orders_3nf (customer_id, order_date) VALUES
  (1, '2026-10-01'),  -- order_id = 1, Иван
  (2, '2026-10-02'),  -- order_id = 2, Мария
  (2, '2026-10-03'),  -- order_id = 3, Мария
  (3, '2026-10-04');  -- order_id = 4, Пётр

INSERT INTO order_items_3nf (order_id, product_id, quantity) VALUES
  (1, 1, 2),  -- Иван: Яблоки × 2
  (1, 2, 1),  -- Иван: Лимоны × 1
  (2, 1, 3),  -- Мария: Яблоки × 3
  (3, 3, 1),  -- Мария: Хурма × 1
  (4, 4, 2);  -- Пётр: Каштаны × 2
```
Проверить:
```sql
SELECT * FROM customers_3nf;
SELECT * FROM products_3nf;
SELECT * FROM orders_3nf;
SELECT * FROM order_items_3nf;
```
Что мы получили?

| Таблица | Что хранит | Дублирование |
|---------|-----------|--------------|
| `customers_3nf` | Клиенты (имя, город) | ❌ Нет |
| `products_3nf` | Товары (название, цена) | ❌ Нет |
| `orders_3nf` | Заказы (кто, когда) | ❌ Нет |
| `order_items_3nf` | Позиции (что, сколько) | ❌ Нет |

**Теперь:**
- Иван хранится **один раз**. Переезд → одно `UPDATE`.
- Цена яблок хранится **один раз**. Изменение → одно `UPDATE`.
- Можно добавить клиента без заказа. Можно добавить товар без продаж.
- Удаление заказа не удаляет клиента и товар.

Убедиться, что новые таблицы работают
```sql
SELECT SUM(p.product_price * oi.quantity) AS total
FROM customers_3nf c
JOIN orders_3nf o      ON c.customer_id = o.customer_id
JOIN order_items_3nf oi ON o.order_id = oi.order_id
JOIN products_3nf p    ON oi.product_id = p.product_id
WHERE c.customer_name = 'Иван';
```
Если результат корректный — нормализация прошла успешно.

Запрос стал длиннее, но:
- Нет дублирования.
- Нет аномалий.
- Легко добавить нового клиента или товар.

### Шаг 4. Перенести данные из старой таблицы в новые

Клиенты (уникальные пары «имя + город»):
```sql
INSERT INTO customers_3nf (customer_name, customer_city)
SELECT DISTINCT customer_name, customer_city
FROM orders_bad;
```
Товары (уникальные пары «название + цена»):
```sql
INSERT INTO products_3nf (product_name, product_price)
SELECT DISTINCT product_name, product_price
FROM orders_bad;
```
Заказы (уникальные order_id + клиент + дата):
```sql
INSERT INTO orders_3nf (order_id, customer_id, order_date)
SELECT o.order_id, c.customer_id, o.order_date
FROM orders_bad o
JOIN customers_3nf c
  ON c.customer_name = o.customer_name
 AND c.customer_city = o.customer_city;
```
Позиции заказов (связь заказ ↔ товар + количество):
```sql
INSERT INTO order_items_3nf (order_id, product_id, quantity)
SELECT o.order_id, p.product_id, o.quantity
FROM orders_bad o
JOIN products_3nf p
  ON p.product_name = o.product_name
 AND p.product_price = o.product_price;
```
Проверить, что данные совпадают
```sql
-- Старая таблица
SELECT SUM(product_price * quantity) AS total_old
FROM orders_bad;

-- Новая схема
SELECT SUM(p.product_price * oi.quantity) AS total_new
FROM order_items_3nf oi
JOIN products_3nf p ON oi.product_id = p.product_id;
```
Числа должны совпасть. Если нет — где-то ошибка в переносе, и старая таблица ещё пригодится для отладки.

Ещё проверки:
```sql
-- Количество заказов
SELECT COUNT(DISTINCT order_id) FROM orders_bad;
SELECT COUNT(*) FROM orders_3nf;

-- Количество клиентов
SELECT COUNT(DISTINCT customer_name, customer_city) FROM orders_bad;
SELECT COUNT(*) FROM customers_3nf;

-- Количество товаров
SELECT COUNT(DISTINCT product_name, product_price) FROM orders_bad;
SELECT COUNT(*) FROM products_3nf;
```
Все пары чисел должны совпасть.

Убедиться, что новые таблицы работают
```sql
-- Сколько потратил Иван?
SELECT SUM(p.product_price * oi.quantity) AS total
FROM customers_3nf c
JOIN orders_3nf o       ON c.customer_id = o.customer_id
JOIN order_items_3nf oi ON o.order_id = oi.order_id
JOIN products_3nf p     ON oi.product_id = p.product_id
WHERE c.customer_name = 'Иван';
```
Если результат корректный — нормализация прошла успешно.


Теперь можно удалить старую таблицу
```sql
DROP TABLE orders_bad;
```
Или, если хотите сохранить её как архив:
```sql
RENAME TABLE orders_bad TO orders_bad_backup;
```
RENAME безопаснее: таблица остаётся в базе, но под другим именем. Если через неделю выяснится, что что-то упустили, — данные под рукой.

**Главная мысль:** нормализация — это не «максимальное количество таблиц», а **устранение аномалий**. Каждый факт должен храниться в одном месте. Если правило нарушено — вы либо получите противоречивые данные, либо не сможете что-то добавить или удалить без потерь.

### Шаг 5. Создание дампа

Создание дампа БД (Экспорт в `.sql` файл)

Выйти из БД
```sql
\q
```
Делаем дамп:
```shell
docker exec -e MYSQL_PWD=123 shop_norm mysqldump -uroot shop > norm_dump.sql
```
после чего в папке `shop_norm` появится текстовый файл `norm_dump.sql`, содержимое которого можно увидеть командой
```shell
cat norm_dump.sql
```
`.sql` файл является текстовым представлением вашей БД (текстовый SQL-скрипт). Его можно хранить как резервную копия вашей БД, из которой потом её можно восстановить

> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!