# Индексы и `JOIN`: оптимизация соединений в MySQL. Практическое руководство

> **Это задание выполнять строго в WSL!**

**Цель:** понять, почему `JOIN` без индексов может «подвесить» сервер, и научиться проектировать индексы так, чтобы соединение таблиц выполнялось за счёт быстрого поиска, а не полного перебора.

**План руководства:**

1. Создать две таблицы и убедиться, что `JOIN` без индекса на столбце соединения даёт `type: ALL`.
2. Создать индекс на столбце соединения и увидеть, как `type` меняется на `ref` или `eq_ref`.
3. Разобрать автоматическое создание индексов для внешних ключей.
4. Понять роль порядка соединения (какая таблица — таблица драйверов).
5. Дополнительно: составной индекс для `JOIN` + `WHERE`.
6. Сравнить `INNER JOIN` и `LEFT JOIN` с точки зрения оптимизации.
7. Научиться вести и экспортировать лог действий пользователя.

---

## 🔹 0. Подготовка `my.cnf`

В каталоге с WSL, откуда будете запускать контейнер (WSL), создайте файл `my.cnf`:
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

> ⚠️ Если файла `my.cnf` не будет, Docker создаст **директорию** с таким именем, и MySQL не запустится. Убедитесь, что это именно файл в WSL. Для Windows (крайне не желательно использовать Power Shell и Git-Bash!) — сохраняйте в **UTF-8 без BOM** и с **LF** (см. отдельное руководство по `my.cnf`).

---

## 🔹 1. Запуск контейнера MySQL

### 1.1. Удалите старый контейнер и volume (если есть)

```shell
docker stop my-mysql 2>/dev/null
docker rm my-mysql 2>/dev/null
docker volume rm mysql_data 2>/dev/null
```

> ⚠️ **Не используйте `docker volume prune -a`** — эта команда удаляет **все** неиспользуемые volumes на хосте, а не только `mysql_data`.

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

## 🔹 2. Создание базы `join_practice` и таблиц

Создадим две таблицы: `users` (пользователи) и `orders` (заказы). Изначально **не будем создавать индекс на `orders.user_id`** — чтобы увидеть, как `JOIN` работает без него.

```sql
CREATE DATABASE join_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE join_practice;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    vip_level INT DEFAULT 0
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    order_date DATE NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'new'
    -- ВНИМАНИЕ: здесь НЕТ внешнего ключа и НЕТ индекса на user_id
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Заполним данными. Создадим 50 пользователей и 500 заказов, чтобы разница в планах была заметнее:

```sql
INSERT INTO users (name, email, vip_level)
SELECT
    CONCAT('User', n),
    CONCAT('user', n, '@example.com'),
    FLOOR(RAND() * 4)
FROM (
    SELECT a.n + b.n * 10 + 1 AS n
    FROM (SELECT 0 AS n UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4
          UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) a,
         (SELECT 0 AS n UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4) b
) numbers;

INSERT INTO orders (user_id, order_date, amount, status)
SELECT
    FLOOR(1 + RAND() * 50),
    DATE_ADD('2026-01-01', INTERVAL FLOOR(RAND() * 365) DAY),
    ROUND(100 + RAND() * 4900, 2),
    ELT(FLOOR(1 + RAND() * 3), 'new', 'paid', 'cancelled')
FROM
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5
     UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) a,
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5
     UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) b,
    (SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5) c;
