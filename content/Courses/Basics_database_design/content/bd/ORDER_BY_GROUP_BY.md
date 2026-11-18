# Ускорение сортировки и группировки в MySQL (ORDER BY / GROUP BY). Практическое руководство

> **Это задание выполнять строго в WSL!**

**Цель:** понять, когда MySQL вынужден выполнять `Using filesort` и `Using temporary`, и научиться проектировать индексы так, чтобы сортировка и группировка выполнялись за счёт порядка хранения данных, а не дополнительных операций.

**План руководства:**

1. Создать таблицу с данными и убедиться, что `ORDER BY` без индекса даёт `Using filesort`.
2. Создать индекс, соответствующий `ORDER BY`, и убрать `filesort`.
3. Разобрать влияние направления сортировки (`ASC`/`DESC`) на возможность использования индекса.
4. Показать, что `GROUP BY` без индекса даёт `Using temporary; Using filesort`.
5. Создать индекс для `GROUP BY` и сравнить `Loose Index Scan` и `Tight Index Scan`.
6. Дополнительно: покрывающий индекс для `GROUP BY`.
7. Научиться вести и экспортировать лог действий пользователя.

---

## 🔹 0. Подготовка `my.cnf`

В каталоге, откуда будете запускать контейнер, создайте файл `my.cnf`:

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

> ⚠️ Если файла `my.cnf` не будет, Docker создаст **директорию** с таким именем, и MySQL не запустится. Убедитесь, что это именно файл.

---

## 🔹 1. Запуск контейнера MySQL

### 1.1. Удалите старый контейнер и volume (если есть)

```shell
docker stop my-mysql 2>/dev/null
docker rm my-mysql 2>/dev/null
docker volume rm mysql_data 2>/dev/null
```

> ⚠️ **Не используйте `docker volume prune -a`** — эта команда удаляет **все** неиспользуемые volumes на хосте, а не только `mysql_data`.

**Удалите файлы логов и дампов от предыдущих заданий!**

Например так (имена файлов могут отличаться!):

Получить все файлы в каталоге `my-mysql`
```shell
ls
```
и
```shell
rm general.log  index_dump_clean.sql  index_dump.sql  my_actions.log
```

### 1.2. Создайте новый контейнер

```bash
docker run --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=123 \
  -e LANG=C.UTF-8 \
  -e MYSQL_INITDB_SKIP_TZINFO=1 \
  -v mysql_data:/var/lib/mysql \
  -v $(pwd)/my.cnf:/etc/mysql/conf.d/my.cnf \
  -d mysql:8.0
```

Убедитесь, что контейнер запущен:

```bash
docker ps
```

### 1.3. Очистите лог перед началом работы

```shell
docker exec -u mysql my-mysql sh -c ': > /var/lib/mysql/general.log'
```

### 1.4. Подключение к MySQL

```bash
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```

Введите пароль `123`. Вы увидите приглашение `mysql>`.

---

## 🔹 2. Создание базы `sort_practice` и таблицы `orders`

```sql
CREATE DATABASE sort_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE sort_practice;

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'new',
    order_date DATE NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    INDEX idx_customer (customer_id)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Заполним данными. Чтобы `filesort` и `temporary` были заметны, создадим несколько сотен строк с разными `status` и `order_date`:

```sql
INSERT INTO orders (customer_id, status, order_date, amount)
SELECT
    FLOOR(1 + RAND() * 50),
    ELT(FLOOR(1 + RAND() * 4), 'new', 'paid', 'cancelled', 'shipped'),
    DATE_ADD('2026-01-01', INTERVAL FLOOR(RAND() * 365) DAY),
    ROUND(100 + RAND() * 4900, 2)
