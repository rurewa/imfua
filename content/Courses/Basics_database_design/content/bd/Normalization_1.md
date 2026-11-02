## Нормализация данных

1. Создайте базу `shop_norm` (SQL).
2. Создайте одну большую «плохую» таблицу `orders_bad` и наполните её.
3. Попробуйте выполнить три операции:
   - Обновите город Ивана в одной строке и посмотрите, что стало с другой.
   - Удалите последний заказ Марии и проверьте, осталась ли информация о ней.
   - Попробуйте вставить клиента без заказа — увидите, что не получается.
4. Создайте четыре таблицы в 3NF.
5. Повторите те же операции — убедитесь, что аномалии исчезли.

Создание БД `shop_norm` с таблицей `orders_bad`
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
Наполним данными:
```sql
INSERT INTO orders_bad (customer_name, customer_city, product_name, product_price, quantity, order_date) VALUES
  ('Иван',  'Москва',          'Яблоки',   120, 2, '2026-10-01'),
  ('Иван',  'Москва',          'Лимоны',   200, 1, '2026-10-01'),
  ('Мария', 'Казань',          'Яблоки',   120, 3, '2026-10-02'),
  ('Мария', 'Казань',          'Хурма',    350, 1, '2026-10-03'),
  ('Пётр',  'Санкт-Петербург', 'Каштаны',  200, 2, '2026-10-04');
```
Сколько потратил Иван?
```sql
SELECT SUM(product_price * quantity)
FROM orders_bad
WHERE customer_name = 'Иван';
```
Работает, но данные дублируются, и любое изменение — риск аномалии.




### ✅ Приведение к 3NF

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

Сколько потратил Иван?
```sql
SELECT SUM(p.product_price * oi.quantity) AS total
FROM customers_3nf c
JOIN orders_3nf o      ON c.customer_id = o.customer_id
JOIN order_items_3nf oi ON o.order_id = oi.order_id
JOIN products_3nf p    ON oi.product_id = p.product_id
WHERE c.customer_name = 'Иван';
```
Запрос стал длиннее, но:
- Нет дублирования.
- Нет аномалий.
- Легко добавить нового клиента или товар.

**Главная мысль:** нормализация — это не «максимальное количество таблиц», а **устранение аномалий**. Каждый факт должен храниться в одном месте. Если правило нарушено — вы либо получите противоречивые данные, либо не сможете что-то добавить или удалить без потерь.

> Если вы обнаружили ошибку в этом тексте - сообщите пожалуйста автору!