# Представления (VIEW) в MySQL: подробное руководство

Вы уже знакомы с основами VIEW. Теперь разберём то, что обычно остаётся за скобками: как MySQL обрабатывает VIEW внутри, какие есть тонкости с алгоритмами, безопасностью и обновляемостью.

---

## 🔹 1. Что такое VIEW на самом деле

**Определение:** VIEW — это **именованный сохранённый запрос** в словаре данных MySQL. Он не хранит строки, а хранит **текст SQL-запроса** и метаданные (имя, определитель, кодировку, алгоритм обработки).

Когда вы пишете:

```sql
SELECT * FROM my_view;
```

MySQL **подставляет** вместо `my_view` его определение и выполняет получившийся запрос. Это называется **развёртыванием (view resolution)**.

**Что хранится в словаре данных:**

| Что | Где хранится |
|-----|--------------|
| Имя VIEW | `information_schema.VIEWS` / `mysql` data dictionary |
| Текст запроса | `information_schema.VIEWS.VIEW_DEFINITION` |
| Определитель (DEFINER) | `information_schema.VIEWS.DEFINER` |
| Режим безопасности | `information_schema.VIEWS.SECURITY_TYPE` |
| Алгоритм | `information_schema.VIEWS.ALGORITHM` |
| Проверка опций | `information_schema.VIEWS.CHECK_OPTION` |

Проверить можно так:

```sql
SELECT * FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'sql_practice'\G
```

---

## 🔹 2. Как MySQL обрабатывает VIEW: три алгоритма

При создании VIEW можно указать `ALGORITHM`:

```sql
CREATE ALGORITHM = MERGE VIEW my_view AS SELECT ...;
CREATE ALGORITHM = TEMPTABLE VIEW my_view AS SELECT ...;
CREATE ALGORITHM = UNDEFINED VIEW my_view AS SELECT ...;  -- по умолчанию
```

### 🔸 MERGE (слияние)

MySQL **вставляет текст VIEW** прямо в внешний запрос, как макрос.

```sql
-- VIEW определён как:
CREATE ALGORITHM = MERGE VIEW moscow_customers AS
SELECT id, name, email FROM customers WHERE city = 'Москва';

-- Вы выполняете:
SELECT name FROM moscow_customers WHERE id > 1;

-- MySQL фактически выполняет:
SELECT name FROM customers WHERE city = 'Москва' AND id > 1;
```

**Плюсы:**
- Оптимизатор видит весь запрос целиком — может использовать индексы, переставлять условия, применять `EXPLAIN` к реальным таблицам.
- Не создаёт временных таблиц.

**Минусы:**
- Не все VIEW можно слить. Если внутри `GROUP BY`, `DISTINCT`, `UNION`, агрегаты, подзапросы в `SELECT` — MERGE невозможен.

### 🔸 TEMPTABLE (временная таблица)

MySQL **сначала выполняет VIEW**, сохраняет результат во **временную таблицу** (в памяти или на диске), а потом применяет к ней внешний запрос.

```sql
-- VIEW с агрегацией:
CREATE ALGORITHM = TEMPTABLE VIEW customer_summary AS
SELECT c.id, c.name, SUM(o.amount) AS total
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Вы выполняете:
SELECT * FROM customer_summary WHERE total > 2000;

-- MySQL:
-- 1. Выполняет весь VIEW → временная таблица
-- 2. Фильтрует временную таблицу по total > 2000
```

**Плюсы:**
- Работает для **любых** запросов, включая агрегаты и `UNION`.

**Минусы:**
- Временная таблица не имеет индексов (кроме случаев, когда MySQL может создать внутренний индекс).
- Внешний `WHERE` **не может** быть спущен внутрь — фильтрация происходит **после** материализации.
- Медленнее на больших данных.

### 🔸 UNDEFINED (по умолчанию)

MySQL **сам выбирает** между MERGE и TEMPTABLE. Если MERGE возможен — использует его. Если нет — переключается на TEMPTABLE.

