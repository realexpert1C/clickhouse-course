### Домашнее задание: Словари + оконные функции

#### Цель

Закрепить навыки:
* работы со словарями (через `DDL`),
* оконных функций (`sum() OVER`),
* использования `dictGet` и `LowCardinality`.

---

##### Шаг 1. Создание основной таблицы

```sql
CREATE TABLE actions
(
    user_id UInt64,
    action LowCardinality(String),
    expense UInt64
)
ENGINE = MergeTree
ORDER BY user_id;
```

---

##### Шаг 2. Наполнение таблицы данными

```sql
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
```

---

##### Шаг 3. Создание справочной таблицы с email

```sql
CREATE TABLE user_emails
(
    user_id UInt64,
    email String
)
ENGINE = TinyLog;
```

```sql
INSERT INTO user_emails VALUES
(1, 'user1@example.com'),
(2, 'user2@example.com'),
(3, 'user3@example.com');
```

---

##### Шаг 4. Создание словаря на основе таблицы

```sql
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
    PASSWORD '*******'
    TABLE 'user_emails'
    DB 'default'
))
LAYOUT(HASHED())
LIFETIME(MIN 60 MAX 300);
```
--- 

##### Шаг 5. Финальный SELECT

```sql
SELECT
    dictGet('user_email_dict', 'email', toUInt64(user_id)) AS email,
    action,
    expense,
    sum(expense) OVER (PARTITION BY user_id, action ORDER BY expense ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS expense_cumsum
FROM actions
ORDER BY email;
```

---

#### 🔎 Результат

Результат выполнения запроса ![hw11_sel1](https://github.com/realexpert1C/clickhouse-course/blob/e7c7e265b249bf22640aad6ca8902586023b6cde/images/hw11_sel1.png)

Запрос возвращает:
* email из словаря через dictGet,
* аккумулятивную сумму expense по каждому user_id и action,
* сортировку по email.

---