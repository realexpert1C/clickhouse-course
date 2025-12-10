## Домашнее задание: "Движки MergeTree Family"

### Цель
1. По заданным описаниям таблиц и вставки данных определить
используемый движок
1. Заполнить пропуски, запустить код
2. Сравнить полученный вывод и результат из условия

---

#### Задание 1: Таблица tbl1

```SQL
CREATE TABLE tbl1
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8,
    Version UInt8
)
ENGINE = VersionedCollapsingMergeTree(Sign, Version)
ORDER BY UserID;

INSERT INTO tbl1 VALUES (4324182021466249494, 5, 146, -1, 1);
INSERT INTO tbl1 VALUES (4324182021466249494, 5, 146, 1, 1),
                         (4324182021466249494, 6, 185, 1, 2);

SELECT * FROM tbl1;
SELECT * FROM tbl1 FINAL;
```

Пояснение:

Выбран движок __`VersionedCollapsingMergeTree`__, который объединяет логику Replacing и Collapsing.

__Результат выполнения вышеуказанных запросов на развернутом в предыдущих этапах сервере Clickhouse:__

1. Результат `SELECT * FROM tbl1;`
![SELECT * FROM tvl1;](https://github.com/realexpert1C/clickhouse-course/blob/49421520179ee1d15c13da04ef7a01638d40d2e3/images/hw07_task1_r.png)


2. Результат `SELECT * FROM tbl1 FINAL;`
![SELECT * FROM tvl1 FINAL;](https://github.com/realexpert1C/clickhouse-course/blob/823afe56e6b23d5f54bd9ce8576e704f9b89bf7c/images/hw07_Sel_FINAL.png)

---
Пояснение

Дано:

INSERT 1:
(4324..., 5, 146, -1, 1)

INSERT 2:
(4324..., 5, 146,  1, 1),
(4324..., 6, 185,  1, 2)

Вывод по условиям ДЗ:

`SELECT * FROM tbl1;`

- 5 146 1 1
- 6 185 1 2
- 5 146 -1 1

(три строки: две положительных, одна отрицательная)

SELECT * FROM tbl1 FINAL

6 185 1 2

(одна строка — только самая свежая версия)

---

__Ключевое наблюдение__

В SELECT без FINAL:

✔ строки НЕ схлопываются по Sign
✔ строки НЕ выбирают последнюю версию
→ значит НЕ CollapsingMergeTree и НЕ ReplacingMergeTree

В SELECT FINAL:

✔ остаётся максимум по версии

→ значит применяется правило: «при конфликте — оставить запись с максимальной версией»

➡ Единственный движок MergeTree, который делает это:

🎯 VersionedCollapsingMergeTree(Sign, Version)

---

🟢 1. Почему не CollapsingMergeTree(Sign)?

Потому что CollapsingMergeTree удалил бы пару (-1,1) и (+1,1).

Но в ДЗ до FINAL видны обе строки:

5 146 1 1  
5 146 -1 1

При Collapsing такого быть не может.

---

🟢 2. Почему не ReplacingMergeTree?

ReplacingMergeTree выбирает последнюю версию в каждом парте.

Но в ДЗ до FINAL обе версии видны, значит Replacing не подходит.

---

🟢 3. Почему VersionedCollapsingMergeTree подходит?

✔ До FINAL он показывает все строки, включая -1
✔ FINAL:
* выбирает последнюю версию
* применяет коллапс по Sign
* и оставляет только (Ver = max, Sign = 1)

Ровно так и показано в условиях ДЗ.


📌 Вывод

|Движок|Подходит?|Причина|
|------|---------|-------|
|CollapsingMergeTree|❌|Удалил бы пару (-1,+1), но она видна|
|ReplacingMergeTree|❌|Не показывает две версии одновременно|
|VersionedCollapsingMergeTree|✅|Полное совпадение с выводом ДЗ|


---

#### Задание 2: Таблица tbl2

```SQL
CREATE TABLE tbl2
(
    key UInt32,
    value UInt32
)
ENGINE = SummingMergeTree()
ORDER BY key;

INSERT INTO tbl2 VALUES (1,1), (1,2), (2,1);
SELECT * FROM tbl2;
```

Пояснение:
Использован __`SummingMergeTree()`__ — по умолчанию складывает значения всех числовых столбцов, не входящих в ORDER BY, для строк с одинаковым ключом в момент слияния партиций.

Результат выполнения запроса ```SELECT * FROM tbl2;```

![Result SummingMerge]()<!-- Вставить скриншот  -->

И он совпадает с представленным результатом в условяих ДЗ.

---

#### Задание 3: Таблица tbl3

CREATE TABLE tbl3
(
    id Int32,
    status String,
    price String,
    comment String
)
ENGINE = ReplacingMergeTree()
ORDER BY (id, status);

INSERT INTO tbl3 VALUES (23, 'success', '1000', 'Confirmed');
INSERT INTO tbl3 VALUES (23, 'success', '2000', 'Cancelled');

SELECT * FROM tbl3 WHERE id=23;
SELECT * FROM tbl3 FINAL WHERE id=23;

Пояснение:
Для замены строк используется ReplacingMergeTree. Без указания версии — будет выбрана одна из версий произвольно (не гарантировано, что последняя).

<!-- Вставить результаты SELECT и SELECT FINAL -->



⸻

#### Задание 4: Таблица tbl4

CREATE TABLE tbl4
(
    CounterID UInt8,
    StartDate Date,
    UserID UInt64
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(StartDate)
ORDER BY (CounterID, StartDate);

INSERT INTO tbl4 VALUES(0, '2019-11-11', 1);
INSERT INTO tbl4 VALUES(1, '2019-11-12', 1);

Пояснение:
Обычный MergeTree используется, так как агрегирования и дедупликации не требуется.

⸻

#### Задание 5: Таблица tbl5 (с агрегатной функцией)

CREATE TABLE tbl5
(
    CounterID UInt8,
    StartDate Date,
    UserID AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(StartDate)
ORDER BY (CounterID, StartDate);

INSERT INTO tbl5
SELECT CounterID, StartDate, uniqState(UserID)
FROM tbl4
GROUP BY CounterID, StartDate;

INSERT INTO tbl5 VALUES (1,'2019-11-12',1);

SELECT uniqMerge(UserID) AS state
FROM tbl5
GROUP BY CounterID, StartDate;

Пояснение:
Использован AggregatingMergeTree — предназначен для хранения агрегатных функций State, с последующим объединением через Merge.

<!-- Вставить результат SELECT uniqMerge -->



⸻

#### Задание 6: Таблица tbl6 (дедупликация через знак)

CREATE TABLE tbl6
(
    id Int32,
    status String,
    price String,
    comment String,
    sign Int8
)
ENGINE = CollapsingMergeTree(sign)
ORDER BY (id, status);

INSERT INTO tbl6 VALUES (23, 'success', '1000', 'Confirmed', 1);
INSERT INTO tbl6 VALUES (23, 'success', '1000', 'Confirmed', -1),
                         (23, 'success', '2000', 'Cancelled', 1);

SELECT * FROM tbl6;
SELECT * FROM tbl6 FINAL;

Пояснение:
Использован CollapsingMergeTree(sign) — реализует логику отмены/удаления по знаку. После FINAL остаются только строки без пары противоположного знака.

<!-- Вставить скриншоты SELECT и SELECT FINAL -->



⸻

Проблемы и решения
	•	В случае с ReplacingMergeTree без указания колонки версии — порядок замены не гарантирован.
	•	Для CollapsingMergeTree важно учитывать, что FINAL-select затратно по ресурсам.
	•	Для AggregatingMergeTree важно использовать AggregateFunction-типы данных и правильно применять uniqState, uniqMerge.

⸻

Источники
	•	[ClickHouse MergeTree Engines](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/)
	•	Tutorial￼
	•	AliCloud Engine Guide￼

⸻