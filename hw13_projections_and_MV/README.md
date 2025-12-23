#### ✅ Домашнее задание "Проекции и материализованные представления в ClickHouse"

---

##### 1️⃣ Создание таблицы `sales`

```sql
CREATE TABLE sales
(
    id UInt32,
    product_id UInt32,
    quantity UInt32,
    price Float32,
    sale_date DateTime
)
ENGINE = MergeTree
ORDER BY (product_id, sale_date);
```

📌 Почему так:
- `MergeTree` — обязателен для проекций
- ORDER BY (product_id, sale_date):
	* логичен для аналитики по продуктам
	* улучшает локальность данных

---

##### 2️⃣ Наполнение тестовыми данными

```sql
INSERT INTO sales VALUES
(1, 101, 2, 10.0, now() - INTERVAL 3 DAY),
(2, 101, 1, 10.0, now() - INTERVAL 2 DAY),
(3, 102, 5, 7.5,  now() - INTERVAL 1 DAY),
(4, 103, 3, 20.0, now()),
(5, 102, 2, 7.5,  now());
```

---

##### 3️⃣ Создание проекции

🎯 Требование

агрегировать данные по `product_id` и считать общее количество и сумму продаж

📌 Проекция

```sql
ALTER TABLE sales
ADD PROJECTION sales_projection
(
    SELECT
        product_id,
        sum(quantity) AS total_quantity,
        sum(quantity * price) AS total_sales
    GROUP BY product_id
);
```

❗ ВАЖНО: Проекция не строится для старых данных автоматически.

Материализация проекции

```sql
ALTER TABLE sales
MATERIALIZE PROJECTION sales_projection;
```

---

##### 4️⃣ Создание материализованного представления

4.1 Таблица-приёмник (explicit MV)

```sql
CREATE TABLE sales_mv
(
    product_id UInt32,
    total_quantity UInt64,
    total_sales Float64
)
ENGINE = SummingMergeTree
ORDER BY product_id;
```

📌 Почему SummingMergeTree:
* MV пишет инкременты (дельты)
* строки с одинаковым product_id будут физически суммироваться
* стандартный и рекомендуемый паттерн для агрегатов по ключу

📌 Почему это EXPLICIT MV:
* таблица создаётся заранее и отдельно
* она независима от MV
* её можно:
    - ALTER
    - TRUNCATE
	- использовать как обычную таблицу

👉 Именно здесь закладывается explicit-подход.

---

4.2 Материализованное представление

```sql
CREATE MATERIALIZED VIEW sales_mv_view
TO sales_mv
AS
SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales
FROM sales
GROUP BY product_id;
```

📌 Что делает MV:
* является триггером на INSERT в sales
* агрегирует только вставляемый батч
* дописывает результат в sales_mv

📌 Чего MV не делает:
* не хранит данные сам
* не управляет схемой таблицы
* не пересчитывает историю

---

##### 5️⃣ Запросы к данным

🔹 Запрос к таблице `sales`

```sql
SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales
FROM sales
GROUP BY product_id;
```