```

Проверим:

```sql
SELECT COUNT(*) FROM users;   -- 50
SELECT COUNT(*) FROM orders;  -- 500
```

---

## 🔹 3. Кратко: как MySQL выполняет `JOIN`

По умолчанию MySQL использует **Nested Loop Join** (вложенный цикл): для каждой строки внешней таблицы (driving table) он ищет соответствующие строки во внутренней таблице (driven table).

- **Без индекса на внутренней таблице:** каждый поиск — это **полный перебор** внутренней таблицы. Сложность: **O(M × N)**. При 500 заказах и 50 пользователях — 25 000 сравнений (терпимо), но при миллионах строк — катастрофа.
- **С индексом на внутренней таблице:** поиск превращается в **поиск по B-дереву**. Сложность: **O(M × log N)**. Для тех же данных — примерно 500 × 6 = 3 000 операций, и это без учёта кэша.

> 💡 Начиная с MySQL 8.0.18, для больших таблиц при равенстве соединения может использоваться **Hash Join**, который ещё быстрее, чем Nested Loop. Увидеть его можно через `EXPLAIN FORMAT=TREE` (в `Extra` будет `Using join buffer (hash join)`).

---

## 🔹 4. Эксперимент 1: `JOIN` без индекса на столбце соединения

**Запрос:** найти все заказы пользователей с `vip_level = 3`.

```sql
EXPLAIN SELECT o.id, o.amount, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.vip_level = 3;
```

**Ожидаемый результат (фрагмент):**

| table | type | key | rows | Extra |
|-------|------|-----|------|-------|
| u | ALL | NULL | 50 | Using where |
| o | **ALL** | NULL | 500 | Using where; Using join buffer |

**Что происходит:**
- `users` (u) — полный перебор (50 строк), потому что `vip_level` не индексирован. Это приемлемо.
- `orders` (o) — **`type: ALL`** — для каждой строки из `users` MySQL перебирает **все** 500 строк `orders`. Это и есть «дорогое» место.
- `Using join buffer` — MySQL использует буфер соединения, чтобы хоть как-то ускорить процесс, но это всё равно полный перебор.

На 500 строках это незаметно. Но если `orders` будет 2 000 000 строк, а `users` — 100 000, количество сравнений может достигать **2 × 10¹¹** — сервер «ляжет».

---

## 🔹 5. Эксперимент 2: индекс на столбце соединения

Создадим индекс на `orders.user_id`:

```sql
CREATE INDEX idx_user_id ON orders(user_id);
```

Повторим `EXPLAIN`:

```sql
EXPLAIN SELECT o.id, o.amount, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.vip_level = 3;
```

**Ожидаемый результат (фрагмент):**

| table | type | key | rows | Extra |
|-------|------|-----|------|-------|
| u | ALL | NULL | 50 | Using where |
| o | **ref** | **idx_user_id** | 10 | Using where |

**Что изменилось:**
- `type` для `orders` стало **`ref`** вместо `ALL`.
- `key` — **`idx_user_id`** — индекс используется.
- `rows` — **10** вместо 500. MySQL предполагает, что для каждого пользователя в среднем будет 10 заказов (500 / 50 = 10). Вместо полного перебора 500 строк он будет искать по индексу.

**Вывод:** индекс на столбце соединения превращает `Nested Loop` из «полного перебора» в «быстрый поиск». Это **самая важная оптимизация для `JOIN`**.

---

## 🔹 6. Внешний ключ и автоматический индекс

В InnoDB создание **внешнего ключа** (`FOREIGN KEY`) **автоматически создаёт индекс** на столбце внешнего ключа, если подходящего индекса ещё нет.

В нашем эксперименте мы **не создавали** внешний ключ, поэтому индекса не было. Если бы мы сразу объявили:

```sql
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_user
  FOREIGN KEY (user_id) REFERENCES users(id);
```

MySQL создал бы индекс `idx_user_id` автоматически, и `JOIN` работал бы быстро **с самого начала**.

> 💡 **Практическое правило:** если вы используете внешние ключи, вы, скорее всего, уже получаете индексы на столбцах соединения «бесплатно». Но если внешних ключей нет (например, в учебной схеме или при денормализации) — индекс нужно создавать **явно**.

---

## 🔹 7. Порядок соединения: какая таблица — Таблица драйверов?

MySQL (для `INNER JOIN`) обычно выбирает в качестве driving table таблицу с **меньшим результирующим набором** после фильтрации — это называется «small table drives large table».

В нашем примере `users` с `vip_level = 3` даёт небольшой набор (может быть, 10–15 строк), а `orders` — 500. Оптимизатор, скорее всего, выберет `users` как Таблица драйверов , а `orders` — как подуправляемая таблица, и будет искать заказы по индексу `idx_user_id`.

### Как проверить порядок

В `EXPLAIN` таблицы перечислены **в порядке соединения**. Первая строка — Таблица драйверов.

Если вы хотите **принудительно** изменить порядок, можно использовать `STRAIGHT_JOIN` (не рекомендуется без необходимости) или **index hints** (`FORCE INDEX FOR JOIN`).

> 💡 **Лучшая практика:** не пытайтесь «руками» управлять порядком соединения, если только `EXPLAIN` не показывает явно плохой план. Оптимизатор MySQL обычно справляется хорошо.

---

## 🔹 8. Составной индекс для `JOIN` + `WHERE`

Часто `JOIN` сопровождается фильтрацией по другому столбцу. Например:

```sql
SELECT o.id, o.amount, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.vip_level = 3 AND o.status = 'paid';
```

Если `orders` фильтруется по `status`, а соединяется по `user_id`, можно создать **составной индекс**:

```sql
CREATE INDEX idx_user_status ON orders(user_id, status);
```

Тогда для каждой строки из `users` MySQL сможет искать в `orders` сразу по паре `(user_id, status)`, не читая лишние строки. Это называется **покрывающим индексом для соединения** (если индекс содержит все нужные столбцы) или просто более эффективным доступом.

**Проверьте `EXPLAIN`:**

```sql
EXPLAIN SELECT o.id, o.amount, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.vip_level = 3 AND o.status = 'paid';
```

`Extra` может содержать `Using index` (если индекс покрывающий) или `Using where` с меньшим `rows`.

> 💡 **Правило левого префикса** работает и здесь: индекс `(user_id, status)` эффективен для `user_id` и `(user_id, status)`, но не для `status` в одиночку.

---

## 🔹 9. `INNER JOIN` vs `LEFT JOIN`: что быстрее?

С точки зрения **оптимизации индексов** разница в следующем:

| Аспект | `INNER JOIN` | `LEFT JOIN` |
|--------|--------------|-------------|
| **Порядок таблиц** | Оптимизатор может **свободно переставлять** таблицы | Левая таблица **фиксирована** как таблица драйверов (в большинстве случаев) |
| **Индексы** | Индекс на **любой** таблице может быть использован для поиска | Индекс на **правой** таблице критичен для поиска; на левой — для фильтрации |
| **Фильтр в `WHERE` по правой таблице** | Может быть применён до соединения | **Превращает `LEFT JOIN` в `INNER JOIN`** (если фильтр отсекает `NULL`), что может сбить оптимизатор с толку |

**Практический вывод:** если вам не нужно сохранять строки левой таблицы без совпадений, **используйте `INNER JOIN`** — он даёт оптимизатору больше свободы и обычно работает быстрее.

---

## 🔹 10. Итоговая шпаргалка

| Что хотите ускорить | Что сделать | Что должно быть в `EXPLAIN` |
|----------------------|-------------|------------------------------|
| `JOIN` по столбцу | Создать индекс на **внутренней** (под управлением) таблице | `type: ref` или `eq_ref` для внутренней таблицы |
| `JOIN` по первичному/уникальному ключу | Индекс уже есть (PK/UNIQUE) | `type: eq_ref` — лучший тип соединения |
| `JOIN` + фильтр по другому столбцу | Составной индекс `(join_col, filter_col)` | `type: ref`, меньше `rows` |
| `LEFT JOIN` | Индекс на **правой** таблице обязателен | `type: ref` или `eq_ref` для правой таблицы |
| Маленькая таблица + большая таблица | Убедиться, что оптимизатор выбрал маленькую как таблицу драйверов | Первая строка в `EXPLAIN` — маленькая таблица |

| Тип в `EXPLAIN` | Что значит для `JOIN` | Оценка |
|------------------|------------------------|--------|
| `eq_ref` | Поиск по PK/UNIQUE для каждой строки предыдущей таблицы | ✅ Отлично |
| `ref` | Поиск по неуникальному индексу | ✅ Хорошо |
| `ALL` | Полный перебор для каждой строки предыдущей таблицы | ❌ Плохо |
| `index` | Полный просмотр индекса | ⚠️ Средне |

---

## 🔹 11. Ведение и экспорт лога действий пользователя

Все команды записываются в `/var/lib/mysql/general.log`.

### 11.1. Проверить, что лог пишется

```sql
\q
```

```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```

### 11.2. Отфильтровать только свои действия

```shell
docker exec my-mysql cat /var/lib/mysql/general.log \
  | grep -E "Query[[:space:]]+(CREATE|INSERT|SELECT|EXPLAIN|SHOW|USE|ALTER|SET)" \
  | grep -v -E "INSERT INTO help_|INSERT INTO time_zone|INSERT INTO mysql\.|SELECT \* FROM mysql\.general_log|SET NAMES|SET autocommit|SET sql_mode|SET character_set_results" \
  > my_actions.log
