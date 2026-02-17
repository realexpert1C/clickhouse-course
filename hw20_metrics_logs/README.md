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
2. Настроить push/pull метрик
3. Показать дашборд со сбором метрик

### Дополнительное задание (*)
Настроить логирование через Engine=Null + ReplicatedMergeTree + MV + репликацию.

---

### Архитектура кластера

Используемый кластер:
- 1 shard
- 4 replicas: ch1, ch2, ch3, ch4
- Replication через ClickHouse Keeper
- Таблицы ReplicatedMergeTree:
  - uk_price_paid
  - uk_price_paid_daily_repl

---

# Вариант 1 — встроенный мониторинг ClickHouse
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

![📸 Скриншот 1 — Штатные метрики ClickHouse](https://github.com/realexpert1C/clickhouse-course/blob/caefb962f59713b6614180ff6700351088b1885c/images/hw20_regular_metrics.png)

---

## Шаг 2. Кастомные метрики


### 2.1 Создание базы и таблицы для кастомных dashboard

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

### 2.2 Добавление кастомных метрик

#### 1. Merge Throughput (rows/sec) - Скорость слияния строк (строк в секунду)

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

#### 2. Active Parts Count - Количество активных партов

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

#### 3. Insert Throughput (rows/sec) — Скорость вставки строк (строк в секунду)

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

### 2.3 Отображение кастомных метрик

Для загрузки кастомных метрик использую:
```sql
SELECT title, query
FROM merge('custom', '^dashboards$')
WHERE dashboard = 'Custom';
```
После этого кастомные графики отображаются в Web Dashboard.

![📸 Скриншот 2 — Все кастомные метрики в Dashboard](https://github.com/realexpert1C/clickhouse-course/blob/caefb962f59713b6614180ff6700351088b1885c/images/hw20_custom_metrics.png)

---

Итог:
* Штатные системные метрики Clickhouse позволяют мониторить обширный ряд показателей работы и использования ресурсов.
* Добавление кастомных метрик позволяет выходить за рамки системных метрик и настраивать более детальную аналитику.

---

# Вариант 2 — Внешний мониторинг (Prometheus + Grafana)

## Шаг 1 Установка Prometheus и Grafana в контейнерах Docker

Создаю папки для контейнеров мониторинга

```bash
mkdir -p ~/monitoring/prometheus
mkdir -p ~/monitoring/prometheus/data
mkdir -p ~/monitoring/grafana
mkdir -p ~/monitoring/grafana/data
```
Права на каталоги
```bash
sudo chown -R 65534:65534 ~/monitoring/prometheus/data
sudo chown -R 472:472 ~/monitoring/grafana/data
```

Устанавливаю контейнеры

#### Prometheus (docker-compose)
```yml
version: '3.8'

services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    networks:
      - infra-net
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./data:/prometheus
    ports:
      - "9090:9090"

networks:
  infra-net:
    external: true
```

#### prometheus.yml

```yml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: clickhouse
    static_configs:
      - targets: ['ch1:9363','ch2:9363','ch3:9363','ch4:9363']

  - job_name: node
    static_configs:
      - targets: ['node_exporter:9100']
```

---

#### Grafana (docker-compose)

```yml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    networks:
      - infra-net
    ports:
      - "3000:3000"
    volumes:
      - ./data:/var/lib/grafana

networks:
  infra-net:
    external: true
```

Проверка в браузере

http://IP:9090 и http://IP:3000

Grafana по умолчанию логин/пароль admin/admin, новый пароль admin123

## Шаг 2. Включаю экспорт метрик ClickHouse

В файл /etc/clickhouse-server/config.xml добавляю

```xml
<clickhouse>
    <prometheus>
        <endpoint>/metrics</endpoint>
        <port>9363</port>
        <metrics>true</metrics>
        <events>true</events>
        <asynchronous_metrics>true</asynchronous_metrics>
    </prometheus>
</clickhouse>
```


```bash
sudo systemctl restart clickhouse-server
```
Проверка - задаю в web интерфейсе Prometheus запросы для проверки:

→ Graph → вставить запрос → Execute → Graph

✅ Количество выполняемых запросов в секунду

`rate(ClickHouseProfileEvents_Query[1m])`

✅ Количество активных партов

`ClickHouseMetrics_PartsActive`

![📸 СКРИНШОТ: метрики в Prometheus](https://github.com/realexpert1C/clickhouse-course/blob/81c2950e5fa491eaab4196de6e9ac0c8576978c2/images/hw20_prom_clean.png)

---

## Шаг 3. Устанавливаю Node Exporter для отслеживания работы железа

Содержимое docker-compose.yml для Node Exporter:

```yml
version: '3.8'

services:

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    restart: unless-stopped
    networks:
      - infra-net
    pid: "host"
    volumes:
      - /:/host:ro,rslave
    command:
      - '--path.rootfs=/host'

networks:
  infra-net:
    external: true
```
Проверка с помощью запросов:

#### CPU загрузка
```promql
rate(node_cpu_seconds_total{mode!="idle"}[1m])
```

#### Использование памяти
```promql
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
```

![📸 СКРИНШОТ: CPU загрузка](https://github.com/realexpert1C/clickhouse-course/blob/dc6902e8e5da678566874d4d6866b6e0a8a326ff/images/hw20_node_exp1.png)

![📸 СКРИНШОТ: Использование памяти](https://github.com/realexpert1C/clickhouse-course/blob/dc6902e8e5da678566874d4d6866b6e0a8a326ff/images/hw20_node_exp2.png)

---


## Шаг 4. Настройка Grafana

✅ Добавить Prometheus как Data Source
1.	Data sources
3.	Add data source
4.	Выбрать Prometheus

В поле URL указать: http://IP:9090
(если Grafana и Prometheus в одной docker-сети infra-net)

Нажать: Save & Test

---

✅ Проверить метрики в Grafana
1.	Перейти в Dashboards
2.	Нажать New → New dashboard
3.	Add new panel
4.	В поле запроса вставить, например:

rate(ClickHouseProfileEvents_Query[1m])

5.	Нажать Apply

---

✅ Альтернатива (импорт готового дашборда)
1.	Dashboards → Import
2.	Ввести ID:

* Node Exporter → ID 11074
* ClickHouse → ID 14192

3.	Выбрать Prometheus datasource
4.	Import

![📸 СКРИНШОТ: Grafana dashboard]()

---

# Дополнительное задание (логирование)

### 1. Таблица логов Engine=Null

```sql
CREATE TABLE logs_null
(
    event_time DateTime,
    message String
) ENGINE = Null;
```
---

### 2. Реплицируемая таблица

```sql
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
```

---

### 3. Materialized View

CREATE MATERIALIZED VIEW mv_logs TO logs_repl AS
SELECT *, hostName() FROM logs_null;


---

### 4. Проверка репликации

INSERT INTO logs_null VALUES (now(), 'test log');

SELECT * FROM logs_repl;

📸 СКРИНШОТ: запись появилась на всех репликах

---

Итоги

Настроен полный мониторинг:

Тип	Инструмент
ClickHouse внутренние метрики	Advanced Dashboard
Метрики ОС	Node Exporter
Метрики ClickHouse	Prometheus
Визуализация	Grafana
Репликация логов	Engine=Null + MV