**Проверить, какой алгоритм выбран:**

```sql
EXPLAIN SELECT * FROM customer_summary WHERE city = 'Москва';
```

В плане ищите:
- `Select type: SIMPLE` и работа с базовыми таблицами → MERGE.
- `Select type: DERIVED` или `<temporary>` → TEMPTABLE.

---

## 🔹 3. Обновляемые и необновляемые VIEW

VIEW можно **обновлять** (`UPDATE`, `INSERT`, `DELETE`) только если MySQL может **однозначно сопоставить** строку VIEW со строкой базовой таблицы.

### ✅ Правила обновляемости

VIEW обновляемо, если:

1. **Одна базовая таблица** в `FROM` (без `JOIN`).
2. **Нет** `GROUP BY`, `HAVING`, `DISTINCT`, `UNION`, агрегатов.
3. **Нет** подзапросов в `SELECT`.
4. **Нет** вычисляемых столбцов без алиасов (например, `amount * 2 AS double_amount` — `double_amount` нельзя обновить, но `amount` можно).
5. **Нет** `LIMIT` (в некоторых версиях).
6. Все неключевые столбцы VIEW **напрямую** соответствуют столбцам таблицы.

### Пример: обновляемое VIEW

```sql
CREATE VIEW active_customers AS
SELECT id, name, email, city
FROM customers
WHERE city IS NOT NULL;

UPDATE active_customers SET city = 'Сочи' WHERE id = 2;
-- Работает: MySQL знает, что city = 'Сочи' нужно записать в customers.city
```

### Пример: необновляемое VIEW

```sql
CREATE VIEW customer_summary AS
SELECT c.id, c.name, COUNT(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

UPDATE customer_summary SET name = 'X' WHERE id = 1;
-- ERROR 1288: The target table customer_summary of the UPDATE is not updatable
```

Почему: `COUNT(o.id)` — агрегат, его нельзя «разложить» обратно. MySQL не знает, какую строку `customers` менять, и что делать с `orders`.

### Проверить обновляемость

```sql
SELECT IS_UPDATABLE
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'sql_practice' AND TABLE_NAME = 'customer_summary';
```

`IS_UPDATABLE = YES` или `NO`.

---

## 🔹 4. WITH CHECK OPTION: защита от «невидимых» строк

По умолчанию через VIEW можно вставить строку, которая **не попадёт** в его фильтр:

```sql
CREATE VIEW moscow_customers AS
SELECT id, name, city FROM customers WHERE city = 'Москва';

INSERT INTO moscow_customers (name, city) VALUES ('Пётр', 'Казань');
-- Строка вставилась в customers, но через VIEW её не видно!
```

`WITH CHECK OPTION` запрещает такие вставки:

```sql
CREATE OR REPLACE VIEW moscow_customers AS
SELECT id, name, city FROM customers WHERE city = 'Москва'
WITH CHECK OPTION;

INSERT INTO moscow_customers (name, city) VALUES ('Пётр', 'Казань');
-- ERROR 1369: CHECK OPTION failed 'sql_practice.moscow_customers'
```

### LOCAL vs CASCADED

Если VIEW основан на другом VIEW, `WITH CHECK OPTION` может проверять только свой уровень (`LOCAL`) или все уровни (`CASCADED`, по умолчанию).

```sql
-- VIEW 1
CREATE VIEW v1 AS
SELECT id, name, city FROM customers WHERE city IS NOT NULL
WITH CHECK OPTION;

-- VIEW 2 на основе VIEW 1
CREATE VIEW v2 AS
SELECT id, name FROM v1 WHERE city = 'Москва'
WITH LOCAL CHECK OPTION;  -- проверяет только v2
-- или
WITH CASCADED CHECK OPTION;  -- проверяет v2 и v1
```

**Правило:** `CASCADED` безопаснее (проверяет всю цепочку), `LOCAL` — только текущий уровень. По умолчанию — `CASCADED`.

---

## 🔹 5. Безопасность: DEFINER и SQL SECURITY