```

Проверить:
```shell
cat my_actions.log
```
или в красивом виде:
```shell
bat my_actions.log
```

### 11.3. Забрать полный лог на хост

```shell
docker cp my-mysql:/var/lib/mysql/general.log ./general.log
```

### 11.4. Очистить лог

```shell
docker exec -u mysql my-mysql sh -c ': > /var/lib/mysql/general.log'
```

---

## 🔹 12. Создание дампа базы

```sql
\q
```

```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot \
  --set-gtid-purged=OFF \
  --single-transaction \
  join_practice > join_dump.sql
```

Посмотрите содержимое (первые 30 строк):

```shell
head -n 30 join_dump.sql
```

> ⚠️ В дамп попадут все индексы, созданные в экспериментах (`idx_user_id`, `idx_user_status`). При необходимости их можно удалить перед дампом.

---

**Результат оформить в виде отдельного `README.md` со скриншотами `EXPLAIN` для `JOIN` до и после создания индекса (сравнение `type` и `rows`), дампом и логом (лог-файл должен содержать только ваши действия) в этой же папке и прислать ссылку преподавателю.**

---

## 📌 Итог

| Что важно запомнить об оптимизации `JOIN` |
|--------------------------------------------|
| **Индекс на столбце соединения** — главное условие быстрого `JOIN`. Без него — `type: ALL` и полный перебор |
| **Внешний ключ** в InnoDB автоматически создаёт индекс на столбце — это «бесплатная» оптимизация |
| **`type: eq_ref`** — лучший тип для `JOIN` (поиск по PK/UNIQUE). **`ref`** — хороший (поиск по обычному индексу). **`ALL`** — плохой |
| **`INNER JOIN`** обычно быстрее `LEFT JOIN`, потому что оптимизатор может свободно переставлять таблицы |
| **Составной индекс** `(join_col, filter_col)` ускоряет `JOIN` с дополнительной фильтрацией |
| **Порядок соединения** («маленькая таблица ведёт большую») выбирается оптимизатором; не вмешивайтесь без необходимости |
| **Hash Join** (MySQL 8.0.18+) может быть быстрее `Nested Loop` для больших таблиц — проверяйте через `EXPLAIN FORMAT=TREE` |

---

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!