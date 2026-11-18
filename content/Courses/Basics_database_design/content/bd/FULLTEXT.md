# Полнотекстовый поиск (FULLTEXT) в MySQL. Практическое руководство

> **Это задание выполнять строго в WSL!**

**Цель:** освоить полнотекстовый поиск, понять, чем он отличается от `LIKE`, и научиться применять его для реальных задач.

**План руководства:**

1. Создать таблицу с текстовыми полями и наполнить её данными.
2. Убедиться, что `LIKE '%слово%'` не использует индекс (full scan).
3. Создать `FULLTEXT`-индекс и выполнить поиск в режиме естественного языка.
4. Разобрать булев режим и его операторы (`+`, `-`, `*`, `"..."`, `~`).
5. Понять ограничения InnoDB: стоп-слова и минимальную длину слова.
6. Сравнить производительность с `LIKE`.
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

> ⚠️ Если файла `my.cnf` не будет, Docker создаст **директорию** с таким именем, и MySQL не запустится.

---

## 🔹 1. Запуск контейнера MySQL

### 1.1. Удалите старый контейнер и volume (если есть)

```shell
docker stop my-mysql 2>/dev/null
docker rm my-mysql 2>/dev/null
docker volume rm mysql_data 2>/dev/null
```

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

## 🔹 2. Создание базы `fulltext_practice` и таблицы `articles`

```sql
CREATE DATABASE fulltext_practice
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE fulltext_practice;

CREATE TABLE articles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    body TEXT NOT NULL,
    author VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Заполним данными:

```sql
INSERT INTO articles (title, body, author) VALUES
  ('MySQL FULLTEXT search', 'MySQL provides full-text search capabilities. You can search for words in text columns using MATCH AGAINST syntax. FULLTEXT indexes are fast.', 'Иван'),
  ('MySQL replication', 'MySQL replication allows you to replicate data from one server to another. It is useful for backup and scaling. FULLTEXT search works well with replication.', 'Мария'),
  ('Optimizing MySQL queries', 'Query optimization is important for performance. Use indexes, EXPLAIN, and FULLTEXT search to speed up slow queries.', 'Пётр'),
  ('InnoDB full-text search', 'InnoDB supports FULLTEXT indexes since MySQL 5.6. InnoDB full-text search is crash-safe and does not have the 50% threshold that MyISAM has.', 'Ольга'),
  ('Boolean mode in MySQL', 'Boolean mode allows you to use operators like + (required), - (excluded), * (wildcard), and "..." (phrase) in FULLTEXT searches.', 'Алексей');
```

Проверим:

```sql
SELECT * FROM articles;
```

---

## 🔹 3. Проблема `LIKE`: полный перебор

### 3.1. `EXPLAIN` для `LIKE`

```sql
EXPLAIN SELECT * FROM articles WHERE body LIKE '%replication%';
```

**Результат:**

| type | key | rows | Extra |
|------|-----|------|-------|
| **ALL** | NULL | 5 | Using where |

`type: ALL` — **full scan**. Индекс не используется, потому что `LIKE '%...%'` не может применить B-дерево (поиск по суффиксу).

> 💡 На 5 строках это не страшно. Но на миллионах строк `LIKE '%слово%'` — катастрофа. FULLTEXT решает эту проблему.

---

## 🔹 4. Создание FULLTEXT-индекса

```sql
CREATE FULLTEXT INDEX ft_articles ON articles(title, body);
```

Убедимся, что он появился:

```sql
SHOW INDEX FROM articles;
```

В списке появится `ft_articles` с `Index_type = FULLTEXT`.

> 💡 **Почему `FULLTEXT` — это отдельный тип:** он хранит не значения столбцов, а **перевёрнутый индекс** (inverted index): для каждого слова — список документов (строк), в которых оно встречается. Это позволяет мгновенно находить строки по слову, а не перебирать весь текст.

---

## 🔹 5. Режим естественного языка (NATURAL LANGUAGE MODE)

### 5.1. Простейший поиск ?

```sql
SELECT id, title,
       MATCH(title, body) AGAINST ('replication') AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('replication')
