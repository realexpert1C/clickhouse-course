# Домашнее задание - Мутации данных и манипуляции с партициями

### 📄 Текст задания

> 
>
> **Цель:**
> - понять, как работают мутации данных в ClickHouse;
> - научиться управлять партициями таблиц и выполнять операции с ними;
>
> **Описание / шаги:**
> 1. Создайте таблицу user_activity с полями:
> - user_id (UInt32) — идентификатор пользователя
> - activity_type (String) — тип активности (например, 'login', 'logout', 'purchase')
> - activity_date (DateTime) — дата и время активности
> - Используйте MergeTree как движок таблицы и настройте партиционирование по дате активности (activity_date).
> 2. Заполните таблицу:
> - Вставьте несколько записей в таблицу user_activity
> - Используйте различные user_id, activity_type и activity_date
> 3. Выполнение мутаций:
> - Выполните мутацию для изменения типа активности у пользователя(-ей)
> 4. Проверка результатов:
> - Напишите запрос для проверки изменений в таблице user_activity
> - Убедитесь, что тип активности у пользователей изменился
> - Приложите логи отслеживания мутаций в системной таблице
> 5. Манипуляции с партициями:
> - Удалите партицию за определённый месяц
> 6. Проверка состояния таблицы:
> - Проверьте текущее состояние таблицы после удаления партиции
> - Убедитесь, что данные за указанный месяц были удалены
>
> 7. Дополнительные задания (по желанию):
> - Исследуйте, как работают другие типы мутаций.
> - Попробуйте создать новую партицию и вставить в неё данные.
> - Изучите возможность использования TTL (Time to Live) для автоматического удаления старых партиций.
> 
> 
> **Критерии оценки:**
>
>Задание считается выполненным, если создана таблица с правильным партиционированием, выполнены мутации данных, проверены изменения, удалена партиция и проверено состояние таблицы после операции.

---

### Шаг 1 - Создание таблицы user_activity

Создаю таблицу с использованием движка MergeTree, партиционирование по дате активности activity_date.

```sql
CREATE TABLE default.user_activity
(
    user_id UInt32,
    activity_type String,
    activity_date DateTime
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(activity_date)
ORDER BY (user_id, activity_date);
```

✅ Скриншот: результат выполнения `SHOW CREATE TABLE user_activity;`.
![hw16_create_tbl](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_create_tbl.png)

---

### Шаг 2 - Заполнение таблицы

Вставляю тестовые данные с различными пользователями и активностями:

```sql
INSERT INTO default.user_activity VALUES 
(1, 'login', '2025-12-01 10:00:00'),
(1, 'logout', '2025-12-01 11:00:00'),
(2, 'purchase', '2025-12-02 15:00:00'),
(3, 'login', '2025-11-15 08:30:00'),
(3, 'purchase', '2025-11-16 09:00:00');
```
✅ Скриншот: результат запроса `SELECT * FROM user_activity ORDER BY activity_date;`.
![hw16_insert_tbl](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_insert_tbl.png)

---

### Шаг 3 - Выполнение мутации

Поменяю тип активности с `login` на `signin`:

```sql
ALTER TABLE default.user_activity
UPDATE activity_type = 'signin'
WHERE activity_type = 'login';
```

✅ Скриншот: `SELECT * FROM user_activity WHERE activity_type = 'signin';`.
![hw16_alter_tbl](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_alter_tbl.png)

---

### Шаг 4 - Проверка мутации

Проверяю статус выполнения:

```sql
SELECT database, table, mutation_id, command, is_done, latest_failed_part
FROM system.mutations
WHERE table = 'user_activity';
```

✅ Скриншот: результат запроса к `system.mutations`.
![hw16_system_mutations](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_system_mutations.png)

---

### Шаг 5 - Удаление партиции

Проверяю список партиций до удаления:

```sql
SELECT partition, min_date, max_date, rows
FROM system.parts
WHERE table = 'user_activity' AND active;
```

