
# Проектная работа по курсу обучения OTUS "Clickhouse для инженеров и архитекторов БД"

# Тема. Аналитика в реальном времени на базе ClickHouse. Сравнительный анализ batch и streaming загрузки данных при неравномерной нагрузке
---

## 1. Обзор проекта

### 🎯 1.1. Цель проекта

Разработать и продемонстрировать MVP аналитической платформы на базе ClickHouse для сравнения двух подходов к загрузке данных:
- пакетная загрузка (batch ingestion)
- потоковая загрузка (streaming ingestion)

при условиях неравномерной нагрузки, приближенной к реальной производственной эксплуатации.

Проект моделирует ситуацию, когда параллельные вставки данных в ClickHouse могут приводить к:
* накоплению очереди вставок (insert queue)
* отставанию слияний частей (merge backlog)
* задержкам выполнения DAG в Airflow
* росту времени выполнения запросов

---

### 1.2. Моделируемая производственная проблема

В реальных системах (например, в высоконагруженных приложениях):
- ~1 000 000 записей в сутки
- выраженная неравномерность нагрузки (пик днём, минимум ночью)
- параллельная загрузка:
- ETL-процессы для BI
- поток сырого лога напрямую в ClickHouse

Это может приводить к:
- конкуренции за ресурсы
- накоплению мелких частей (small parts)
- задержкам выполнения зависимых задач
- деградации производительности

Данный проект воспроизводит подобный сценарий в контролируемых условиях.

Ввиду действующего между мной и закзачиком соглашения о конфиденциальности, нет возможности представить модель на реальных данных, поэтому выбран близкий по параметрам сценарий. В проекте будут использованы открытые данные о проведенных сделках на бирже Binance по самым распространенным торговым парам, а также данные о погоде и потреблении электричества в США. Период для всех датасетов - июль 2023 года - месяц, когда в США была зафиксирована аномальная жара с самой высокой температурой за всю историю наблюдений. Нет цели доказать корреляцию этих данных, использую только для имитации реальной работы.

---

### 1.3. Архитектурная концепция проекта

Проект представляет собой лабораторную платформу для исследования поведения ClickHouse при:
- неравномерной нагрузке (bursty traffic),
- параллельной загрузке данных,
- сравнении batch ingestion и streaming ingestion,
- управляемом увеличении интенсивности потока.


### 1.4. Источник истины (Source of Truth)

__Исторические датасеты__

Перед запуском основной системы выполняется однократная выгрузка всех сделок Binance (aggTrades) за июль 2023 года, а также однократная выгрузка данных о погоде и потреблении электроэнергии в США за тот же период

Данные о сделках:
- сохраняются локально,
- не изменяются,
- используются как единственный источник событий.

Формат хранения: Parquet

Файл размещается в:

`infra/replay/data/july_2023.parquet`

Это обеспечивает:
- воспроизводимость экспериментов,
- отсутствие зависимости от Binance API,
- контроль над нагрузкой,
- возможность циклического воспроизведения.

Данные о погоде и потреблении электроэнергии выгружаются напрямую в Clickhouse и используются в качестве справочников

---

### 1.5. Replay Service — центральный управляющий компонент

Replay Service — отдельный контейнер в Docker Compose, который:
- читает исторический файл,
- воспроизводит события с контролем времени,
- генерирует дополнительные строки,
- формирует два параллельных потока:
  * streaming,
  * batch.

---

#### Общая схема потоков
```txt
        Исторический файл (июль 2023)
                    ↓
              Replay Service
                    ↓
    ┌───────────────┬────────────────┐
    ↓                                ↓
Kafka Topic                     Batch Buffer
    ↓                                ↓
ClickHouse (stream)          ClickHouse (batch)
```
Replay Service является интеллектуальным слоем системы. Kafka используется только как транспорт сообщений.

---

#### Имитация реального времени (Time Replay Engine)