ORDER BY score DESC;
```

**Результат:**

| id | title | score |
|----|-------|-------|
| 2 | MySQL replication | 0.0906... |
| 1 | MySQL FULLTEXT search | ... |
| ... | ... | ... |

**Что происходит:**
- `MATCH(title, body) AGAINST ('replication')` — ищет слово `replication` в обоих столбцах.
- `score` — **релевантность**: чем чаще слово встречается в документе (и чем реже — во всей коллекции), тем выше оценка.
- **InnoDB не применяет 50%-й порог**, который есть у MyISAM. Слова, встречающиеся чаще чем в 50% строк, **не отбрасываются** (кроме стоп-слов).

### 5.2. Несколько слов

```sql
SELECT id, title, MATCH(title, body) AGAINST ('MySQL replication') AS score
FROM articles
WHERE MATCH(title, body) AGAINST ('MySQL replication')
ORDER BY score DESC;
```

В естественном режиме слова объединяются через **OR**: находятся документы, содержащие **хотя бы одно** из слов.

> 💡 В естественном режиме **порядок слов не важен**, а **регистр** обычно игнорируется (зависит от `collation`).

---

## 🔹 6. Булев режим (BOOLEAN MODE): точный контроль

### 6.1. Синтаксис

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('+MySQL -replication' IN BOOLEAN MODE);
```

### 6.2. Операторы булева режима

| Оператор | Значение | Пример |
|----------|----------|--------|
| `+` | Слово **обязательно** должно присутствовать | `+MySQL` |
| `-` | Слово **должно отсутствовать** | `-Oracle` |
| `*` | **Wildcard** (суффикс) | `replicat*` → replication, replicate, replicating... |
| `"..."` | **Точная фраза** | `"query optimizer"` |
| `~` | **Уменьшить релевантность** (если слово есть) | `~performance` |
| `>` | **Увеличить** вес слова | `>important` |
| `<` | **Уменьшить** вес слова | `<optional` |
| `(...)` | Группировка подвыражений | `+(mysql +(index query))` |

> 💡 **Важно:** в булевом режиме **нет автоматической сортировки по релевантности**. Если нужен порядок — используйте `ORDER BY MATCH(...) AGAINST(...) DESC`.

### 6.3. Примеры

**Найти статьи, где есть `MySQL`, но нет `replication`:**

```sql
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('+MySQL -replication' IN BOOLEAN MODE);
```

**Найти точную фразу `query optimization`:**

```sql
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('"query optimization"' IN BOOLEAN MODE);
```

**Найти слова, начинающиеся на `replicat`:**

```sql
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('replicat*' IN BOOLEAN MODE);
```

**Найти статьи про MySQL-индексы, но не про MyISAM:**

```sql
SELECT id, title FROM articles
WHERE MATCH(title, body) AGAINST ('+MySQL +(index* partition*) -MyISAM' IN BOOLEAN MODE);
```

---

## 🔹 7. Ограничения InnoDB: стоп-слова и длина слова

### 7.1. Минимальная длина слова

InnoDB **игнорирует** слова короче `innodb_ft_min_token_size` (по умолчанию **3**).

```sql
SHOW VARIABLES LIKE 'innodb_ft_min_token_size';
```

**Что это значит:** слово `SQL` (3 символа) будет проиндексировано, а `DB` (2 символа) — **нет**. Если ищете короткие слова — нужно уменьшить параметр в `my.cnf`:

```ini
[mysqld]
innodb_ft_min_token_size = 2
```

⚠️ После изменения параметра нужно **пересоздать индекс**:

```sql
ALTER TABLE articles DROP INDEX ft_articles;
ALTER TABLE articles ADD FULLTEXT INDEX ft_articles (title, body);
```

### 7.2. Стоп-слова

InnoDB имеет **встроенный список стоп-слов** (артикли, предлоги и т.п.). Слова из этого списка **не индексируются** и не находятся.

Проверить, включены ли стоп-слова:

```sql
SHOW VARIABLES LIKE 'innodb_ft_enable_stopword';
```

Если `ON` (по умолчанию) — стоп-слова работают. **InnoDB использует свой список**, а не `ft_stopword_file` (это для MyISAM).

> 💡 Если нужно **отключить** стоп-слова:
> ```sql
> SET GLOBAL innodb_ft_enable_stopword = OFF;
> ```
> И пересоздать индекс.

### 7.3. Максимальная длина слова

Слова длиннее `innodb_ft_max_token_size` (по умолчанию 84) тоже игнорируются. Для большинства задач это не критично.

---

## 🔹 8. Сравнение производительности: `LIKE` vs `FULLTEXT`

Исследования показывают, что на больших наборах данных **FULLTEXT работает в разы быстрее** `LIKE '%...%'`, потому что использует перевёрнутый индекс, а не полный перебор.

