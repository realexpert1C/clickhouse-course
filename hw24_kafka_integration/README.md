# Домашнее задание 24 - Интеграция ClickHouse с Apache Kafka

#### Цель работы

Изучить интеграцию ClickHouse с Apache Kafka и реализовать пайплайн записи и чтения данных:
* запись данных в Kafka
* чтение данных через Kafka Engine
* загрузка в MergeTree через Materialized View
* проверка корректности загрузки

---
#### Архитектура стенда

Инфраструктура
- ClickHouse Cluster
  * 1 shard
  * 4 replicas: ch1, ch2, ch3, ch4
  * Репликация через ClickHouse Keeper (развернут на ch1)
- Airflow 3.1.7
- Python 3.10
- Executor: LocalExecutor
- Docker Compose
- Общая сеть: infra-net
- Apache Kafka (развернут в Docker)

---

#### Разворачивание Apache Kafka теория

Что такое Kafka в контексте задачи

__Apache Kafka — это распределённый commit log__

В нашем пайплайне:

Producer → Kafka Topic → ClickHouse Kafka Engine → Materialized View → MergeTree

Kafka здесь выступает:
- буфером
- брокером сообщений
- механизмом доставки
- хранилищем offset


__Компоненты Kafka__

Минимальная архитектура для ДЗ:
- Broker — хранит данные
- Topic — лог сообщений
- Partition — параллелизм
- Consumer Group — группа потребителей (ClickHouse будет consumer)

## Шаг 1. Разворачивание Apache Kafka

Создаю рабочую директорию infra/kafka:

```bash
mkdir ~/infra/kafka
mkdir ~/infra/kafka/kafka-data
cd ~/infra/kafka
```
В папке kafka создаю docker-compose.yaml:

```yml
version: "3.8"

services:

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: kafka-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - infra-net

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: kafka-zookeeper:2181

      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT

      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

      KAFKA_LOG_RETENTION_HOURS: 72
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1

    volumes:
      - ./kafka-data:/var/lib/kafka/data

    networks:
      - infra-net

networks:
  infra-net:
    external: true
```

Создание топика:

```bash
docker exec -it kafka kafka-topics \
  --create \
  --topic events_topic \
  --bootstrap-server kafka:9092 \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=259200000 \
  --config segment.ms=3600000
```
#### ! Важно
```bash
--partitions 6
```
Параллелизм под:
* 4 consumer в ClickHouse
* будущий рост нагрузки
```bash
--replication-factor 1
```
У меня 1 broker
```bash
retention.ms=259200000
```
72 часа хранения (3 дня).
```bash
segment.ms=3600000
```
Ротация сегментов раз в 1 час
(удобно для лабораторных экспериментов).

![📸 Скриншот 1: список топиков (kafka-topics --list)](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_kafka_topic_list.png)

---

## Шаг 2. Настройка ClickHouse для работы с Kafka

Kafka Engine используется на уровне таблицы.

Проверка доступности Kafka из контейнера ClickHouse:

```bash
docker exec -it ch1 bash -c "</dev/tcp/kafka/9092" && echo OK
```
![📸 Скриншот 2: OK kafka из ch1](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_conn_check.png)

__Как ClickHouse читает Kafka__

Kafka Engine в ClickHouse:
- запускает consumer
- подписывается на topic
- читает данные пачками
- передаёт строки в Materialized View
- коммитит offset только после успешной обработки

Важно:

Kafka Engine не хранит данные — он потоковый источник.

---

## Шаг 3. Создание таблиц в ClickHouse

Работа выполняется на кластере через ON CLUSTER.

### 3.1 Таблица назначения (MergeTree)

```sql
CREATE TABLE default.events_local ON CLUSTER replicated_cluster
(
    event_id UInt64,
    user_id UInt64,
    event_type String,
    event_time DateTime
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events_local',
    '{replica}'
)
PARTITION BY toYYYYMM(event_time)
ORDER BY (event_time, user_id);
```

### 3.2 Distributed таблица

```sql
CREATE TABLE default.events_dist ON CLUSTER replicated_cluster
AS default.events_local
ENGINE = Distributed(replicated_cluster, default, events_local, rand());
```
Ниже результат проверочного запроса:

```sql
SELECT
    hostName() AS host,
    name,
    engine
FROM clusterAllReplicas('replicated_cluster', system.tables)
WHERE database = 'default' AND name LIKE 'events%'
ORDER BY host, name;
```

![📸 Скриншот 3: SHOW TABLES на кластере](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_check_tbls.png)

---

## Шаг 4. Создание Kafka Engine таблицы

