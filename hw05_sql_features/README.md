## 🧩 Домашнее задание: Работа с SQL в ClickHouse

---

🎯 __Цель__:
* отработать основные sql-операции;
* изучить на практике особенности sql-диалекта;

---

### 🔧 Шаг 1: Создание БД и переход

```sql
CREATE DATABASE restaurant_db;

USE restaurant_db;
```

_📊 Анализ через системные таблицы:_
```sql
SELECT * FROM system.databases WHERE name = 'restaurant_db';
```

_📌 Вывод: база данных создана с движком Atomic, см. [скриншот1](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_1_1.png), [скриншот2](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_1_2.png)_

---

### 🍽️ Шаг 2. Создание таблицы "Меню ресторана"

```sql
CREATE TABLE menu
(
    item_id UInt32 COMMENT 'Уникальный идентификатор блюда',
    name String COMMENT 'Название блюда',
    category LowCardinality(String) COMMENT 'Категория блюда',
    price Decimal(8,2) COMMENT 'Цена',
    available Nullable(Bool) COMMENT 'Доступность для заказа'
)
ENGINE = MergeTree
ORDER BY item_id;
```

```sql
INSERT INTO menu VALUES
(1, 'Борщ', 'Супы', 290.00, 1),
(2, 'Цезарь', 'Салаты', 340.00, 1),
(3, 'Стейк', 'Горячее', 850.00, 0);
```

_📊 Проверка через system.tables и columns:_

```sql
SELECT name, engine, total_rows FROM system.tables WHERE database = 'restaurant_db';
SELECT * FROM system.columns WHERE table = 'menu';
```

_📌 Вывод: таблица создана, типы колонок подтверждены (см. [скриншот3](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_3.png), [скриншот4](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_4.png), [скриншот5](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_5.png))._

---

### ✍️ Шаг 3: CRUD-операции

* Read:

```sql
SELECT * FROM menu;
```

* Update (через DELETE + INSERT):

```
ALTER TABLE menu DELETE WHERE item_id = 2;
INSERT INTO menu VALUES (2, 'Цезарь с курицей', 'Салаты', 390.00, 1);
```

* Delete:

```
ALTER TABLE menu DELETE WHERE item_id = 3;
```

_📊 Анализ через system и через select * from menu:_

```sql
SELECT query, type, event_time, read_rows, result_rows
FROM system.query_log
WHERE query LIKE '%menu%'
AND type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 5;
```

_📌 Вывод: действия CRUD отражаются в логах с указанием времени и объема данных (см. [скриншот6](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_3.png), [скриншот7](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_4.png), [скриншот8](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_5.png), [скриншот9](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_5.png), [скриншот10](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_2_5.png))._