Результат выполнения запроса ![hw13_sel1](https://github.com/realexpert1C/clickhouse-course/blob/5a4036be2c4f9f918d4463126113cc8e2cac5051/images/hw13_sel1.png)


🔹 Запрос к проекции

⚠️ Проекция используется прозрачно, запрос обычный:

```sql
SET optimize_use_projections = 1;

SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales
FROM sales
GROUP BY product_id;
```

👉 ClickHouse сам выберет `sales_projection

Результат выполнения запроса ![hw13_sel2](https://github.com/realexpert1C/clickhouse-course/blob/5a4036be2c4f9f918d4463126113cc8e2cac5051/images/hw13_sel2.png)

---

🔹 Запрос к материализованному представлению

```sql
SELECT
    product_id,
    total_quantity,
    total_sales
FROM sales_mv;
```
Перед выполнением этого запроса необходимо сделать INSERT в таблицу `sales`. 
В противном случае MV не обновится и будет пустой результат. Поэтому выполняю
`TRUNCATE sales;` и затем повторяю `INSERT INTO sales VALUES ...` из п.2


Результат выполнения запроса ![hw13_sel3](https://github.com/realexpert1C/clickhouse-course/blob/5a4036be2c4f9f918d4463126113cc8e2cac5051/images/hw13_sel3.png)

---

##### 6️⃣ Сравнение производительности

Что сравниваем

|Источник|Что происходит|
|--------|--------------|
|sales|scan + агрегация|
|projection|чтение уже агрегированных данных|
|sales_mv|чтение готовых агрегатов|

Ожидаемый результат

|Источник|Время|CPU|IO|
|--------|-----|---|--|
|sales| ❌ медленно 0.005сек|🔴|🔴|
|projection|🟡 быстрее	0.003сек|🟡|🟢|
|sales_mv|🟢 максимально быстро	0.002сек|🟢|🟢|

📌 MV всегда быстрее, но:
- дороже INSERT
- сложнее сопровождение

---

 #### 🔹 Другие агрегаты - ⭐ Задание со звёздочкой

Разные агрегатные функции в проекциях и материализованных представлениях

---

##### 1️⃣ Проекции: разные агрегаты

📌 Важно помнить про проекции
- проекция — это копия данных / агрегатов внутри той же таблицы
- используется прозрачно
- агрегаты должны полностью совпадать с запросом

---

1.1 Проекция: `sum`, `count`, `avg`

```sql
ALTER TABLE sales
ADD PROJECTION sales_proj_basic
(
    SELECT
        product_id,
        sum(quantity)            AS total_quantity,
        sum(quantity * price)    AS total_sales,
        count()                  AS cnt,
        avg(price)               AS avg_price
    GROUP BY product_id
);
```

Материализация:

```sql
ALTER TABLE sales MATERIALIZE PROJECTION sales_proj_basic;
```

Запрос, который использует проекцию

```sql
SET optimize_use_projections = 1;

SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales,
    count() AS cnt,
    avg(price) AS avg_price
FROM sales
GROUP BY product_id;
```

📌 Если изменить хоть одну агрегацию — проекция не будет использована.

---

1.2 Проекция с `min` / `max`

```sql
ALTER TABLE sales
ADD PROJECTION sales_proj_minmax
(
    SELECT
        product_id,
        min(price) AS min_price,
        max(price) AS max_price
    GROUP BY product_id
);
```
Материализация:

```sql
ALTER TABLE sales MATERIALIZE PROJECTION sales_proj_minmax;
```

Запрос:

```sql
SET optimize_use_projections = 1;

SELECT
    product_id,
    min(price),
    max(price)
FROM sales
GROUP BY product_id;
```

---

1.3 Что будет при INSERT / UPDATE / DELETE

```sql
INSERT INTO sales VALUES
(100, 101, 1, 12.0, now());
```

* ❗ проекция не обновляется сразу
* данные попадут в неё после merge
* `OPTIMIZE TABLE sales FINAL` ускорит обновление

---

##### 2️⃣ Материализованные представления: разные агрегаты

Теперь самое важное отличие:
👉 MV пишет инкременты (только новые INSERT), поэтому не все агрегаты одинаково безопасны.

---

2.1 MV + SummingMergeTree (простые агрегаты)

Таблица-приёмник

```sql
CREATE TABLE sales_mv_sum
(
    product_id UInt32,
    total_quantity UInt64,
    total_sales Float64,
    cnt UInt64
)
ENGINE = SummingMergeTree
ORDER BY product_id;
```

MV

```sql
CREATE MATERIALIZED VIEW sales_mv_sum_view
TO sales_mv_sum
AS
SELECT
    product_id,
    sum(quantity) AS total_quantity,
    sum(quantity * price) AS total_sales,
    count() AS cnt
FROM sales
GROUP BY product_id;
```

📌 Работает корректно, потому что:
- sum, count → аддитивные
- можно складывать инкременты

---

2.2 Почему avg() в SummingMergeTree — ошибка

❌ НЕПРАВИЛЬНО:

avg(price)

Почему:
* среднее неаддитивно
* среднее от средних = ❌

---

2.3 Правильный avg() через AggregatingMergeTree

Таблица

```sql
CREATE TABLE sales_mv_agg
(
    product_id UInt32,
    avg_price_state AggregateFunction(avg, Float32),
    max_price_state AggregateFunction(max, Float32)
)
ENGINE = AggregatingMergeTree
ORDER BY product_id;
```

MV

```sql
CREATE MATERIALIZED VIEW sales_mv_agg_view
TO sales_mv_agg
AS
SELECT
    product_id,
    avgState(price) AS avg_price_state,
    maxState(price) AS max_price_state
FROM sales
GROUP BY product_id;
```

Запрос к MV

```sql
SELECT
    product_id,
    avgMerge(avg_price_state) AS avg_price,
    maxMerge(max_price_state) AS max_price
FROM sales_mv_agg
GROUP BY product_id;
```

📌 Это единственный корректный способ считать avg, uniq, quantile в MV.

---

2.4 MV с `min` / `max`

min и max:
- неаддитивны
- но моноидальны
- работают и в SummingMergeTree, и в AggregatingMergeTree

Рекомендуется всё же AggregatingMergeTree.

---

3️⃣ Как изменения в sales влияют на projection и MV

|Операция|Projection|MV|
|--------|----------|--|
|INSERT|	✔ (после merge)	|✔ сразу|
|DELETE|	✔ (после merge)	|❌|
|UPDATE|	✔ (через merge)	|❌|
|TRUNCATE|	✔	|❌|
|OPTIMIZE|	✔	|❌|

📌 MV — append-only по смыслу

---

4️⃣ Выводы

В проекциях можно использовать любые агрегатные функции, если запрос полностью совпадает с определением проекции.
В материализованных представлениях необходимо учитывать, что они пишут инкременты: для аддитивных функций подходит SummingMergeTree, а для сложных агрегатов (avg, uniq, quantile) требуется AggregatingMergeTree с состояниями агрегатных функций.

---