✅ Скриншот: `system.parts` до удаления партиции 202511.
![hw16_system_parts_before](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_system_parts_before.png)

Удаляю партицию за ноябрь 2025 (202511):

```sql
ALTER TABLE default.user_activity
DROP PARTITION 202511;
```

✅ Скриншот: результаты `SELECT * FROM user_activity` до и после удаления партиции.
![hw16_user_before_del_part](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_user_before_del_part.png)
![hw16_user_after_del_part](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_user_after_del_part.png)

---

### Шаг 6 - Проверка состояния таблицы

Проверяю список партиций:

```sql
SELECT partition, min_date, max_date, rows
FROM system.parts
WHERE table = 'user_activity' AND active;
```

✅ Скриншот: подтверждение отсутствия партиции 202511.
![hw16_system_parts_after](https://github.com/realexpert1C/clickhouse-course/blob/a57fa07f7267814db03eaca47b265e12a33d9952/images/hw16_system_parts_after.png)

---

### Шаг 7. Дополнительные задания по теме мутаций и работы с партициями в ClickHouse.

---

- Исследование других типов мутаций
- Создание новой партиции и вставка данных
- Использование TTL для автоматического удаления старых данных

---

🔧 Работа с партициями по дате (сутки)

#### 🧱 7.1: Создание новой таблицы с суточным партиционированием

На основе имеющейся тестовой таблицы `default.uk_price_paid` создаю новую таблицу с суточным партиционированием

```sql
CREATE TABLE default.uk_price_paid_daily
AS default.uk_price_paid
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(date)
ORDER BY (postcode1, postcode2, addr1, addr2);
```
---
#### 📥 7.2: Перенос данных за один день (например, 2025-11-01)

```sql
INSERT INTO default.uk_price_paid_daily
SELECT *
FROM default.uk_price_paid
WHERE date = '2025-11-01';
```

Проверка созданной партиции:

```sql
SELECT DISTINCT partition
FROM system.parts
WHERE table = 'uk_price_paid_daily';
```

Скриншот результата запроса
![hw16_add_step2](https://github.com/realexpert1C/clickhouse-course/blob/adfe2e4b3be1a4a49e54924190081db1c79dd9ce/images/hw16_add_step2.png)

---

#### 🧹 7.3: Удаление партиции за 2025-11-01

```sql
ALTER TABLE default.uk_price_paid_daily
DROP PARTITION 20251101;
```
Проверка
```sql
SELECT partition, min_date, max_date, active
FROM system.parts
WHERE table = 'uk_price_paid_daily'
  AND partition = '20251101';
```

Скриншот результата запроса
![hw16_add_step3](https://github.com/realexpert1C/clickhouse-course/blob/adfe2e4b3be1a4a49e54924190081db1c79dd9ce/images/hw16_add_step3.png)

---

#### 🔁 7.4: Повторная вставка данных за тот же день

```sql
INSERT INTO default.uk_price_paid_daily
SELECT *
FROM default.uk_price_paid
WHERE date = '2025-11-01';
```
---

#### 🧩 7.5: DETACH партиции за 2025-11-01

```sql
ALTER TABLE default.uk_price_paid_daily
DETACH PARTITION 20251101;
```
Проверка detached-партов:
```sql
SELECT * 
FROM system.detached_parts 
WHERE table = 'uk_price_paid_daily';
```
Скриншот результата запроса
![hw16_add_step5](https://github.com/realexpert1C/clickhouse-course/blob/adfe2e4b3be1a4a49e54924190081db1c79dd9ce/images/hw16_add_step5.png)

---

#### ⚖️ 7.6: Сравнение DETACHED и вставленных данных

Создаю таблицу для сравнения:

```sql
CREATE TABLE default.uk_price_paid_daily_compare
AS default.uk_price_paid_daily
ENGINE = MergeTree
PARTITION BY toYYYYMMDD(date)
ORDER BY (postcode1, postcode2, addr1, addr2);
```
	1.	ATTACH PART из detached:
```sql
ALTER TABLE default.uk_price_paid_daily_compare
ATTACH PARTITION 20251101
FROM uk_price_paid_daily;
```
	2.	Повторно вставляю свежие данные:

```sql
INSERT INTO default.uk_price_paid_daily_compare
SELECT *
FROM default.uk_price_paid
WHERE date = '2025-11-01';
```
Сравнение:
```sql
SELECT date, count(*) 
FROM default.uk_price_paid_daily_compare
GROUP BY date;
```

Скриншот результата запроса
![hw16_add_step6](https://github.com/realexpert1C/clickhouse-course/blob/adfe2e4b3be1a4a49e54924190081db1c79dd9ce/images/hw16_add_step6.png)


---

#### 🧬 7.7: Реплицируемая таблица + FETCH

Возвращаю партицию из detached обратно в `uk_price_paid_daily`

```sql
ALTER TABLE default.uk_price_paid_daily
ATTACH PARTITION 20251101;
```

Создаю реплицируемую таблицу

```sql
CREATE TABLE default.uk_price_paid_daily_repl
AS default.uk_price_paid_daily
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/default/uk_price_paid_daily_repl', '{replica}')
PARTITION BY toYYYYMMDD(date)
ORDER BY (postcode1, postcode2, addr1, addr2);
```

Скачиваю партицию:

```sql
ALTER TABLE default.uk_price_paid_daily_repl
ATTACH PARTITION 20251101
FROM default.uk_price_paid_daily;
```

Проверяю наличие нужной партиции

```sql
SELECT partition, count(), min_date, max_date
FROM system.parts
WHERE table = 'uk_price_paid_daily_repl'
  AND partition = '20251101'
  AND active
GROUP BY partition, min_date, max_date;
```

Скриншот результат запроса
![hw16_add_step7](https://github.com/realexpert1C/clickhouse-course/blob/adfe2e4b3be1a4a49e54924190081db1c79dd9ce/images/hw16_add_step7.png)

---

#### ⏳ 7.8: TTL на удаление старых партиций

Создаю таблицу с PARTITION BY toYYYYMM(date) и TTL:
```sql
CREATE TABLE default.uk_price_paid_monthly_ttl
AS default.uk_price_paid
ENGINE = MergeTree
PARTITION BY toYYYYMM(date)
ORDER BY (postcode1, postcode2, addr1, addr2)
TTL date + INTERVAL 1 MONTH;
```

Вставляю устаревшие записи:

```sql
INSERT INTO default.uk_price_paid_monthly_ttl
SELECT *
FROM default.uk_price_paid
WHERE date BETWEEN '2015-01-01' AND '2015-12-31';
```

Проверка созданных партиций:
```sql
SELECT
    partition,
    min_date,
    max_date,
    count() AS rows
FROM system.parts
WHERE table = 'uk_price_paid_monthly_ttl'
  AND active
GROUP BY partition, min_date, max_date
ORDER BY partition;
```


После срабатывания TTL (или вручную через OPTIMIZE TABLE) данные удаляются.
Ручное удаление `OPTIMIZE TABLE default.uk_price_paid_monthly_ttl FINAL;`
Проверка:

```sql
SELECT DISTINCT partition
FROM system.parts
WHERE table = 'uk_price_paid_monthly_ttl';
```

Скриншот результата запроса
![hw16_add_step8](https://github.com/realexpert1C/clickhouse-course/blob/adfe2e4b3be1a4a49e54924190081db1c79dd9ce/images/hw16_add_step8.png)

---

### Выводы

* Создана и заполнена таблица `user_activity` с партиционированием по дате.
* Выполнена мутация с обновлением данных и проверена через system.mutations.
* Успешно удалена партиция по дате и подтверждено её отсутствие.
* Реализован полный цикл работы с партициями: INSERT, DROP, DETACH, ATTACH, REPL.
* Настроена TTL-очистка устаревших данных на примере месячного партиционирования.
* Отработаны ключевые приёмы безопасного восстановления и сравнения партиций.