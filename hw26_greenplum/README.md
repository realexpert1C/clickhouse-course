# Домашнее задание 26 - Интеграция ClickHouse с PostgreSQL

Цели задания:
1.	Установка PostgreSQL в Docker (в уже существующей сети infra-net)
2.	Создание тестовой таблицы
3.	Запрос данных из PostgresQL c помощью функции postgresql()
4. На стороне ClickHouse создание таблицы для интеграции с PostgreSQL через движок Postgres.
5. На стороне ClickHouse создание базы данных для интеграции с PostgreSQL.

---

## Шаг 1. Установка PostgreSQL (Docker)

Создаю каталог:
```bash
mkdir ~/infra/postgres-ch
mkdir ~/infra/postgres-ch/pg_data
cd ~/infra/postgres-ch
```
Файл `docker-compose.yaml`:
```yml
version: "3.8"

services:
  postgres_ch:
    image: postgres:15
    container_name: postgres_ch
    environment:
      POSTGRES_DB: demo_pg
      POSTGRES_USER: pguser
      POSTGRES_PASSWORD: pgpassword
    ports:
      - "5433:5432"
    volumes:
      - ./pg_data:/var/lib/postgresql/data
    networks:
      - infra-net

networks:
  infra-net:
    external: true
```
Запуск:
```bash
docker compose up -d
```
---

## Шаг 2. Тестовый датасет в PostgreSQL

Подключаюсь:
```bash
docker exec -it postgres_ch psql -U pguser -d demo_pg
```
Создаю таблицу:
```sql
CREATE TABLE trades_pg (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(20),
    price NUMERIC(18,8),
    quantity NUMERIC(18,8),
    trade_time TIMESTAMP
);
```

Заполняю:
```sql
INSERT INTO trades_pg (symbol, price, quantity, trade_time) VALUES
('BTCUSDT', 30000.12, 0.5, '2024-01-01 10:00:00'),
('BTCUSDT', 30010.45, 0.3, '2024-01-01 10:01:00'),
('ETHUSDT', 2000.10, 1.2, '2024-01-01 10:02:00');
```
Проверка:
```sql
SELECT * FROM trades_pg;
```
![Скриншот1 SELECT после INSERT]()

---

## Шаг 3. Запрос из ClickHouse через функцию postgresql()

Теперь из ClickHouse:
```sql
SELECT *
FROM postgresql(
    'postgres_ch:5432',
    'demo_pg',
    'trades_pg',
    'pguser',
    'pgpassword'
);
```
Ожидаемый результат — 3 строки.

![Скриншот2 SELECT через postgres]()

Данные из Postgres получены

---

## Шаг 4. Таблица в ClickHouse через ENGINE = PostgreSQL

Создаю таблицу в Clickhouse:
```sql
CREATE TABLE demo.trades_pg_engine
(
    id UInt32,
    symbol String,
    price Float64,
    quantity Float64,
    trade_time DateTime
)
ENGINE = PostgreSQL(
    'postgres_ch:5432',
    'demo_pg',
    'trades_pg',
    'pguser',
    'pgpassword'
);
```
Проверка:
```sql
SELECT * FROM demo.trades_pg_engine;
```
![Скриншот3 SELECT * FROM demo.trades_pg_engine]()

---

## Шаг 5. База данных на движке PostgreSQL

Создаю БД:
```sql
CREATE DATABASE demo_pg_db
ENGINE = PostgreSQL(
    'postgres_ch:5432',
    'demo_pg',
    'pguser',
    'pgpassword'
);
```
Проверка:

```sql
SHOW TABLES FROM demo_pg_db;
```
Должна появиться таблица:

`trades_pg`

![Скриншот4 SHOW TABLES]()

Таблица видна.

Проверка содержимого таблицы:
```sql
SELECT * FROM demo_pg_db.trades_pg;
```
![Скриншот5 Результаты SELECT из ClickHouse]()
База данных создана, данные читаются

---
### Итог
Архитектурно видим три уровня интеграции:
* ClickHouse → PostgreSQL через table function postgresql()
* ClickHouse → PostgreSQL через table engine
* ClickHouse → PostgreSQL через database engine