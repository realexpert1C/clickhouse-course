Ниже — готовый текст для README.md. Структура соответствует критериям проверки. В нужных местах добавлены пометки для скриншотов.

⸻

Домашнее задание 24

Интеграция ClickHouse с Apache Kafka

Цель работы

Изучить интеграцию ClickHouse с Apache Kafka и реализовать пайплайн записи и чтения данных:
	•	запись данных в Kafka
	•	чтение данных через Kafka Engine
	•	загрузка в MergeTree через Materialized View
	•	проверка корректности загрузки

⸻

1. Архитектура стенда

Инфраструктура
	•	ClickHouse Cluster
	•	1 shard
	•	4 replicas: ch1, ch2, ch3, ch4
	•	Репликация через ClickHouse Keeper (развернут на ch1)
	•	Airflow 3.1.7
	•	Python 3.10
	•	Executor: LocalExecutor
	•	Docker Compose
	•	Общая сеть: infra-net
	•	Apache Kafka (развернут в Docker)

⸻

2. Разворачивание Apache Kafka

Kafka развернут через Docker Compose.

Пример сервиса:

kafka:
  image: confluentinc/cp-kafka:7.5.0
  container_name: kafka
  ports:
    - "9092:9092"
  environment:
    KAFKA_BROKER_ID: 1
    KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  networks:
    - infra-net

Создание топика:

docker exec -it kafka kafka-topics \
  --create \
  --topic events_topic \
  --bootstrap-server kafka:9092 \
  --partitions 3 \
  --replication-factor 1

📸 Скриншот 1: список топиков (kafka-topics --list)

⸻

3. Настройка ClickHouse для работы с Kafka

Kafka Engine используется на уровне таблицы.

Проверка доступности Kafka из контейнера ClickHouse:

docker exec -it ch1 ping kafka

📸 Скриншот 2: успешный ping kafka из ch1

⸻

4. Создание таблиц в ClickHouse

Работа выполнялась на кластере через ON CLUSTER.

4.1 Таблица назначения (MergeTree)

CREATE TABLE default.events_local ON CLUSTER cluster_1sh_4rep
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

Distributed таблица

CREATE TABLE default.events_dist ON CLUSTER cluster_1sh_4rep
AS default.events_local
ENGINE = Distributed(cluster_1sh_4rep, default, events_local, rand());

📸 Скриншот 3: SHOW TABLES на кластере

⸻

5. Создание Kafka Engine таблицы

CREATE TABLE default.events_kafka ON CLUSTER cluster_1sh_4rep
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

📸 Скриншот 4: DESCRIBE TABLE events_kafka

⸻

6. Materialized View для загрузки данных

CREATE MATERIALIZED VIEW default.events_mv ON CLUSTER cluster_1sh_4rep
TO default.events_local
AS
SELECT
    event_id,
    user_id,
    event_type,
    event_time
FROM default.events_kafka;

📸 Скриншот 5: SHOW CREATE TABLE events_mv

⸻

7. Запись данных в Kafka

Тестовая отправка сообщений:

docker exec -it kafka kafka-console-producer \
  --topic events_topic \
  --bootstrap-server kafka:9092

Пример сообщений:

{"event_id":1,"user_id":101,"event_type":"click","event_time":"2024-01-01 10:00:00"}
{"event_id":2,"user_id":102,"event_type":"view","event_time":"2024-01-01 10:01:00"}
{"event_id":3,"user_id":103,"event_type":"purchase","event_time":"2024-01-01 10:02:00"}

📸 Скриншот 6: отправка сообщений в Kafka

⸻

8. Проверка загрузки данных в ClickHouse

Проверка таблицы:

SELECT *
FROM default.events_dist
ORDER BY event_time;

Ожидаемый результат:

event_id	user_id	event_type	event_time
1	101	click	2024-01-01 10:00:00
2	102	view	2024-01-01 10:01:00
3	103	purchase	2024-01-01 10:02:00

📸 Скриншот 7: результат SELECT из events_dist

⸻

9. Проверка offset и состояния consumer group

Проверка через Kafka:

kafka-consumer-groups \
  --bootstrap-server kafka:9092 \
  --describe \
  --group clickhouse_consumer_group

📸 Скриншот 8: offset = 0 (lag отсутствует)

⸻

⭐ Задание со звездочкой

9.1 Запись в Kafka через ClickHouse

Можно использовать таблицу Kafka как producer:

INSERT INTO events_kafka
VALUES (10, 200, 'test', now());


⸻

9.2 Пайплайн без Kafka Engine

Альтернативный вариант:
	1.	Airflow DAG
	2.	Python consumer (confluent-kafka)
	3.	Batch insert в ClickHouse через HTTP API

Пример вставки:

import requests

data = '{"event_id":11,"user_id":201,"event_type":"click","event_time":"2024-01-01 12:00:00"}'

requests.post(
    "http://ch1:8123/?query=INSERT INTO default.events_local FORMAT JSONEachRow",
    data=data
)

Плюсы подхода:
	•	гибкая трансформация
	•	контроль batch size
	•	retry logic

Минусы:
	•	нет встроенного exactly-once
	•	требуется дополнительная логика обработки offset

⸻

10. Выводы

В рамках задания:
	•	Развернут Apache Kafka
	•	Настроен ClickHouse кластер (1 shard, 4 replicas)
	•	Создан Kafka Engine
	•	Настроен Materialized View
	•	Данные успешно передаются из Kafka в MergeTree
	•	Проверена корректность загрузки и отсутствие lag

Пайплайн работает корректно.

⸻

Ссылка на репозиторий

👉 [Вставить ссылку на GitHub репозиторий]

⸻

Если нужно, могу дополнительно:
	•	добавить схему архитектуры
	•	добавить расширенный пример с Avro/Protobuf
	•	добавить production-рекомендации (HA Kafka + 3 brokers + replication 3)
	•	оформить версию для защиты (короткая версия на 3–4 страницы)