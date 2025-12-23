### Домашнее задание: Словари + оконные функции

#### Цель

Закрепить навыки:
* работы со словарями (через `DDL`),
* оконных функций (`sum() OVER`),
* использования `dictGet` и `LowCardinality`.

⸻

Шаг 1. Создание основной таблицы

CREATE TABLE actions
(
    user_id UInt64,
    action LowCardinality(String),
    expense UInt64
)
ENGINE = MergeTree
ORDER BY user_id;


⸻

Шаг 2. Наполнение таблицы данными

INSERT INTO actions VALUES
(1, 'click', 100),
(1, 'click', 200),
(1, 'buy', 300),
(2, 'click', 50),
(2, 'buy', 150),
(2, 'buy', 100),
(3, 'click', 70),
(3, 'click', 30),
(3, 'buy', 90);


⸻

Шаг 3. Создание справочной таблицы с email

CREATE TABLE user_emails
(
    user_id UInt64,
    email String
)
ENGINE = TinyLog;

INSERT INTO user_emails VALUES
(1, 'user1@example.com'),
(2, 'user2@example.com'),
(3, 'user3@example.com');


⸻

Шаг 4. Создание словаря на основе таблицы

CREATE DICTIONARY user_email_dict
(
    user_id UInt64,
    email String
)
PRIMARY KEY user_id
SOURCE(CLICKHOUSE(
    HOST 'localhost'
    PORT 9000
    USER 'default'
    TABLE 'user_emails'
    DB 'default'
))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 300);


⸻

Шаг 5. Финальный SELECT

SELECT
    dictGet('user_email_dict', 'email', toUInt64(user_id)) AS email,
    action,
    expense,
    sum(expense) OVER (PARTITION BY user_id, action ORDER BY expense ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS expense_cumsum
FROM actions
ORDER BY email;


⸻

🔎 Результат

Запрос возвращает:
	•	email из словаря через dictGet,
	•	аккумулятивную сумму expense по каждому user_id и action,
	•	сортировку по email.

⸻

Если хочешь — добавлю автоочистку/удаление словаря и таблиц в конце.