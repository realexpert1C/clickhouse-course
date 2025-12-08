## Домашнее заданеие: "Движки MergeTree Family"

#### Цель
	•	Изучить принципы работы движка MergeTree в ClickHouse
	•	Рассмотреть выбор движка для различных задач
	•	Научиться дедуплицировать данные и заменять операции delete/update

⸻

#### Задание 1: Таблица tbl1

CREATE TABLE tbl1
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8,
    Version UInt8
)
ENGINE = CollapsingMergeTree(Sign)
ORDER BY UserID;

INSERT INTO tbl1 VALUES (4324182021466249494, 5, 146, -1, 1);
INSERT INTO tbl1 VALUES (4324182021466249494, 5, 146, 1, 1),
                         (4324182021466249494, 6, 185, 1, 2);

SELECT * FROM tbl1;
SELECT * FROM tbl1 FINAL;

Пояснение:
Выбран движок CollapsingMergeTree, так как используется колонка Sign для дедупликации данных по противоположным знакам.

<!-- Вставить скриншот/результат SELECT * FROM tbl1 -->


<!-- Вставить скриншот/результат SELECT * FROM tbl1 FINAL -->



⸻

#### Задание 2: Таблица tbl2

CREATE TABLE tbl2
(
    key UInt32,
    value UInt32
)
ENGINE = ReplacingMergeTree()
ORDER BY key;

INSERT INTO tbl2 VALUES (1,1), (1,2), (2,1);
SELECT * FROM tbl2;

Пояснение:
Использован ReplacingMergeTree() — по умолчанию выбирает последнюю версию строки по ORDER BY, подходит для логики «замены» строк.

<!-- Вставить скриншот SELECT * FROM tbl2 -->



⸻

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