Replay Service:
1.	Сортирует события по event_time.
2.	Вычисляет разницу между соседними событиями.
3.	Воспроизводит поток с масштабированием времени.

Параметр:

`TIME_SCALE`

Примеры:
- 1.0 → реальная скорость
- 10.0 → в 10 раз быстрее
- 0.5 → в 2 раза медленнее

Таким образом сохраняется естественная неравномерность (bursty pattern).

---

#### Контроль интенсивности нагрузки

Параметр:

`INTENSITY_K`

Позволяет:
* генерировать дополнительные строки на основе оригинального события,
* масштабировать нагрузку синхронно для batch и streaming,
* исследовать пределы деградации системы.

Если `INTENSITY_K` = 5, то на каждое исходное событие создаётся 4 дополнительных.

Дополнительные строки:
- сохраняют структуру,
- могут иметь микро-смещение timestamp,
- увеличивают частоту вставок.

---

#### Streaming Pipeline

Streaming реализуется через:
- Kafka Topic,
- ClickHouse Kafka Engine,
- Materialized View для записи в raw-таблицу.

Поток:

Replay Service → Kafka → ClickHouse Kafka Engine → Raw Layer

Streaming-поток:
- не агрегируется до вставки,
- обрабатывается в near real-time,
- чувствителен к burst-нагрузке.

---

#### Batch Pipeline

Batch-поток формируется Replay Service параллельно streaming.

Логика формирования батча:
- вставка при достижении MIN_BATCH_SIZE,
- либо по таймеру MAX_BATCH_INTERVAL,
- но не менее 4 вставок в час.

Таким образом моделируется классический ETL-процесс.

---

### 1.6. Многоуровневая архитектура хранения

#### Raw Layer (HOT)

Назначение:
- хранение детализированных сделок,
- прием batch и streaming,
- временное хранение.

Реализация:
- ReplicatedMergeTree,
- партиционирование по дате,
- сортировка по (symbol, event_time, trade_id).

Очистка данных:
- TRUNCATE / DROP PARTITION по завершении цикла месяца.

---

#### Aggregation Layer (WARM)

Реализация:
- Materialized Views,
- агрегации по:
	* 1 минуте,
	* 15 минутам,
	* 1 часу,
	* 1 дню.

---

#### Data Mart Layer (Analytics / BI)

Формируются витрины, которые:
* используют агрегированные сделки,
* выполняют JOIN с:
	- таблицей погоды,
	- таблицей потребления электроэнергии.

JOIN выполняется по часовым интервалам.

Именно этот слой используется для BI.

---

### 1.7. Визуализация и сравнение

Инструмент:

Yandex DataLens

Создаются два полностью идентичных дашборда:
1. Dashboard A — данные из batch ingestion
2. Dashboard B — данные из streaming ingestion

Селекторы синхронизированы.

Дополнительно выводятся контрольные метрики:
- общее количество строк,
- суммы по колонкам,
- агрегированные показатели.

Это позволяет:
- сравнивать идентичность данных,
- анализировать влияние способа ingestion на нагрузку.

---

### 1.8. Мониторинг

Используются:
* Prometheus
* Grafana

Мониторятся:
- insert rate
- количество частей (parts count)
- backlog merge
- CPU
- память
- Kafka lag
- время выполнения запросов

---

🧠 Что именно нужно мониторить в Replay Service

🔹 1️⃣ Скорость отправки событий

Метрики:
	•	replay_events_total
	•	replay_events_per_second
	•	batch_flush_total
	•	batch_size_current
	•	kafka_messages_sent_total

Это позволит понять:
	•	реально ли идут данные
	•	совпадает ли интенсивность с ожидаемой
	•	нет ли просадки

⸻

🔹 2️⃣ Ошибки вставки
	•	kafka_send_errors_total
	•	clickhouse_insert_errors_total
	•	retry_count_total

⚠ Это критично для алертов.

⸻