VIEW может выполняться от имени:

- **`DEFINER`** (по умолчанию) — пользователя, создавшего VIEW.
- **`INVOKER`** — пользователя, обращающегося к VIEW.

```sql
CREATE DEFINER = 'admin'@'localhost' SQL SECURITY DEFINER VIEW my_view AS ...;
CREATE DEFINER = 'admin'@'localhost' SQL SECURITY INVOKER VIEW my_view AS ...;
```

### SQL SECURITY DEFINER (по умолчанию)

VIEW выполняется с **правами создателя**. Пользователь может не иметь доступа к таблице `customers`, но если ему дано право на VIEW, он получит данные.

**Сценарий:** администратор создаёт VIEW, скрывающий зарплаты, и даёт пользователю доступ к VIEW без доступа к таблице.

### SQL SECURITY INVOKER

VIEW выполняется с **правами того, кто обращается**. Если у пользователя нет доступа к `customers`, VIEW вернёт ошибку.

**Сценарий:** когда нужно, чтобы права проверялись на уровне базовых таблиц.

### Проверить:

```sql
SELECT DEFINER, SECURITY_TYPE
FROM information_schema.VIEWS
WHERE TABLE_SCHEMA = 'sql_practice' AND TABLE_NAME = 'my_view';
```

---

## 🔹 6. VIEW и производительность: тонкости

### MERGE не всегда лучше

Если VIEW содержит `WHERE city = 'Москва'`, а внешний запрос — `WHERE id > 100`, MERGE сольёт условия:

```sql
-- Фактически выполняется:
WHERE city = 'Москва' AND id > 100
```

Оптимизатор может использовать индекс по `city` или по `id` — как выгоднее.

### TEMPTABLE всегда материализует

Если VIEW содержит `GROUP BY`, внешний `WHERE` **не может** быть применён до материализации. Временная таблица создаётся **целиком**, потом фильтруется.

```sql
CREATE VIEW customer_summary AS
SELECT c.id, c.name, SUM(o.amount) AS total
FROM customers c LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;

-- Этот запрос:
SELECT * FROM customer_summary WHERE total > 10000;

-- Фактически:
-- 1. Выполнить весь VIEW (все клиенты, все суммы) → temp table
-- 2. Отфильтровать temp table по total > 10000
```

**Вывод:** если нужен частый фильтр по агрегату — лучше хранить результат в **реальной таблице** (материализовать) и обновлять по расписанию.

### EXPLAIN для VIEW

```sql
EXPLAIN SELECT * FROM customer_summary WHERE city = 'Москва';
```

Смотрите:
- `Select type: DERIVED` — VIEW материализуется во временную таблицу (TEMPTABLE).
- `Select type: SIMPLE` + работа с базовыми таблицами — MERGE.

Если план плохой — можно:
1. Упростить VIEW.
2. Переписать VIEW так, чтобы был возможен MERGE (убрать `GROUP BY` или `DISTINCT`).
3. Использовать прямые запросы.

---

## 🔹 7. Ограничения VIEW в MySQL

| Ограничение | Детали |
|-------------|--------|
| Нельзя создать индекс на VIEW | Индексы только на таблицах |
| Нельзя использовать временную таблицу в VIEW | `CREATE TEMPORARY TABLE` не допускается в определении |
| Нельзя использовать параметры | VIEW не принимает аргументы (в отличие от хранимых процедур) |
| Нельзя ссылаться на таблицы из другой СУБД | Только текущий сервер |
| Определение VIEW хранится в текстовом виде | Если базовая таблица переименована — VIEW «сломается» до обновления |
| Максимальная глубина вложенности | Ограничена `max_view_nesting` (по умолчанию 64) |
| Нельзя обновлять VIEW с `LIMIT` | В некоторых версиях MySQL |
| `ORDER BY` в VIEW игнорируется | Если внешний запрос не содержит `ORDER BY`, порядок не гарантирован |

---

## 🔹 8. Практические сценарии использования