| Аспект | `LIKE '%слово%'` | `FULLTEXT` |
|--------|-------------------|------------|
| **Индекс** | ❌ Не используется | ✅ Используется (`type: fulltext`) |
| **Скорость на больших данных** | Медленно (full scan) | Быстро (inverted index) |
| **Поиск по суффиксу** | ✅ Работает | ⚠️ Только `*` в булевом режиме |
| **Релевантность** | ❌ Нет | ✅ Есть (score) |
| **Стоп-слова** | ❌ Не учитываются | ✅ Учитываются (InnoDB) |

**Проверим `EXPLAIN` для FULLTEXT:**

```sql
EXPLAIN SELECT * FROM articles WHERE MATCH(title, body) AGAINST ('replication');
```

**Результат:**

| type | key | rows | Extra |
|------|-----|------|-------|
| **fulltext** | ft_articles | 1 | Using where; Ft_hints: sorted |

`type: fulltext`, `key: ft_articles` — индекс используется.

> 💡 **Ограничение:** `MATCH ... AGAINST` в `WHERE` **не может** использовать другие индексы одновременно (если только не `FORCE INDEX`). Это «или FULLTEXT, или ничего».

---

## 🔹 9. Расширение запроса (WITH QUERY EXPANSION)

```sql
SELECT id, title
FROM articles
WHERE MATCH(title, body) AGAINST ('replication' WITH QUERY EXPANSION);
```

MySQL выполняет поиск **дважды**: сначала по исходному слову, затем — по словам из найденных документов. Это может **расширить** результаты (появляются статьи, где нет слова `replication`, но есть связанные термины).

> 💡 Полезно, когда пользователь вводит слишком узкий запрос. Минус — может добавить «шум».

---

## 🔹 10. Ведение и экспорт лога действий пользователя

Все команды записываются в `/var/lib/mysql/general.log`.

### 10.1. Проверить, что лог пишется

```sql
\q
```
Показать налицие файла лога в контейнере:
```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```

### 10.2. Отфильтровать только свои действия

```shell
docker exec my-mysql cat /var/lib/mysql/general.log \
  | grep -E "Query[[:space:]]+(CREATE|INSERT|SELECT|UPDATE|DELETE|ALTER|DROP|EXPLAIN|SHOW|USE|SET)" \
  | grep -v -E "INSERT INTO help_|INSERT INTO time_zone|INSERT INTO mysql\.|SELECT \* FROM mysql\.general_log|SET NAMES|SET autocommit|SET sql_mode|SET character_set_results" \
  > my_actions.log
```

Проверить:

```shell
cat my_actions.log
```

### 10.3. Забрать полный лог на хост

```shell
docker cp my-mysql:/var/lib/mysql/general.log ./general.log
```

### 10.4. Очистить лог

```shell
docker exec -u mysql my-mysql sh -c ': > /var/lib/mysql/general.log'
```

---

## 🔹 11. Создание дампа базы

```sql
\q
```

```shell
docker exec -e MYSQL_PWD=123 my-mysql mysqldump -uroot \
  --set-gtid-purged=OFF \
  --single-transaction \
  fulltext_practice > fulltext_dump.sql
```

Посмотрите содержимое (первые 30 строк):

```shell
head -n 30 fulltext_dump.sql
```

> ⚠️ В дамп попадёт **и FULLTEXT-индекс** (`ft_articles`). При восстановлении он будет создан автоматически.

---

## 📌 Итог

| Что важно запомнить о FULLTEXT |
|----------------------------------|
| `FULLTEXT` — это **перевёрнутый индекс** (inverted index), а не B-дерево |
| Работает только для `CHAR`, `VARCHAR`, `TEXT` в InnoDB и MyISAM |
| `MATCH(col) AGAINST('слово')` — основной синтаксис |
| **Режимы:** `NATURAL LANGUAGE` (по умолчанию, с релевантностью), `BOOLEAN` (с операторами `+`, `-`, `*`, `"..."`, `~`), `WITH QUERY EXPANSION` (расширение) |
| **InnoDB:** нет 50%-го порога, но есть стоп-слова и минимальная длина слова (`innodb_ft_min_token_size`, по умолчанию 3) |
| `LIKE '%слово%'` **не использует** индекс, `FULLTEXT` — использует |
| На больших данных `FULLTEXT` **значительно быстрее** `LIKE` |
| `MATCH ... AGAINST` в `WHERE` **не комбинируется** с другими индексами |

---

**Результат оформить в виде отдельного `README.md` со скриншотами `EXPLAIN` (до и после создания FULLTEXT-индекса), результатами поиска в разных режимах, дампом и логом (лог-файл должен содержать только ваши действия) в этой же папке и прислать ссылку преподавателю.**

---

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!