🔹 3️⃣ Задержки
	•	batch_flush_duration_seconds
	•	kafka_send_latency_seconds
	•	replay_processing_lag_seconds

Это позволит увидеть:
	•	не начал ли batch “тормозить”
	•	не зависает ли Kafka producer

⸻

📈 Что мониторит Prometheus в целом

ClickHouse
	•	InsertQuery
	•	PartsNumber
	•	BackgroundMergesAndMutationsPoolTask
	•	ReplicasMaxQueueSize
	•	QueryDuration
	•	DelayedInserts

Kafka
	•	consumer lag
	•	messages in per second
	•	messages out per second

Система
	•	CPU
	•	RAM
	•	IO wait
	•	Disk write

⸻

🚨 Какие алерты нужны

По Replay Service
	•	нет новых событий > 60 сек
	•	batch не flush > 20 мин
	•	error rate > 1%
	•	INTENSITY_K применён, а скорость не выросла

⸻

По ClickHouse
	•	parts > 10 000
	•	merges backlog > threshold
	•	insert queue delay > X
	•	replication queue > X

⸻

По Kafka
	•	consumer lag растёт > 5 минут
	•	producer error

⸻

📜 Логирование Replay Service

Должно быть:
	•	JSON-логи
	•	structured logging
	•	уровень INFO / WARNING / ERROR
	•	вывод в stdout (Docker logs)

Дальше:
	•	Promtail → Loki (опционально)
или
	•	просто хранение docker logs

⸻

🎯 Важное понимание

Replay Service — это не просто генератор.
Это участник системы, и его нужно мониторить так же, как ClickHouse.

⸻

🧱 Архитектурно правильно

Replay Service
    ↓ metrics
Prometheus
    ↓
Grafana Dashboard:
    - Replay rate
    - Batch rate
    - Kafka rate
    - Insert rate
    - CH parts
    - CPU



Это позволяет наблюдать реакцию системы при изменении нагрузки.

---

### 1.9. Циклический режим работы

После завершения воспроизведения июля 2023:
1.	выполняется очистка raw-таблиц,
2.	счётчики сбрасываются,
3.	воспроизведение начинается заново.

Таким образом создаётся замкнутый лабораторный цикл.

---

### 1.10. Кластер ClickHouse

Конфигурация:
- 1 shard,
- 2 replicas,
- 3 ClickHouse Keeper.

Обеспечивается:
- репликация,
- отказоустойчивость,
- корректная работа Distributed-таблиц.
---
### 1.11. Инфраструктура и аппаратная конфигурация (Deployment Environment)

Сервер (Execution Environment)
- CPU: 12 vCPU
- RAM: 40 GB
- Storage: NVMe SSD 200GB
- OS: Ubuntu 22.04 LTS
- Docker: 29.2.1
--- 

### 1.12. Архитектурная ценность проекта

Архитектура демонстрирует:
* различия batch и streaming ingestion,
* влияние неравномерной нагрузки,
* влияние масштабирования интенсивности,
* поведение ClickHouse при burst-нагрузке,
* управление жизненным циклом данных,
* построение production-подобной архитектуры аналитической платформы.

PHASE 0 — Базовая инфраструктура

Цель:

Подготовить сервер, сеть и ClickHouse кластер.

Что входит:
	•	Ubuntu Server
	•	Docker + Docker Compose
	•	WireGuard (VPS ↔ Ubuntu)
	•	Проброс портов через VPS
	•	ClickHouse cluster (1 shard, 2 replicas)
	•	ClickHouse Keeper (3 nodes)
	•	Portainer

⸻

📍 Текущий статус (МЫ ЗДЕСЬ)

✔ Ubuntu Server развернут
✔ Docker работает
✔ WireGuard настроен
✔ VPS пробрасывает порт
✔ Развернут ClickHouse cluster
✔ Keeper cluster работает
✔ Portainer работает

➡ Базовый слой готов.

⸻