### 8.1. Скрытие столбцов (безопасность)

```sql
CREATE VIEW public_employees AS
SELECT id, name, department FROM employees;  -- без salary

GRANT SELECT ON sql_practice.public_employees TO 'analyst'@'%';
-- Нет доступа к employees напрямую
```

### 8.2. Упрощение сложных запросов

```sql
CREATE VIEW order_details AS
SELECT o.id AS order_id, o.order_date,
       c.name AS customer, c.city,
       p.title AS product, oi.quantity, p.price,
       oi.quantity * p.price AS line_total
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id;
```

Теперь вместо четырёх `JOIN` — одна строка:

```sql
SELECT * FROM order_details WHERE order_id = 1;
```

### 8.3. Абстракция от изменений схемы

Если приложение обращается к `customer_summary`, а вы переименовали `customers` в `clients` — достаточно обновить VIEW (`CREATE OR REPLACE`), и приложение продолжит работать.

### 8.4. Разграничение доступа по строкам

```sql
CREATE VIEW my_orders AS
SELECT * FROM orders WHERE customer_id = CURRENT_USER_ID();
```

Пользователь видит только свои заказы, хотя таблица содержит все.

---

## 🔹 9. Типичные ошибки и подводные камни

| Ошибка | Причина | Решение |
|--------|---------|---------|
| `View 'x' already exists` | VIEW с таким именем уже есть | `CREATE OR REPLACE` или `DROP VIEW` |
| `The target table x of the UPDATE is not updatable` | VIEW содержит `GROUP BY` / агрегаты / `UNION` / `DISTINCT` | Не обновлять через VIEW |
| `CHECK OPTION failed` | Вставка нарушает `WHERE` в VIEW | Не вставлять или убрать `WITH CHECK OPTION` |
| `Unknown table 'x'` | VIEW не существует или не выбрана база | `SHOW FULL TABLES WHERE Table_type = 'VIEW';` |
| `View's SELECT contains a subquery in the FROM clause` | Подзапрос в `FROM` внутри VIEW | Переписать без подзапроса или использовать TEMPTABLE |
| VIEW работает медленно | TEMPTABLE + большой объём | `EXPLAIN`, упростить, материализовать вручную |
| Данные в VIEW не обновляются | VIEW не обновляемо или TEMPTABLE | Проверить `IS_UPDATABLE`, использовать прямые запросы |

---

## 🔹 10. Шпаргалка: всё о VIEW в одной таблице

| Аспект | Что важно |
|--------|-----------|
| **Суть** | Сохранённый запрос, не хранит данные |
| **Алгоритмы** | `MERGE` (слияние), `TEMPTABLE` (материализация), `UNDEFINED` (авто) |
| **MERGE** | Быстро, индексы, спуск условий |
| **TEMPTABLE** | Медленно, временная таблица, нет индексов |
| **Обновляемость** | Только одна таблица, без агрегатов и `GROUP BY` |
| **WITH CHECK OPTION** | Запрет на вставку строк вне фильтра (`LOCAL` / `CASCADED`) |
| **SQL SECURITY** | `DEFINER` (права создателя) / `INVOKER` (права вызывающего) |
| **Индексы** | На VIEW нельзя |
| **Параметры** | VIEW не принимает аргументы |
| **Дамп** | `mysqldump` включает VIEW по умолчанию |
| **Права** | `GRANT SELECT ON view` отдельно от таблицы |

---

## 📌 Итог

VIEW — это **не «таблица без данных»**, а **механизм подстановки запроса**. Понимание трёх вещей делает работу с ними осознанной:

1. **Алгоритм (MERGE vs TEMPTABLE)** определяет производительность: MERGE — быстро, TEMPTABLE — предсказуемо, но медленно.
2. **Обновляемость** — не все VIEW можно менять: одна таблица, без агрегатов.
3. **Безопасность** — `DEFINER` / `INVOKER` и `WITH CHECK OPTION` дают контроль доступа и целостности.

