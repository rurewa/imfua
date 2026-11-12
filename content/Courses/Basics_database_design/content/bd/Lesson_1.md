# SQL-команды для создания и заполнения нормализованных таблиц

> Если вы уверены, что на хосте нет других важных контейнеров и их volumes (тома с данными), можно использовать команды для их удаления:
```shell
docker stop $(docker ps -q)
docker container prune
docker volume prune -a
```
> Но **будьте осторожны** — они удаляют всё неиспользуемое в Docker, а не только учебный проект.

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
Параметры `general_log = 1` и `general_log_file = /var/lib/mysql/general.log` нужны для того, чтобы **MySQL** вела файл журнала записей происходящего внутри базы данных (лог).

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
Чтобы убедиться, что контейнер успешно запущен, выполните:
```bash
docker ps
```
Вы должны увидеть свой контейнер **my-mysql** в списке работающих.

Перед началом работы (сразу после создания контейнера) очистить лог:
```shell
docker exec my-mysql sh -c ': > /var/lib/mysql/general.log'
```
Выполните следующую команду, чтобы открыть интерактивную оболочку **MySQL** (Git-Bash, Power Shell):
```bash
docker exec -it my-mysql mysql --default-character-set=utf8mb4 -uroot -p
```
Введите пароль `123`. Вы увидите приглашение `mysql>`.

Получить список имеющихся баз данных в контейнере с MySQL:
```sql
SHOW DATABASES;
```
Создадим базу данных с именем `sales` с таблицей `users`. Рекомендуется сразу указывать кодировку:
```sql
CREATE DATABASE sales
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
USE sales;
```

Создадим четыре таблицы в базе данных **sales**

> **Важно:** порядок создания и заполнения — от независимых таблиц к зависимым:
> 1. `customers` (покупатели)
> 2. `products` (товары)
> 3. `reports` (отчёты)
> 4. `report_items` (реализованные товары)

## 🔹 1. Таблица «Покупатели» (`customers`)

### Создание
```sql
CREATE TABLE customers (
    customer_id   INT PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
### Заполнение
```sql
INSERT INTO customers (customer_id, customer_name) VALUES
  (12, 'Дания'),
  (23, 'Испания'),
  (25, 'Албания');
```

## 🔹 2. Таблица «Товары» (`products`)

### Создание
```sql
CREATE TABLE products (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price        INT NOT NULL
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Заполнение
```sql
INSERT INTO products (product_id, product_name, price) VALUES
  (101, 'Дыня',     800),
  (102, 'Клубника', 150),
  (103, 'Яблоко',   120),
  (104, 'Лимон',    200);
```

## 🔹 3. Таблица «Отчёты» (`reports`)

### Создание
```sql
CREATE TABLE reports (
    report_id   INT PRIMARY KEY,
    report_date DATE NOT NULL,
    customer_id INT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
> Даты указаны как `05.03`, `07.03` и т.д. В SQL используем тип `DATE` и подставляем год `2026` (как в предыдущих примерах). Если год не важен, можно заменить `DATE` на `VARCHAR(5)` и хранить `'05.03'` как строку.

### Заполнение
```sql
INSERT INTO reports (report_id, report_date, customer_id) VALUES
  (1101, '2026-03-05', 12),
  (1102, '2026-03-07', 23),
  (1103, '2026-03-08', 25),
  (1104, '2026-03-10', 12),
  (1105, '2026-03-12', 25);
```

## 🔹 4. Таблица «Реализованные товары» (`report_items`)

### Создание
```sql
CREATE TABLE report_items (
    report_id  INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL,
    PRIMARY KEY (report_id, product_id),
    FOREIGN KEY (report_id)  REFERENCES reports(report_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Заполнение
```sql
INSERT INTO report_items (report_id, product_id, quantity) VALUES
  (1101, 101, 1100),
  (1101, 102,  300),
  (1102, 103, 1700),
  (1103, 104,  500),
  (1104, 101, 2500),
  (1105, 103, 2000),
  (1105, 104,  700);
```

## 📌 Проверка

После выполнения всех команд можно проверить данные:

```sql
SELECT * FROM customers;
SELECT * FROM products;
SELECT * FROM reports;
SELECT * FROM report_items;
```
Или выполнить запрос, который восстанавливает исходную «плоскую» (ненормализованную) таблицу:
```sql
SELECT r.report_id, r.report_date, c.customer_name,
       p.product_name, p.price, ri.quantity
FROM reports r
JOIN customers c   ON r.customer_id = c.customer_id
JOIN report_items ri ON r.report_id = ri.report_id
JOIN products p    ON ri.product_id = p.product_id
ORDER BY r.report_id, p.product_id;
```
**Результат:**

| report_id | report_date | customer_name | product_name | price | quantity |
|-----------|-------------|---------------|--------------|-------|----------|
| 1101      | 2026-03-05  | Дания         | Дыня         | 800   | 1100     |
| 1101      | 2026-03-05  | Дания         | Клубника     | 150   | 300      |
| 1102      | 2026-03-07  | Испания       | Яблоко       | 120   | 1700     |
| 1103      | 2026-03-08  | Албания       | Лимон        | 200   | 500      |
| 1104      | 2026-03-10  | Дания         | Дыня         | 800   | 2500     |
| 1105      | 2026-03-12  | Албания       | Яблоко       | 120   | 2000     |
| 1105      | 2026-03-12  | Албания       | Лимон        | 200   | 700      |

Это полностью соответствует исходной ненормализованной таблице.

### Получить лог

Файл лога лежит внутри контейнера по пути `/var/lib/mysql/general.log`.

Покинуть контейнер:
```sql
\q
```
Проверить, что файл существует:
```shell
docker exec my-mysql ls -la /var/lib/mysql/general.log
```
Заглянуть в него:
```shell
docker exec my-mysql cat /var/lib/mysql/general.log
```
В логе много служебных записей (`Connect`, `Quit`, `SET NAMES`, `SHOW VARIABLES` и т.п.). Отфильтровать их можно одной командой на хосте (Git-Bash):
```shell
docker exec my-mysql cat /var/lib/mysql/general.log \
  | grep "Query" \
  | grep -v -E "SET NAMES|SET autocommit|SET sql_mode|SET character_set_results|SHOW VARIABLES|SHOW STATUS|SHOW WARNINGS|SELECT @@|SELECT DATABASE\(\)|time_zone" \
  > my_actions.log
```
Power Shell
```shell
docker exec my-mysql cat /var/lib/mysql/general.log `
  | grep "Query" `
  | grep -v -E "SET NAMES|SET autocommit|SET sql_mode|SET character_set_results|SHOW VARIABLES|SHOW STATUS|SHOW WARNINGS|SELECT @@|SELECT DATABASE\(\)|time_zone" `
  > my_actions.log
```
Теперь в файле `my_actions.log` — только ваши осмысленные SQL-команды.

Проверить (Git-Bash):
```shell
cat my_actions.log
```

---

## 📌 Итог

| Таблица | Команда создания | Команда заполнения |
|---------|------------------|---------------------|
| `customers` | `CREATE TABLE customers (...)` | `INSERT INTO customers ...` |
| `products`  | `CREATE TABLE products (...)`  | `INSERT INTO products ...`  |
| `reports`   | `CREATE TABLE reports (...)`   | `INSERT INTO reports ...`   |
| `report_items` | `CREATE TABLE report_items (...)` | `INSERT INTO report_items ...` |

Порядок создания и заполнения — строго от `customers` и `products` к `reports` и `report_items`, чтобы внешние ключи не нарушались.

> Если вы обнаружили ошибку в этом тексте — сообщите пожалуйста автору!