🔜 PHASE 1 — Инфраструктура данных (Bootstrap Layer)

Цель:

Подготовить систему к работе с данными.

1.1 Установка и настройка Kafka
	•	контейнер Kafka
	•	контейнер Zookeeper (если не используем KRaft)
	•	внутренние сети Docker
	•	topic для сделок

1.2 Установка Prometheus
	•	сбор метрик:
	•	ClickHouse
	•	Kafka
	•	Node Exporter
	•	Docker

1.3 Установка Grafana
	•	базовые dashboards:
	•	CPU
	•	RAM
	•	Disk
	•	ClickHouse
	•	Kafka

1.4 Развертывание MinIO (cold storage)
	•	S3-compatible storage
	•	бакет для архивов

⸻

🔜 PHASE 2 — Bootstrap загрузка данных через Airflow

Цель:

Создать source-of-truth датасет (июль 2023).

2.1 Развернуть Airflow
	•	docker-compose
	•	Postgres backend
	•	volume для dags
	•	публичный доступ через Nginx

2.2 DAG: Загрузка Binance July 2023
	•	загрузка через API
	•	запись в staging table
	•	экспорт в Parquet
	•	сохранение в:
	•	локальный volume
	•	MinIO (опционально)

2.3 DAG: Загрузка погоды
	•	загрузка hourly
	•	создание reference table

2.4 DAG: Загрузка электроэнергии
	•	загрузка hourly
	•	создание reference table

⸻

🔜 PHASE 3 — Проектирование схемы ClickHouse

Цель:

Создать правильную архитектуру таблиц.

3.1 Raw Layer
	•	replicated raw table
	•	TTL или manual cleanup

3.2 Kafka Engine Table
	•	таблица для streaming ingestion

3.3 Materialized Views
	•	из Kafka → Raw
	•	агрегации (1m, 15m, 1h, 1d)

3.4 Data Mart
	•	таблицы с JOIN:
	•	trades
	•	weather
	•	electricity

⸻

🔜 PHASE 4 — Replay Service (ключевой этап)

Цель:

Создать управляемый генератор нагрузки.

4.1 Контейнер Replay Service
	•	чтение Parquet
	•	TIME_SCALE
	•	INTENSITY_K
	•	batch + streaming

4.2 Kafka Producer
	•	acks=all
	•	idempotence=true

4.3 Batch Engine
	•	MIN_BATCH_SIZE
	•	MAX_BATCH_INTERVAL

4.4 State Control
	•	start / stop / pause
	•	loop July

⸻

🔜 PHASE 5 — Мониторинг лаборатории

Цель:

Сделать систему наблюдаемой.

5.1 ClickHouse metrics
	•	insert rate
	•	parts
	•	merges backlog

5.2 Kafka metrics
	•	lag
	•	throughput

5.3 Replay metrics
	•	events/sec
	•	batch size
	•	error rate

5.4 Grafana dashboards
	•	unified ingestion experiment dashboard

⸻

🔜 PHASE 6 — Визуализация

Цель:

Сравнение batch vs streaming.

6.1 Развернуть DataLens

(или подключить к существующему)

6.2 Два идентичных dashboard
	•	batch source
	•	streaming source

6.3 Синхронизированные фильтры

⸻

🔜 PHASE 7 — Управление жизненным циклом

7.1 TTL или TRUNCATE после цикла

7.2 DAG для очистки

7.3 DAG для запуска нового эксперимента

⸻

🔜 PHASE 8 — Backup & Recovery

8.1 clickhouse-backup

8.2 backup в MinIO

8.3 восстановление кластера

⸻

🔜 PHASE 9 — Документация и репозиторий

9.1 README.md
	•	архитектура
	•	схемы
	•	инструкции запуска

9.2 Скриншоты
	•	Portainer
	•	Grafana
	•	DataLens
	•	нагрузка