FROM
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5
     UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) a,
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5
     UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) b,
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5) c;
```

Проверим количество строк:

```sql
SELECT COUNT(*) FROM orders;
```

Должно быть **500** строк.

---

## 🔹 3. Кратко: откуда берутся `Using filesort` и `Using temporary`

| Маркер в `EXPLAIN` | Что означает | Когда возникает |
|--------------------|--------------|-----------------|
| **`Using filesort`** | MySQL **не может использовать индекс** для получения строк в нужном порядке и выполняет **дополнительную сортировку** (в памяти или на диске). Название обманчиво: это не обязательно «файл», а именно **дополнительный этап сортировки**. | `ORDER BY` по столбцу, который не является первым в индексе, или направление сортировки не совпадает с индексом. |
| **`Using temporary`** | MySQL создаёт **внутреннюю временную таблицу** для промежуточных результатов. | `GROUP BY` или `DISTINCT` по неиндексированному столбцу; `ORDER BY` по другому столбцу, чем `GROUP BY`. |

> **Правило:** один маркер — повод задуматься. Два одновременно (`Using temporary; Using filesort`) — почти всегда требуют оптимизации.

---

## 🔹 4. Эксперимент 1: `ORDER BY` без индекса

**Задача:** получить заказы, отсортированные по `amount` (по убыванию).

```sql
EXPLAIN SELECT * FROM orders ORDER BY amount DESC;
```

**Ожидаемый результат:**

| type | key | rows | Extra |
|------|-----|------|-------|
| ALL | NULL | 500 | **Using filesort** |

`type: ALL` — полный перебор таблицы. `Extra: Using filesort` — MySQL сортирует строки **после** их чтения. На 500 строках это ещё терпимо, но на миллионах — серьёзная нагрузка.

---

## 🔹 5. Эксперимент 2: индекс, соответствующий `ORDER BY`

Создадим индекс по `amount`:

```sql
CREATE INDEX idx_amount ON orders(amount);
```

Повторим `EXPLAIN`:

```sql
EXPLAIN SELECT * FROM orders ORDER BY amount DESC;
```

**Ожидаемый результат:**

| type | key | rows | Extra |
|------|-----|------|-------|
| index | idx_amount | 500 | **NULL** (или `Using index`) |

`Extra` больше **не содержит** `Using filesort` — индекс `idx_amount` хранит значения в отсортированном порядке, и MySQL читает их сразу в нужном порядке.

> 💡 Если вы используете `SELECT *`, MySQL может всё равно выбрать полный перебор, если чтение таблицы «дороже» чтения индекса. В нашем случае индекс используется, потому что `amount` — единственный столбец, нужный для сортировки.

---

## 🔹 6. Эксперимент 3: направление сортировки и индекс

Индекс может использоваться, если направления сортировки **однородны** (все `ASC` или все `DESC`) или **соответствуют** направлениям в индексе.

### 6.1. Смешанная сортировка `ORDER BY amount ASC` (по возрастанию)

```sql
EXPLAIN SELECT * FROM orders ORDER BY amount ASC;
```

MySQL **может** использовать индекс `idx_amount` в обратном порядке (backward scan), если это эффективно. `Using filesort` может отсутствовать.

### 6.2. Смешанная сортировка по двум столбцам

Создадим составной индекс:

```sql
CREATE INDEX idx_status_amount ON orders(status, amount);
```

Проверим запрос с **однородной** сортировкой:

```sql
EXPLAIN SELECT * FROM orders ORDER BY status ASC, amount ASC;
```

**Ожидаемый результат:** `key: idx_status_amount`, `Extra` без `filesort` — индекс хранит данные в порядке `(status, amount)`.

А теперь запрос с **разнонаправленной** сортировкой:

```sql
EXPLAIN SELECT * FROM orders ORDER BY status ASC, amount DESC;
```

**Ожидаемый результат:** `Extra: Using filesort` — индекс `(status ASC, amount ASC)` не может напрямую удовлетворить `ORDER BY status ASC, amount DESC`. MySQL придётся сортировать.

> 💡 В MySQL 8.0 можно создать **нисходящий индекс**: `CREATE INDEX idx_status_amount_desc ON orders(status ASC, amount DESC);` Тогда запрос с `ORDER BY status ASC, amount DESC` будет использовать этот индекс без `filesort`.

---

## 🔹 7. Эксперимент 4: `GROUP BY` без индекса

**Задача:** посчитать количество заказов по каждому статусу.

```sql
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status;
```

**Ожидаемый результат:**

| type | key | rows | Extra |
|------|-----|------|-------|
| ALL | NULL | 500 | **Using temporary; Using filesort** |

MySQL создаёт временную таблицу для группировки и сортирует её. Это самый ресурсоёмкий вариант.

---

## 🔹 8. Эксперимент 5: индекс для `GROUP BY` и `Loose Index Scan`

Создадим индекс по `status`:

```sql
CREATE INDEX idx_status ON orders(status);
```

Повторим `EXPLAIN`:

```sql
EXPLAIN SELECT status, COUNT(*) FROM orders GROUP BY status;
```

**Ожидаемый результат:**

| type | key | rows | Extra |
|------|-----|------|-------|
| index | idx_status | 4 (или меньше) | **NULL** |

`Using temporary; Using filesort` **исчезли**. MySQL использует **Loose Index Scan** — самый эффективный способ группировки, при котором читается **только по одной строке на группу**, а не все строки.

> 💡 **Loose Index Scan** возможен, если:
> - запрос к одной таблице;
> - `GROUP BY` использует **левый префикс** индекса;
> - агрегатные функции — только `MIN()` или `MAX()` по столбцу, который идёт сразу после `GROUP BY` в индексе.

---

## 🔹 9. Эксперимент 6: `Tight Index Scan`

Если условия для `Loose Index Scan` не выполняются, но индекс по `GROUP BY` есть, MySQL может использовать **Tight Index Scan** — он читает **все** ключи в диапазоне, но всё равно избегает временной таблицы.

Пример: `GROUP BY` с агрегатом, отличным от `MIN`/`MAX`:

```sql
EXPLAIN SELECT status, SUM(amount) FROM orders GROUP BY status;
```

С индексом `idx_status` MySQL, скорее всего, выполнит **Tight Index Scan**: `Extra` может содержать `Using index` (если индекс покрывающий) или быть пустым, но **без** `Using temporary`.

---

## 🔹 10. Эксперимент 7: покрывающий индекс для `GROUP BY`

Если индекс содержит **все** столбцы, нужные для группировки и агрегата, MySQL может обойтись только индексом, без чтения таблицы.

```sql
CREATE INDEX idx_status_amount_cover ON orders(status, amount);

