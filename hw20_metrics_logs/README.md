# Домашнее задание — Мониторинг ClickHouse

## Условие задания

Цель: настроить мониторинг и оптимизировать производительность ClickHouse.

Дано 2 варианта выполнения:

### Вариант 1 — встроенный мониторинг ClickHouse
1. Придумать запросы для персонализированного мониторинга
2. Создать таблицу с запросами в нужном формате
3. Показать скриншот встроенного Web Dashboard ClickHouse

### Вариант 2 — внешний мониторинг
1. Развернуть Prometheus / Grafana
2. Настроить сбор метрик ClickHouse и OS
3. Показать дашборд со сбором метрик

### Дополнительное задание (*)
Настроить логирование через Engine=Null + ReplicatedMergeTree + MV + репликацию.

---

# Архитектура кластера

Используемый кластер:
- 1 shard
- 4 replicas: ch1, ch2, ch3, ch4
- Replication через ClickHouse Keeper
- Таблицы ReplicatedMergeTree:
  - uk_price_paid
  - uk_price_paid_daily_repl

---


## Шаг 1. Штатные метрики ClickHouse (Advanced Dashboard)

ClickHouse по умолчанию предоставляет встроенный Web Dashboard:

http://:8123/dashboard

На нем уже доступны следующие метрики:

- Queries / second  
- CPU Usage (cores)  
- Queries Running  
- Merges Running  
- Selected Bytes / second  
- IO Wait  
- CPU Wait  
- Read From Disk  
- Read From Filesystem  
- Memory (tracked)  
- Load Average (15 minutes)  
- Selected Rows / second  
- Inserted Rows / second  
- Total MergeTree Parts  
- Max Parts For Partition  
- Concurrent Network Connections  

Эти метрики отражают:

- Нагрузку на CPU
- IO
- Количество запросов
- Количество частей
- Системные показатели

![📸 Скриншот 1 — Штатные метрики ClickHouse]()

---

# Шаг 2. Кастомные метрики

Штатные метрики показывают ресурсы,  
но не показывают:

- Ошибки запросов
- Проблемы репликации
- Очереди задач
- Блокировки
- Saturation background pool
- Долгие запросы

Поэтому добавлены кастомные метрики:

1. Failed Queries  
2. Delayed Inserts  
3. Replication Absolute Delay  
4. Replication Queue Size  
5. Background Pool Usage  
6. Context Lock Wait  
7. Long Running Queries  
8. Replication Errors  

---

## 2.1 Создание базы и таблицы для кастомных dashboard

```sql
CREATE DATABASE IF NOT EXISTS custom;

CREATE TABLE custom.dashboards
(
    dashboard String,
    title String,
    query String
)
ENGINE = MergeTree
ORDER BY tuple();
```

---

## 2.2 Добавление кастомных метрик

### 1. Merge Throughput (rows/sec) - Скорость слияния строк (строк в секунду)

```sql
INSERT INTO custom.dashboards VALUES
(
'Custom',
'Merge Throughput (rows/sec)',
'SELECT
  toStartOfInterval(event_time, INTERVAL {rounding:UInt32} SECOND)::INT AS t,
  coalesce(max(ProfileEvent_MergedRows)
           - min(ProfileEvent_MergedRows), 0)
FROM merge(''system'', ''^metric_log$'')
WHERE event_time >= now() - {seconds:UInt32}
GROUP BY t
ORDER BY t
'
);
```
---

### 2. Active Parts Count - Количество активных партов

```sql
INSERT INTO custom.dashboards VALUES
(
'Custom',
'Active Parts Count',
'SELECT
  toStartOfInterval(event_time, INTERVAL {rounding:UInt32} SECOND)::INT AS t,
  coalesce(avg(CurrentMetric_PartsActive), 0)
FROM merge(''system'', ''^metric_log$'')
WHERE event_time >= now() - {seconds:UInt32}
GROUP BY t
ORDER BY t
WITH FILL STEP {rounding:UInt32}
'
);
```
---