9.3 Описание эксперимента
	•	при INTENSITY_K = 1
	•	при INTENSITY_K = 5
	•	при INTENSITY_K = 10

⸻

🔜 PHASE 10 — Презентация (10–15 минут)
	1.	Проблема
	2.	Архитектура
	3.	Инженерное решение
	4.	Демонстрация
	5.	Результаты
	6.	Ограничения
	7.	Что можно улучшить


## 2. Реализация проекта












⸻

1. Проектирование хранилища данных (Data Warehouse)

4.1 Создание базы данных

CREATE DATABASE IF NOT EXISTS demo;


⸻

4.2 Таблица сделок Binance

CREATE TABLE IF NOT EXISTS demo.binance_aggtrades_raw
(
    symbol LowCardinality(String),

    event_time DateTime64(3, 'UTC'),

    agg_trade_id UInt64,

    price Float64,
    quantity Float64,

    first_trade_id UInt64,
    last_trade_id UInt64,

    is_buyer_maker UInt8
)
ENGINE = MergeTree
PARTITION BY toDate(event_time)
ORDER BY (symbol, event_time);

Пояснение структуры:
	•	symbol — торговая пара
	•	event_time — время сделки
	•	agg_trade_id — агрегированный идентификатор сделки
	•	price — цена
	•	quantity — объём
	•	first_trade_id, last_trade_id — диапазон идентификаторов
	•	is_buyer_maker — флаг стороны инициатора

📸 [МЕСТО ДЛЯ СКРИНШОТА: clickhouse_table_structure.png]

⸻

4.3 Таблица погодных данных

CREATE TABLE IF NOT EXISTS demo.weather_hourly
(
    timestamp DateTime('UTC'),
    temperature Float64,
    humidity Float64,
    precipitation Float64
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)
ORDER BY timestamp;


⸻

4.4 Таблица потребления электроэнергии

CREATE TABLE IF NOT EXISTS demo.electricity_hourly
(
    timestamp DateTime('UTC'),
    region String,
    demand_mw Float64
)
ENGINE = MergeTree
PARTITION BY toDate(timestamp)
ORDER BY (region, timestamp);


⸻

5. Пайплайны обработки данных

⸻

5.1 Пакетная загрузка (Batch ingestion) через Airflow

DAG: load_demo_day.py

Функциональность:
	•	загрузка данных Binance за 2023-07-01
	•	загрузка погодных данных
	•	загрузка данных по потреблению электроэнергии
	•	вставка данных в ClickHouse

📸 [МЕСТО ДЛЯ СКРИНШОТА: airflow_dag_graph.png]

⸻

Переменные окружения Airflow

Добавляются в docker-compose:

environment:
  - CLICKHOUSE_HOST=ch1
  - CLICKHOUSE_PORT=8123
  - CLICKHOUSE_USER=default
  - CLICKHOUSE_PASSWORD=default123
  - EIA_API_KEY=YOUR_KEY


⸻

Основные параметры Airflow

AIRFLOW__CORE__EXECUTOR: LocalExecutor
AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: "true"

📸 [МЕСТО ДЛЯ СКРИНШОТА: airflow_success_run.png]

⸻

5.2 Потоковая загрузка (Streaming ingestion) через Kafka

⚠ В разработке

Планируется:
	•	Kafka topic: binance_trades
	•	продюсер (producer), имитирующий реальное поступление данных
	•	таблица Kafka Engine в ClickHouse
	•	Materialized View (материализованное представление) для загрузки

CREATE TABLE demo.binance_kafka
(
    symbol String,
    event_time DateTime64(3, 'UTC'),
    price Float64,
    quantity Float64
)
ENGINE = Kafka
SETTINGS kafka_broker_list = 'kafka:9092',
         kafka_topic_list = 'binance_trades',
         kafka_group_name = 'clickhouse_group',
         kafka_format = 'JSONEachRow';