```sql
CREATE TABLE default.events_kafka ON CLUSTER replicated_cluster
(
    event_id UInt64,
    user_id UInt64,
    event_type String,
    event_time DateTime
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'events_topic',
    kafka_group_name = 'clickhouse_consumer_group',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 4,
    kafka_skip_broken_messages = 1;
```
![📸 Скриншот 4: DESCRIBE TABLE events_kafka](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_desc_events_kafka.png)

Разбор параметров

|Параметр|Значение|
|--------|--------|
|kafka_broker_list|Адрес Kafka|
|kafka_topic_list|Топик|
|kafka_group_name|Consumer Group|
|kafka_format|Формат сообщений|
|kafka_num_consumers|Параллельные consumers|
|kafka_skip_broken_messages|Игнор битых JSON|


Как работает consumer group
* Все реплики используют одну группу
* Kafka распределяет partitions
* Offset хранится в Kafka

Если Materialized View падает — offset не коммитится.

Это сделано для обепечения at-least-once доставки.

---

## Шаг 5. Materialized View для загрузки данных

```sql
CREATE MATERIALIZED VIEW default.events_mv ON CLUSTER replicated_cluster
TO default.events_local
AS
SELECT
    event_id,
    user_id,
    event_type,
    event_time
FROM default.events_kafka;
```

![📸 Скриншот 5: SHOW CREATE TABLE events_mv](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_show_events_mv.png)

Как это работает:
1.	Kafka Engine получает пачку сообщений
2.	MV обрабатывает SELECT
3.	Данные пишутся в events_local
4.	Если вставка успешна — offset коммитится

Важно:

MV должен быть создан ПОСЛЕ Kafka таблицы.

---

## Шаг 6. Запись данных в Kafka

Тестовая отправка сообщений:

```bash
docker exec -it kafka kafka-console-producer \
  --topic events_topic \
  --bootstrap-server kafka:9092
```
Пример сообщений:
```json
{"event_id":1,"user_id":101,"event_type":"click","event_time":"2024-01-01 10:00:00"}
{"event_id":2,"user_id":102,"event_type":"view","event_time":"2024-01-01 10:01:00"}
{"event_id":3,"user_id":103,"event_type":"purchase","event_time":"2024-01-01 10:02:00"}
```
![📸 Скриншот 6: отправка сообщений в Kafka](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_events_in_kafka.png)

---

## Шаг 7. Проверка загрузки данных в ClickHouse

Проверка таблицы:

```sql
SELECT *
FROM default.events_dist
ORDER BY event_time;
```
Ожидаемый результат:

| event_id | user_id | event_type | event_time           |
|----------|----------|------------|----------------------|
| 1        | 101      | click      | 2024-01-01 10:00:00  |
| 2        | 102      | view       | 2024-01-01 10:01:00  |
| 3        | 103      | purchase   | 2024-01-01 10:02:00  |