### 3. Insert Throughput (rows/sec) — Скорость вставки строк (строк в секунду)

```sql
INSERT INTO custom.dashboards VALUES
(
'Custom',
'Insert Throughput (rows/sec)',
'SELECT
  toStartOfInterval(event_time, INTERVAL {rounding:UInt32} SECOND)::INT AS t,
  coalesce(max(ProfileEvent_InsertedRows)
           - min(ProfileEvent_InsertedRows), 0)
FROM merge(''system'', ''^metric_log$'')
WHERE event_time >= now() - {seconds:UInt32}
GROUP BY t
ORDER BY t
'
);
```
---

## 2.3 Отображение кастомных метрик

Для загрузки кастомных метрик использую:
```sql
SELECT title, query
FROM merge('custom', '^dashboards$')
WHERE dashboard = 'Custom';
```
После этого кастомные графики отображаются в Web Dashboard.

![📸 Скриншот 2 — Все кастомные метрики в Dashboard]()

---

Итог

Добавленные кастомные метрики позволяют:
	•	Контролировать репликацию
	•	Выявлять ошибки
	•	Отслеживать блокировки
	•	Контролировать saturation background pool
	•	Выявлять долгие запросы

Таким образом мониторинг выходит за рамки системных метрик и позволяет диагностировать реальные production-проблемы.

---

Часть 2 — Внешний мониторинг (Prometheus + Grafana)

1. Включаем экспорт метрик ClickHouse

Файл /etc/clickhouse-server/config.xml

<prometheus>
    <endpoint>/metrics</endpoint>
    <port>9363</port>
    <metrics>true</metrics>
    <events>true</events>
    <asynchronous_metrics>true</asynchronous_metrics>
</prometheus>

sudo systemctl restart clickhouse-server

Проверка:

curl http://localhost:9363/metrics

📸 СКРИНШОТ: метрики открываются

⸻

2. Установка Node Exporter

sudo useradd -rs /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.9.0.linux-amd64.tar.gz
tar -xvf node_exporter*.tar.gz
sudo mv node_exporter*/node_exporter /usr/local/bin/

Service:

sudo nano /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter

[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target

sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter

📸 СКРИНШОТ: http://localhost:9100/metrics

⸻

3. Настройка Prometheus

Файл /etc/prometheus/prometheus.yml

scrape_configs:

  - job_name: clickhouse
    static_configs:
      - targets: ['ch1:9363']

  - job_name: node
    static_configs:
      - targets: ['ch1:9100']

sudo systemctl restart prometheus

📸 СКРИНШОТ: http://localhost:9090/targets

⸻

4. Grafana

Импорт дашбордов:
	•	Node Exporter → ID 11074
	•	ClickHouse → ID 14192

📸 СКРИНШОТ: Grafana dashboard

⸻

Часть 3 — Доп. задание (логирование)

1. Таблица логов Engine=Null

CREATE TABLE logs_null
(
    event_time DateTime,
    message String
) ENGINE = Null;


⸻

2. Реплицируемая таблица

CREATE TABLE logs_repl
(
    event_time DateTime,
    message String,
    replica String DEFAULT hostName()
)
ENGINE = ReplicatedMergeTree(
'/clickhouse/tables/{shard}/logs_repl',
'{replica}'
)
ORDER BY event_time;


⸻

3. Materialized View

CREATE MATERIALIZED VIEW mv_logs TO logs_repl AS
SELECT *, hostName() FROM logs_null;


⸻

4. Проверка репликации

INSERT INTO logs_null VALUES (now(), 'test log');

SELECT * FROM logs_repl;

📸 СКРИНШОТ: запись появилась на всех репликах

⸻

Итог

Настроен полный мониторинг:

Тип	Инструмент
ClickHouse внутренние метрики	Advanced Dashboard
Метрики ОС	Node Exporter
Метрики ClickHouse	Prometheus
Визуализация	Grafana
Репликация логов	Engine=Null + MV