CREATE MATERIALIZED VIEW demo.binance_mv
TO demo.binance_aggtrades_raw
AS SELECT * FROM demo.binance_kafka;

📸 [МЕСТО ДЛЯ СКРИНШОТА: kafka_streaming.png]

⸻

6. Мониторинг

6.1 Prometheus

Собираемые метрики:
	•	загрузка CPU
	•	использование памяти
	•	дисковые операции
	•	метрики ClickHouse
	•	задержка потребителя Kafka (consumer lag)

📸 [МЕСТО ДЛЯ СКРИНШОТА: prometheus_targets.png]

⸻

6.2 Grafana

Дашборды включают:
	•	скорость вставок (insert rate)
	•	количество частей (parts count)
	•	отставание merge
	•	нагрузка CPU
	•	задержки выполнения запросов

📸 [МЕСТО ДЛЯ СКРИНШОТА: grafana_dashboard.png]

⸻

6.3 Оповещения (Alerts)

⚠ Планируется реализовать:
	•	превышение размера очереди вставок
	•	высокая загрузка CPU
	•	рост задержки Kafka

⸻

7. Моделирование нагрузки

Планируется реализация регулируемых параметров:

Параметр	Назначение
TIME_SCALE	ускорение воспроизведения исторических данных
INTENSITY_K	коэффициент увеличения плотности событий

Это позволит определить порог деградации инфраструктуры.

⸻

8. Структура репозитория

project-root/
│
├── infra/
│   ├── clickhouse/
│   ├── airflow/
│   ├── kafka/
│   └── monitoring/
│
├── dags/
│   └── load_demo_day.py
│
├── sql/
│   ├── create_tables.sql
│
├── screenshots/
│
└── README.md


⸻

9. Документация и воспроизводимость

⚠ Раздел будет дополнен:
	1.	Клонирование репозитория
	2.	Настройка .env
	3.	Запуск docker compose
	4.	Доступ к Airflow
	5.	Запуск DAG
	6.	Просмотр Grafana

⸻

10. Текущий статус реализации

Реализовано:
	•	инфраструктура
	•	пакетная загрузка
	•	базовая схема хранилища
	•	мониторинг

В разработке:
	•	потоковая загрузка
	•	стресс-тестирование
	•	система оповещений
	•	централизованное логирование

⸻

11. План сравнения Batch и Streaming

Метрики для сравнения:
	•	задержка вставки (insert latency)
	•	количество частей (parts)
	•	задержка merge
	•	загрузка CPU
	•	задержка выполнения DAG

⸻

12. Соответствие требованиям курса

✔ Развернутая инфраструктура
✔ Реальные датасеты
✔ Построенная архитектура хранилища
✔ Пакетный пайплайн
⚠ Потоковый пайплайн — в разработке
⚠ Alerts — в разработке
⚠ Централизованное логирование — планируется

⸻

13. MVP проекта

MVP включает:
	•	работающий кластер ClickHouse
	•	пакетную загрузку через Airflow
	•	мониторинг через Prometheus + Grafana
	•	демонстрационные данные
	•	документированную архитектуру

⸻

14. Перспективы развития
	•	реализация streaming ingestion
	•	нагрузочное тестирование
	•	автоматизация деплоя
	•	логирование (Loki / ELK)
	•	CI/CD

⸻

15. Презентация

📎 [Ссылка на презентацию будет добавлена после подготовки]

⸻

16. Чек-лист скриншотов
	•	Архитектура
	•	Граф DAG
	•	Успешный запуск DAG
	•	Структура таблиц
	•	Дашборд Grafana
	•	Prometheus targets
	•	Kafka ingestion
	•	Alerts

⸻

Если хочешь, дальше можем:
	1.	Сделать версию README уровня “идеально на защиту”
	2.	Добавить раздел “Архитектурные решения и обоснование выбора”
	3.	Начать реализовывать Kafka слой полностью
	4.	Сформировать структуру презентации на 12 слайдов

Выбирай направление.