![📸 Скриншот 7: результат SELECT из events_dist](https://github.com/realexpert1C/clickhouse-course/blob/13b0d8d72e4474a6dcb90651b5c1f3ad327ff64e/images/hw24_check_events_dist.png)

---

## Шаг 8. Проверка offset и состояния consumer group

Проверка через Kafka:

```bash
docker exec -it kafka kafka-consumer-groups \
  --bootstrap-server kafka:9092 \
  --describe \
  --group clickhouse_consumer_group
```

![📸 Скриншот 8: offset = 0 (lag отсутствует)]()
offset = 0 (lag отсутствует)

📌 Что видим

Partition 2

- PARTITION        2
- CURRENT-OFFSET   5
- LOG-END-OFFSET   5
- LAG              0
- CONSUMER-ID      ClickHouse-ch1-...

Это означает:

- В partition 2 записано 5 сообщений
- ClickHouse их полностью прочитал
- Offset закоммичен
- Отставания (lag) нет

Остальные partition (0,1,3,4,5)

* CURRENT-OFFSET   -
* LOG-END-OFFSET   0
* LAG              -

Это означает:
- В этих partition нет сообщений, поэтому offset не установлен
- Lag не считается

Это нормально.

---

## Шаг 9 ⭐ Задание со звездочкой

### 9.1 Запись в Kafka через ClickHouse

Можно использовать таблицу Kafka как producer:

```sql
INSERT INTO events_kafka
VALUES (10, 200, 'test', now());
```
Проверяю вставку запросом 

```sql
SELECT *
FROM default.events_dist
ORDER BY event_time;
```
Данные добавились

![📸 Скриншот 9: INSERT из CH успешен]()

---

### 9.2 Пайплайн без Kafka Engine

В рамках альтернативного варианта ingestion реализован поток обработки без использования Kafka Engine в ClickHouse.

Архитектура:

Kafka → Python Consumer (confluent-kafka) → Airflow DAG → HTTP INSERT → ClickHouse (MergeTree)

Компоненты:
1.	Airflow DAG
2.	Python consumer (confluent-kafka)
3.	Batch insert в ClickHouse через HTTP API
4.	Таблица назначения: default.events_local

Логика работы DAG

DAG выполняет следующие шаги:
1.	Подключается к Kafka
2.	Читает сообщения из events_topic
3.	Формирует batch
4.	Отправляет batch в ClickHouse через HTTP API
5.	Коммитит offset

Структура файла DAG

Файл размещён в каталоге:

airflow/dags/kafka_to_clickhouse.py


Содержимое файла kafka_to_clickhouse.py:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime
from confluent_kafka import Consumer
import requests
import json

KAFKA_CONFIG = {
    'bootstrap.servers': 'kafka:9092',
    'group.id': 'airflow_consumer_group',
    'auto.offset.reset': 'earliest'
}

CLICKHOUSE_URL = "http://ch1:8123/?query=INSERT INTO default.events_local FORMAT JSONEachRow"

def consume_and_insert():

    consumer = Consumer(KAFKA_CONFIG)
    consumer.subscribe(['events_topic'])

    messages = []
    batch_size = 100

    try:
        while len(messages) < batch_size:
            msg = consumer.poll(timeout=1.0)
            if msg is None:
                break
            if msg.error():
                continue

            messages.append(msg.value().decode('utf-8'))

        if messages:
            payload = "\n".join(messages)
            response = requests.post(CLICKHOUSE_URL, data=payload)

            if response.status_code != 200:
                raise Exception(f"ClickHouse insert failed: {response.text}")

            consumer.commit()

    finally:
        consumer.close()


with DAG(
    dag_id="kafka_to_clickhouse_streaming",
    start_date=datetime(2024, 1, 1),
    schedule="@once",
    catchup=False,
) as dag:

    streaming_task = PythonOperator(
        task_id="consume_and_insert",
        python_callable=consume_and_insert
    )

    streaming_task

```

Размещение зависимостей

В контейнер Airflow должны быть установлены библиотеки:

```bash
pip install confluent-kafka requests
```
Если Dockerfile кастомный как у меня ( я ранее уже устанавливал Airflow с зависимостями), то новая редакция Dockerfile:
```bash
FROM apache/airflow:3.1.7-python3.10

USER root

# системные зависимости для confluent-kafka
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        librdkafka-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

USER airflow

RUN pip install --no-cache-dir \
    clickhouse-connect==0.7.8 \
    requests==2.31.0 \
    confluent-kafka==2.3.0
```
Если используется Dockerfile для Airflow — зависимости добавляются туда.

И пересобираю Airflow:

```bash
docker compose build airflow
docker compose up -d airflow
```

Останавливаю ранее созданную MV
```sql
SYSTEM STOP VIEW default.events_mv;
```
Или отключаю Kafka-потребление
```sql
DETACH TABLE default.events_kafka;
```

Добавляю новые события:
```bash
docker exec -it kafka kafka-console-producer \
  --topic events_topic \
  --bootstrap-server kafka:9092
```
```json
{"event_id":1004,"user_id":204,"event_type":"read","event_time":"2024-01-01 12:10:00"}
{"event_id":1005,"user_id":205,"event_type":"copy","event_time":"2024-01-01 12:11:00"}
{"event_id":1006,"user_id":206,"event_type":"past","event_time":"2024-01-01 12:12:00"}
```

После добавления новых событий запускаю DAG
1.	Перейти в Airflow UI
2.	Активировать DAG kafka_to_clickhouse_streaming
3.	Запустить вручную

![📸 Скриншот: запуск DAG в Airflow UI]()

Проверка результата

После выполнения DAG проверить данные в ClickHouse:
```sql
SELECT *
FROM default.events_dist
ORDER BY event_time;
```
![📸 Скриншот: результат SELECT из events_dist]()
Данные успешно добавились


Плюсы подхода:
- гибкая трансформация
- контроль batch size (размера вставки)
- retry logic (логика повторных попыток)

Минусы:
- нет встроенного exactly-once (обработка один раз без потерь и дублей)
- требуется дополнительная логика обработки offset

---

## Выводы

В рамках задания:
* Развернут Apache Kafka
* Подключен ClickHouse кластер (1 shard, 4 replicas)
* Создан Kafka Engine
* Настроен Materialized View
* Данные успешно передаются из Kafka в MergeTree и читаются
* Проверена корректность загрузки и отсутствие lag
* Проверены альтернативные варианты загрузки данных