Отлично. Поздравляю с высокой оценкой по 25-й 👏
Теперь делаем 26-ю строго, технически корректно и без «воды».

⸻

Домашняя работа №26

Интеграция ClickHouse с PostgreSQL

Мы сделаем:
	1.	PostgreSQL в Docker (в твоей infra)
	2.	Тестовый датасет
	3.	Запрос через функцию postgres()
	4.	Таблицу на движке PostgreSQL
	5.	Базу данных на движке PostgreSQL

⸻

Шаг 1. PostgreSQL (Docker)

Создаём каталог:

mkdir ~/infra/postgres-ch
cd ~/infra/postgres-ch

docker-compose.yaml

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

Запуск:

docker compose up -d


⸻

Шаг 2. Тестовый датасет в PostgreSQL

Подключаемся:

docker exec -it postgres_ch psql -U pguser -d demo_pg

Создаём таблицу:

CREATE TABLE trades_pg (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(20),
    price NUMERIC(18,8),
    quantity NUMERIC(18,8),
    trade_time TIMESTAMP
);

Заполняем:

INSERT INTO trades_pg (symbol, price, quantity, trade_time) VALUES
('BTCUSDT', 30000.12, 0.5, '2024-01-01 10:00:00'),
('BTCUSDT', 30010.45, 0.3, '2024-01-01 10:01:00'),
('ETHUSDT', 2000.10, 1.2, '2024-01-01 10:02:00');

Проверка:

SELECT * FROM trades_pg;


⸻

Шаг 3. Запрос из ClickHouse через функцию postgres()

Теперь из ClickHouse:

SELECT *
FROM postgres(
    'postgres_ch:5432',
    'demo_pg',
    'trades_pg',
    'pguser',
    'pgpassword'
);

Ожидаемый результат — 3 строки.

⸻

Шаг 4. Таблица в ClickHouse через ENGINE = PostgreSQL

Создаём таблицу:

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

Проверка:

SELECT * FROM demo.trades_pg_engine;


⸻

Шаг 5. База данных на движке PostgreSQL

Создаём БД:

CREATE DATABASE demo_pg_db
ENGINE = PostgreSQL(
    'postgres_ch:5432',
    'demo_pg',
    'pguser',
    'pgpassword'
);

Проверка:

SHOW TABLES FROM demo_pg_db;

Должна появиться таблица:

trades_pg

Проверка запроса:

SELECT * FROM demo_pg_db.trades_pg;


⸻

Что отправлять на проверку

В отчёт включить:
	1.	CREATE TABLE в PostgreSQL
	2.	INSERT
	3.	SELECT через postgres()
	4.	CREATE TABLE ENGINE = PostgreSQL
	5.	CREATE DATABASE ENGINE = PostgreSQL
	6.	Результаты SELECT из ClickHouse

⸻

Архитектурно что ты показал
	•	ClickHouse → PostgreSQL через table function
	•	ClickHouse → PostgreSQL через table engine
	•	ClickHouse → PostgreSQL через database engine

Это три уровня интеграции.

⸻

Если хочешь, дальше можем:
	•	сравнить производительность postgres() vs ENGINE
	•	сделать материализацию данных из Postgres в MergeTree
	•	разобрать pushdown фильтров
	•	или сделать реальный кейс для финального проекта

Готов начинать настройку?