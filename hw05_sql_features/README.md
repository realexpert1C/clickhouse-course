
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

```sql
ALTER TABLE menu DELETE WHERE item_id = 2;
INSERT INTO menu VALUES (2, 'Цезарь с курицей', 'Салаты', 390.00, 1);
```

* Delete:

```sql
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

_📌 Вывод: действия CRUD отражаются в логах с указанием времени и объема данных (см. [скриншот6](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_3_6.png), [скриншот7](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_3_7.png), [скриншот8](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_3_8.png), [скриншот9](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_3_9.png), [скриншот10](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_3_10.png))._

---
### 🧱 Шаг 4. Изменение структуры таблицы

```sql
ALTER TABLE menu ADD COLUMN description Nullable(String);
ALTER TABLE menu ADD COLUMN calories UInt16;
ALTER TABLE menu DROP COLUMN available;
ALTER TABLE menu DROP COLUMN category;
```

_📊 Проверка через system.columns:_

```sql
SELECT name, type FROM system.columns WHERE table = 'menu';
```

_📌 Вывод: структура таблицы успешно изменена (см. [скриншот11](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_4_11.png), [скриншот12](https://github.com/realexpert1C/clickhouse-course/blob/main/images/step3_4_12.png))._

--- 

### 📦 Шаг 5. Выборка из sample dataset

```sql
SELECT * FROM system.numbers LIMIT 1000;
```
_📊 Проверка:_
```sql
SELECT name, total_rows, total_bytes FROM system.tables WHERE name = 'numbers';
```
--- 
### 🧬 Шаг 6. Материализация таблицы

```sql
CREATE TABLE numbers_copy ENGINE = MergeTree ORDER BY number AS
SELECT * FROM system.numbers LIMIT 1000;
```
_📊 Сравнение по system.tables:_
```sql
SELECT name, total_rows, total_bytes FROM system.tables WHERE name IN ('numbers', 'numbers_copy');
```
_📌 Вывод: структура и объем скопированы._

--- 
### 🧠 Шаг 7: Создание и использование Materialized View

Создаём лог по дорогим блюдам:
```sql
CREATE TABLE expensive_log
(
    id UInt32,
    name String,
    price Float32
) ENGINE = MergeTree
ORDER BY id;
```
```sql
CREATE MATERIALIZED VIEW log_mv TO expensive_log AS
SELECT id, name, price FROM menu WHERE price > 1000;
```

_📊 Проверка автоматической вставки:_
```sql
INSERT INTO menu VALUES (4, 'Рибай', 'Горячее', 1800, 1);
```
```sql
SELECT * FROM expensive_log;
```
_📊 Контроль через:_
```sql
SELECT * FROM system.query_log WHERE query ILIKE '%INSERT%' AND table = 'expensive_log';
```
---
### 🧩 Шаг 8. Работа с партициями

```sql
-- Пример партиционированной таблицы
CREATE TABLE menu_partitioned
(
    item_id UInt32,
    name String,
    category String,
    created_at Date
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(created_at)
ORDER BY item_id;
```
```sql
INSERT INTO menu_partitioned VALUES (4, 'Пельмени', 'Горячее', '2025-07-10');
```
```sql
ALTER TABLE menu_partitioned DETACH PARTITION 202507;
ALTER TABLE menu_partitioned ATTACH PARTITION 202507;
ALTER TABLE menu_partitioned DROP PARTITION 202507;
```
_📊 Проверка партиций:_
```sql
SELECT partition, name, active FROM system.parts WHERE table = 'menu_partitioned';
```
_📌 Вывод: изменения с партициями видны и отслеживаются._

---
### 📥 Дополнительно: Мониторинг ресурсов

```sql
SELECT ProfileEvents['DiskWriteElapsedMicroseconds'] AS disk_write_time
FROM system.query_log
WHERE query ILIKE '%INSERT%' ORDER BY event_time DESC LIMIT 1;
```
---
### 📌 Заключение

Каждый шаг работы с SQL в ClickHouse можно и нужно сопровождать обращениями к системным таблицам, которые позволяют оценить состояние, поведение запросов и объемы обрабатываемых данных. Это важный навык при построении производительных и прозрачных решений.



# Конспект лекции "Язык запросов SQL в ClickHouse"

## 📌 Основная информация
**Преподаватель**: Алексей Железной (Senior Data Engineer, OTUS)  
**Формат**:  
- Активные обсуждения в чате (#OTUS ClickHouse-2024-04)  
- Вопросы через чат/голос  

---

## 🔍 Ключевые темы
### 1. Диалекты ClickHouse
- Поддерживает **SQL** (классический) и **PRQL** (Pipelined Relational Query Language)  
- Переключение:  
  ```sql
  SET dialect='clickhouse'; -- по умолчанию
  SET dialect='prql';

* Особенности SQL в ClickHouse:

- Нет стандартных UPDATE/DELETE → используются ALTER TABLE ... UPDATE|DELETE

- Пример PRQL → SQL преобразования:
  ```sql
  from sales | filter date >= '2023-01-01' | group product_id (total_quantity = sum quantity)
  ```
---
### 2. CRUD-операции
   
__Операция	Особенности ClickHouse__

`SELECT`	  Полная поддержка, сложные агрегации
```sql
SELECT 
    product_id,
    SUM(quantity) AS total_quantity
FROM sales
WHERE date >= '2023-01-01'
GROUP BY product_id
FORMAT Vertical;
```

`INSERT`	  Стандартный синтаксис
```sql
CREATE TABLE test (id UInt32) ENGINE=MergeTree ORDER BY id;
INSERT INTO test VALUES (1), (2);
```

`UPDATE`  	Мутация данных (перезапись партиций) → требует 2x места
```sql
ALTER TABLE test 
UPDATE id = 3 
WHERE id = 1;
```

`DELETE`  	Аналогично UPDATE, не рекомендуется для частых изменений
```sql
ALTER TABLE test 
DELETE WHERE id = 2;
```

---
### 3. Создание/удаление БД и таблиц
* Базы данных:
```sql
CREATE DATABASE db_name ENGINE=Atomic;  -- Recommended
DROP DATABASE db_name SYNC;            -- Мгновенное удаление
```
* Таблицы:
```sql
CREATE TABLE logs (date Date, message String) ENGINE=MergeTree ORDER BY date;
```
---
### 4. Модификация таблиц
* Добавление столбцов:
```sql
ALTER TABLE logs ADD COLUMN user_id String AFTER date;
```
* Удаление данных:
```sql
ALTER TABLE logs CLEAR COLUMN message;
```
---
### 5. Работа с партициями
```sql
ALTER TABLE logs DROP PARTITION '2023-01-01';  -- Удаление
ALTER TABLE logs ATTACH PARTITION '2023-01-01'; -- Восстановление
```

--- 
### ⚠️ Важные ограничения ClickHouse
1. Мутации (UPDATE/DELETE):

* Требуют 2x места от размера данных

* Выполняются перезаписью партиций

2. Рекомендации:

* Используйте ReplacingMergeTree для частых обновлений

* Избегайте точечных изменений в больших таблицах

### 📝 Проверка знаний
1. Какой синтаксис предпочтительнее для аналитики: _SQL_ или _PRQL_?

2. Почему `Atomic` - рекомендуемый движок для БД?

3. Чем `CLEAR COLUMN` отличается от `DROP COLUMN`?

🔗 Полезные ссылки
- [Документация ClickHouse](https://clickhouse.com/docs/ru)

- [Материалы курса OTUS](https://otus.ru/lessons/clickhouse/)

- [PRQL документация](https://prql-lang.org/)