EXPLAIN SELECT status, SUM(amount) FROM orders GROUP BY status;
```

**Ожидаемый результат:** `Extra: Using index`. Временная таблица не создаётся, таблица не читается — всё берётся из индекса.

---

## 🔹 11. Итоговая шпаргалка

| Что хотите ускорить | Какой индекс создать | Что должно быть в `EXPLAIN` |
|----------------------|----------------------|------------------------------|
| `ORDER BY col` | `INDEX(col)` | `Extra` без `Using filesort` |
| `ORDER BY col1, col2` | `INDEX(col1, col2)` | `Extra` без `Using filesort` |
| `WHERE col = const ORDER BY col2` | `INDEX(col, col2)` | `Extra` без `Using filesort` |
| `GROUP BY col` | `INDEX(col)` | `Extra` без `Using temporary` |
| `GROUP BY col1, col2` | `INDEX(col1, col2)` | `Extra` без `Using temporary` |
| `GROUP BY col` + агрегат | Покрывающий `INDEX(col, agg_col)` | `Extra: Using index` |

| Маркер | Хорошо/плохо | Что делать |
|--------|--------------|------------|
| `Using filesort` | ❌ Плохо | Создать индекс, соответствующий `ORDER BY` |
| `Using temporary` | ❌ Плохо | Создать индекс, соответствующий `GROUP BY` / `DISTINCT` |
| `Using index` | ✅ Отлично | Покрывающий индекс работает |

---

## 🔹 12. Ведение и экспорт лога действий пользователя

Все команды записываются в `/var/lib/mysql/general.log`.

### 12.1. Проверить, что лог пишется

```sql
\q
```

```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```

### 12.2. Отфильтровать только свои действия

```shell
docker exec my-mysql cat /var/lib/mysql/general.log \
  | grep -E "Query[[:space:]]+(CREATE|INSERT|SELECT|EXPLAIN|SHOW|USE|SET)" \
  | grep -v -E "INSERT INTO help_|INSERT INTO time_zone|INSERT INTO mysql\.|SELECT \* FROM mysql\.general_log|SET NAMES|SET autocommit|SET sql_mode|SET character_set_results" \
  > my_actions.log
```

Проверить:

```shell
cat my_actions.log
```

### 12.3. Забрать полный лог на хост

```shell
docker cp my-mysql:/var/lib/mysql/general.log ./general.log
```

---

## 🔹 13. Создание дампа базы

```sql
\q
```

```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot \
  --set-gtid-purged=OFF \
  --single-transaction \
  sort_practice > sort_dump.sql
```

Посмотрите содержимое (первые 30 строк):

```shell
head -n 30 sort_dump.sql
```

> ⚠️ В дамп попадут все индексы, созданные в экспериментах (`idx_amount`, `idx_status_amount`, `idx_status`, `idx_status_amount_cover`). При необходимости их можно удалить перед дампом.

---

**Результат оформить в виде отдельного `README.md` со скриншотами `EXPLAIN` до и после создания индексов (для `ORDER BY` и `GROUP BY`), дампом и логом (лог-файл должен содержать только ваши действия) в этой же папке и прислать ссылку преподавателю.**

---

## 📌 Итог

| Что важно запомнить об ускорении `ORDER BY` и `GROUP BY` |
|-----------------------------------------------------------|
| **`Using filesort`** — MySQL не может использовать индекс для сортировки и выполняет дополнительный этап сортировки |
| **`Using temporary`** — MySQL создаёт временную таблицу для группировки или дедупликации |
| Индекс должен **соответствовать** `ORDER BY` / `GROUP BY` по столбцам и **направлению** сортировки |
| **Loose Index Scan** — самый быстрый способ `GROUP BY`: читает по одной строке на группу |
| **Покрывающий индекс** для `GROUP BY` даёт `Using index` — временная таблица не создаётся |
| Для смешанной сортировки (`ASC`/`DESC`) можно создать **нисходящий индекс** (MySQL 8.0+) |

---

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!