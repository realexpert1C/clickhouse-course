
# Проектная работа по курсу обучения OTUS "Clickhouse для инженеров и архитекторов БД"

# Тема. Аналитика в реальном времени на базе ClickHouse. Сравнительный анализ batch и streaming загрузки данных при неравномерной нагрузке
---

## 1. Обзор проекта

### 🎯 1.1. Цель проекта

Разработать и продемонстрировать MVP аналитической платформы на базе ClickHouse для сравнения двух подходов к загрузке данных:
- пакетная загрузка (batch ingestion)
- потоковая загрузка (streaming ingestion)

в условиях неравномерной нагрузки, приближенной к реальной производственной эксплуатации.

Проект моделирует ситуацию, когда параллельные вставки данных в ClickHouse могут приводить к:
- накоплению большого количества частей данных (small parts)
- отставанию процессов слияния (merge backlog)
- росту нагрузки на CPU и дисковую подсистему
- увеличению времени выполнения аналитических запросов

Основная цель эксперимента — показать различие поведения ClickHouse при __batch__ и __streaming__ ingestion и оценить стоимость near real-time загрузки данных с точки зрения ресурсов системы.

---

### 1.2. Моделируемая производственная проблема

В реальных аналитических системах часто встречается следующая нагрузка:
- ~1 000 000 и более событий в сутки
- выраженная неравномерность потока (пики днём, спад ночью)
- одновременная загрузка данных из разных источников

Типичный сценарий:
- потоковые события поступают в систему практически в реальном времени
- параллельно выполняются ETL-процессы для BI-аналитики

Это может приводить к:
- конкуренции за ресурсы
- накоплению большого количества мелких частей данных
- росту нагрузки на merge-процессы
- деградации производительности аналитических запросов

В рамках проекта данный сценарий воспроизводится в контролируемых условиях.

---

### 1.3. Архитектурная концепция проекта

Проект представляет собой лабораторную платформу для исследования поведения ClickHouse при:
- неравномерной нагрузке (bursty traffic)
- параллельной загрузке данных
- сравнении batch ingestion и streaming ingestion

Для воспроизведения нагрузки используется исторический набор событий, который воспроизводится с сохранением исходного временного профиля.

__Streaming__-поток имитирует near real-time ingestion через Kafka,
__batch__-поток имитирует классический ETL-процесс периодической загрузки данных.

---

### 1.4. Источник данных (Source of Truth)

Перед запуском системы выполняется однократная выгрузка исторических данных:
- все сделки Binance (aggTrades) по нескольким торговым парам
- период: июль 2023 года

Данные:
- сохраняются локально
- не изменяются
- используются как единственный источник событий

Формат хранения:

`Parquet`

Файл размещается в:

`S3-совместимое хранилище MinIO`

Это обеспечивает:
- воспроизводимость экспериментов
- независимость от Binance API
- возможность повторного запуска эксперимента
- контроль над нагрузкой

---

Дополнительный датасет

Для имитации реальных аналитических задач используется дополнительный набор данных:

`Потребление электроэнергии на территории США в июле 2023 года`

Данные:
- загружаются напрямую в ClickHouse
- используются как reference-таблица
- применяются в JOIN-операциях при построении аналитических витрин

---

### 1.5. Архитектура потоков данных

Исторический датасет используется для формирования двух независимых потоков загрузки:
- streaming ingestion
- batch ingestion

<details>
<summary>Основная архитектура потоков данных</summary>

```mermaid
flowchart TD

A[Historical Dataset<br>Binance Trades<br>July 2023]
B[Parquet Storage]
C[MinIO S3 Storage]
D[Apache Airflow<br>Pipeline Orchestrator]

E[Streaming Pipeline]
F[Batch Pipeline]

G[Apache Kafka]
H[Batch Buffer<br>Airflow DAG]

I[ClickHouse Kafka Engine]
J[INSERT INTO ClickHouse]

K[Raw Streaming Table<br>ReplicatedMergeTree]
L[Raw Batch Table<br>ReplicatedMergeTree]

M[Aggregation Layer<br>Materialized Views]
N[Data Mart Layer<br>Aggregated Tables]

O[Electricity Consumption<br>Reference Table]

P[Yandex DataLens]

Q[Streaming Dashboard]
R[Batch Dashboard]

A --> B
B --> C
C --> D

D --> E
D --> F

E --> G
G --> I
I --> K

F --> H
H --> J
J --> L

K --> M
L --> M

M --> N
N --> O

O --> P

P --> Q
P --> R
```

</details>
</br>


Apache Airflow выполняет роль оркестратора загрузки данных, управляя формированием потоков и периодичностью выполнения задач.

---

### 1.6. Streaming Pipeline

__Streaming__ реализуется через:
- Kafka Topic
- ClickHouse Kafka Engine
- Materialized View

Поток данных:

`Airflow → Kafka → ClickHouse Kafka Engine → Raw Table`

Особенности streaming ingestion:
- события поступают практически в реальном времени (каждую секунду)
- вставки выполняются небольшими блоками (максимальный размер блока 5000 строк)
- создаётся большое количество частей данных

Это позволяет наблюдать:
- рост числа `parts`
- нагрузку на `merge` - процессы
- влияние частых вставок на производительность системы.

---

### 1.7. Batch Pipeline

__Batch__ ingestion моделирует классический ETL-подход.

Airflow формирует буфер данных и выполняет вставку в ClickHouse при выполнении одного из условий:
- накоплено 50 000 строк
- прошло 15 минут с момента предыдущей вставки

Таким образом:
- минимальный размер вставки — 50 000 строк
- вставки выполняются не чаще 4 раз в час

Поток данных:

`Airflow → Batch Buffer → INSERT INTO ClickHouse → Raw Batch Table`

Это приводит к:
- меньшему количеству частей данных
- более стабильной работе `merge` - процессов
- меньшей нагрузке на систему.

---

### 1.8. Архитектура хранения данных

Архитектура хранения данных построена по многоуровневой модели, включающей COLD, HOT и WARM уровни хранения, что позволяет эффективно управлять ingestion, обработкой и аналитическим использованием данных

<details>
<summary>Схема хранения данных</summary>

```mermaid
flowchart TB

COLD[Historical Dataset<br>Parquet Files<br>MinIO Storage]

HOT[Raw Data Layer<br>Streaming / Batch Tables<br>ReplicatedMergeTree]

WARM[Aggregation Layer<br>Materialized Views<br>Aggregated Tables]

MART[Data Mart Layer<br>Analytical Tables]

BI[Yandex DataLens<br>Dashboards]

COLD --> HOT
HOT --> WARM
WARM --> MART
MART --> BI
```

</details>
</br>

__Historical Dataset (COLD)__

Назначение:
- хранение исходного исторического набора данных сделок Binance
- обеспечение полной воспроизводимости эксперимента
- исключение зависимости от внешних API во время работы системы
- использование датасета как единственного источника истины (Source of Truth) для генерации потоков данных

Реализация:
- формат хранения: Parquet
- объектное хранилище: MinIO (S3-compatible storage)
- источник данных: Binance aggTrades
- период данных: июль 2023 года
- датасет предварительно масштабирован с коэффициентом интенсивности ×5

Поскольку датасет является immutable, это позволяет заново воспроизвести полный цикл эксперимента.

__Raw Layer (HOT)__

Назначение:
- хранение детализированных сделок
- приём streaming и batch вставок

Реализация:
- ReplicatedMergeTree
- партиционирование по дате
- сортировка по (symbol, event_time, agg_trade_id)

__Streaming__ и __batch__ данные сохраняются в разные таблицы, что позволяет сравнивать поведение системы при различных способах ingestion.

Очистка данных выполняется после завершения цикла воспроизведения данных.


__Aggregation Layer (WARM)__

На основе raw-данных формируются агрегаты с использованием Materialized Views.

Агрегации:
- 1 минута
- 15 минут
- 1 час
- 1 день

Агрегированные данные используются для аналитических витрин.


__Data Mart Layer__

Формируются витрины данных, которые:
- используют агрегированные сделки
- выполняют JOIN с таблицей потребления электроэнергии

JOIN выполняется по часовым интервалам.

Этот слой используется BI-инструментами.

---

### 1.9. Визуализация

Для визуализации используется:

Yandex DataLens

Создаются два идентичных дашборда:

1️⃣ Dashboard A — данные batch ingestion

2️⃣ Dashboard B — данные streaming ingestion

Фильтры синхронизированы.

Дополнительно отображаются контрольные метрики:
- общее количество строк
- агрегированные значения
- показатели активности рынка

Это позволяет визуально сравнивать результаты двух способов загрузки данных.

---

### 1.10. Мониторинг и наблюдаемость системы

Цель мониторинга — наблюдать поведение системы при различных режимах ingestion.

<details>
<summary>Схема архитектуры мониторинга</summary>

```mermaid
flowchart TD

A[ClickHouse Metrics]
B[Kafka Metrics]
C[Airflow Metrics]
D[Node Exporter<br>System Metrics]

E[Prometheus<br>Metrics Storage]

F[Grafana<br>Monitoring UI]

A --> E
B --> E
C --> E
D --> E

E --> F
```

</details>
</br>

Используются:
- Prometheus — сбор метрик
- Grafana — визуализация

#### 1.10.1. Мониторинг ClickHouse

Основные метрики:
- Insert rate
- PartsNumber
- BackgroundMergesAndMutationsPoolTask
- ReplicasMaxQueueSize
- DelayedInserts
- QueryDuration

Анализируется:
- рост количества частей
- загрузка merge-процессов
- задержки вставок
- влияние нагрузки на аналитические запросы.

#### 1.10.2. Мониторинг Kafka

Основные метрики:
- consumer lag
- messages in per second
- messages out per second
- producer error rate

Позволяет выявлять:
- накопление lag
- перегрузку топиков
- проблемы доставки сообщений.

#### 1.10.3. Мониторинг Apache Airflow

Для контроля корректности выполнения ETL-процессов используется мониторинг Airflow.

Контролируются:
- состояние DAG
- длительность выполнения задач
- количество ошибок
- зависшие или не завершённые задачи

Основные метрики:
- DAG run status
- task duration
- failed tasks
- scheduler health

Метрики Airflow также экспортируются в Prometheus и отображаются в Grafana, что позволяет отслеживать состояние пайплайнов без необходимости заходить в интерфейс Airflow.

#### 1.10.4. Системные метрики

Собираются через Node Exporter.

Метрики:
- CPU usage
- RAM usage
- IO wait
- Disk write rate
- Disk space usage

Позволяют оценивать влияние различных способов загрузки данных на ресурсы сервера.

---

### 1.11. Циклический режим работы

После завершения воспроизведения исторического датасета:
1.	выполняется очистка raw-таблиц
2.	счётчики сбрасываются
3.	воспроизведение начинается заново

Таким образом создаётся замкнутый лабораторный цикл.

<details>
<summary>Схема жизненного цикла данных</summary>

```mermaid
flowchart TD

A[Streaming Insert<br>Kafka Engine]
B[Batch Insert<br>Airflow]

C[Raw Tables<br>ReplicatedMergeTree]

D[New Data Parts<br>Created by INSERT]

E[Parts Storage Layer]

F[Background Merge Processes]

G[Merged Large Parts]

H[Aggregation Layer<br>Materialized Views]

I[Aggregated Tables<br>minute / hour / day]

J[Data Mart]

K[BI Layer<br>Yandex DataLens]

A --> C
B --> C

C --> D
D --> E

E --> F
F --> G

G --> H
H --> I

I --> J
J --> K
```

</details>
</br>

---

### 1.12. Кластер ClickHouse

Конфигурация:
- 1 shard
- 2 replicas
- 3 ClickHouse Keeper

Кластер обеспечивает:
- репликацию данных
- отказоустойчивость
- корректную работу распределённых таблиц.


✅ Представленная архиектура проекта:
-	соответствует цели реализации проекта
- соответствует требованиям к проектной работе

---
### 1.13. Инфраструктура и аппаратная конфигурация (Deployment Environment)

1. Сервер VPS для использования публичного IP и доступа из Интернет для демонстрации работы системы:
- CPU: 1 vCPU
- RAM: 1 GB
- Storage: NVMe SSD 15GB
- OS: Ubuntu 22.04 LTS

2. Локальный сервер, на котором разворачиваются контейнеры и хранятся данные:
- CPU: 12 vCPU
- RAM: 40 GB
- Storage: NVMe SSD 200GB
- OS: Ubuntu 22.04 LTS
- Docker: 29.2.1
--- 

### 1.14. Архитектурная ценность проекта

Архитектура демонстрирует:
* различия batch и streaming ingestion,
* влияние неравномерной нагрузки,
* влияние масштабирования интенсивности,
* поведение ClickHouse при burst-нагрузке,
* управление жизненным циклом данных,
* построение production-подобной архитектуры аналитической платформы.


## 2. Реализация проекта

### Шаг 0 - Базовая инфраструктура 

🎯 Цель:

Подготовить сервер, сеть и ClickHouse кластер.

Что входит:
#### Установка OS Ubuntu Server 22.04 LTS 
  Операционная система развернута из стандартного дистрибутива с сайта (https://releases.ubuntu.com/22.04/). Имя сервера - `ch-lab`
#### Установка Docker и Docker Compose на `ch-lab` 
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Проверка:
```bash
docker --version
docker compose version
```

#### Развертывание кластера ClickHouse cluster (1 shard, 2 replicas) и ClickHouse Keeper (3 nodes)
На сервере `ch-lab`:

🧱 1. Создание структуры каталогов
```bash
mkdir -p ~/infra/ch-cluster
cd ~/infra/ch-cluster

mkdir -p ch1/config.d
mkdir -p ch2/config.d

mkdir -p keeper1
mkdir -p keeper2
mkdir -p keeper3

mkdir -p /data/clickhouse/ch1
mkdir -p /data/clickhouse/ch2

mkdir -p /data/keeper/keeper1
mkdir -p /data/keeper/keeper2
mkdir -p /data/keeper/keeper3
```

---

🔐 2. Настройка прав доступа

Критически важно — иначе контейнеры не стартуют.
```bash
sudo chown -R 101:101 /data/clickhouse
sudo chown -R 101:101 /data/keeper

sudo chmod -R 750 /data/clickhouse
sudo chmod -R 750 /data/keeper
```
Где:
- 101 — UID пользователя clickhouse внутри контейнера.

---

🧩 3. Конфигурация Keeper

Файл: `config.xml`
<details>
<summary>infra/ch-cluster/keeper1/config.xml</summary>

```xml
<clickhouse>
    <logger>
        <level>information</level>
        <console>true</console>
    </logger>
       
    <listen_host>0.0.0.0</listen_host>

    <keeper_server>
        <tcp_port>9181</tcp_port>

        <server_id>1</server_id>

        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
        </coordination_settings>

        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>keeper1</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>keeper2</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>keeper3</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

</details>
</br>

Для keeper2 и keeper3 меняется только:
```xml
<server_id>2</server_id>
```
и
```xml
<server_id>3</server_id>
```
---


🗂 4. Макросы `macros.xml` (обязательно для кластера)

<details>
<summary>infra/ch-cluster/ch1/config.d/macros.xml</summary>

```xml
<clickhouse>
  <macros>
    <cluster>replicated_cluster</cluster>
    <shard>01</shard>
    <replica>ch1</replica>
  </macros>
</clickhouse>
```
</details>

<details>
<summary>infra/ch-cluster/ch2/config.d/macros.xml</summary>

```xml
<clickhouse>
  <macros>
    <cluster>replicated_cluster</cluster>
    <shard>01</shard>
    <replica>ch2</replica>
  </macros>
</clickhouse>
```
</details>
</br>

---


🌐 5. Файл `remote_servers.xml` (одинаковый на нодах ch1 и ch2)

<details>
<summary>infra/ch-cluster/ch1/config.d/remote_servers.xml</summary>

```xml
<clickhouse>
  <remote_servers>
    <cluster_1>
      <shard>
        <replica>
          <host>ch1</host>
          <port>9000</port>
          <user>default</user>
          <password from_env="CLICKHOUSE_PASSWORD"/>
        </replica>
        <replica>
          <host>ch2</host>
          <port>9000</port>
          <user>default</user>
          <password from_env="CLICKHOUSE_PASSWORD"/>
        </replica>
      </shard>
    </cluster_1>
  </remote_servers>
</clickhouse>
```
</details>
</br>

---

🔗 6. Файл `zookeeper.xml` - подключение к Keeper (одинаковый на нодах ch1 и ch2)

<details>
<summary>infra/ch-cluster/ch1/config.d/zookeeper.xml</summary>

```xml
<clickhouse>
  <zookeeper>
    <node>
      <host>keeper1</host>
      <port>9181</port>
    </node>
    <node>
      <host>keeper2</host>
      <port>9181</port>
    </node>
    <node>
      <host>keeper3</host>
      <port>9181</port>
    </node>
  </zookeeper>
</clickhouse>
```
</details>
</br>

---

🌍 7. Настройка хоста для доступа
Файл `listen.xml` на ch1 и ch2 (чтобы HTTP был доступен извне)

<details>
<summary>infra/ch-cluster/ch1/config.d/listen.xml</summary>

```xml
<clickhouse>
  <listen_host>0.0.0.0</listen_host>
</clickhouse>
```
</details>
</br>

Это необходимо, потому что все элементы кластера разворачиваются в контейнерах, в которых localhost в каждом случае свой и находится внутри, а мне нужно, чтобы в контейнеры был доступ снаружи через HTTP

---

🔑 8. Файл `users_override.xml` - создание `default` пользователя (одинаковый на нодах ch1 и ch2)

<details>
<summary>infra/ch-cluster/ch1/config.d/users_override.xml</summary>

```xml
<clickhouse>
    <users>
        <default>
            <password_sha256_hex>...</password_sha256_hex>
            <networks>
                <ip>::/0</ip>
            </networks>
        </default>
    </users>
</clickhouse>
```
</details>
</br>

---

⚙️ 9. Распределение ресурсов. Файл `memory_limits.xml` (одинаковый на нодах ch1 и ch2)

<details>
<summary>infra/ch-cluster/ch1/config.d/memory_limits.xml</summary>

```xml
<clickhouse>

    <!-- Общий лимит сервера -->
    <max_server_memory_usage>8000000000</max_server_memory_usage>

    <profiles>
        <default>
            <!-- Лимит на один запрос -->
            <max_memory_usage>2000000000</max_memory_usage>

            <!-- Spill на диск -->
            <max_bytes_before_external_group_by>1000000000</max_bytes_before_external_group_by>
            <max_bytes_before_external_sort>1000000000</max_bytes_before_external_sort>
        </default>
    </profiles>

</clickhouse>
```
</details>
</br>

---
🐳 10. Файл `docker-compose.yml` (ClickHouse cluster)

<details>
<summary>infra/ch-cluster/docker-compose.yml</summary>

```yml
services:

  keeper1:
    image: clickhouse/clickhouse-keeper:25.3
    container_name: keeper1
    hostname: keeper1
    restart: unless-stopped
    user: "101:101"
    mem_limit: 1g
    cpus: "1.0"
    environment:
      CLICKHOUSE_CONFIG: /etc/clickhouse-keeper/keeper_config.xml
    volumes:
      - /data/keeper/keeper1:/var/lib/clickhouse
      - ./keeper1/config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    networks:
      - infra-net

  keeper2:
    image: clickhouse/clickhouse-keeper:25.3
    container_name: keeper2
    hostname: keeper2
    restart: unless-stopped
    user: "101:101"
    mem_limit: 1g
    cpus: "1.0"
    environment:
      CLICKHOUSE_CONFIG: /etc/clickhouse-keeper/keeper_config.xml
    volumes:
      - /data/keeper/keeper2:/var/lib/clickhouse
      - ./keeper2/config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    networks:
      - infra-net

  keeper3:
    image: clickhouse/clickhouse-keeper:25.3
    container_name: keeper3
    hostname: keeper3
    restart: unless-stopped
    user: "101:101"
    mem_limit: 1g
    cpus: "1.0"
    environment:
      CLICKHOUSE_CONFIG: /etc/clickhouse-keeper/keeper_config.xml
    volumes:
      - /data/keeper/keeper3:/var/lib/clickhouse
      - ./keeper3/config.xml:/etc/clickhouse-keeper/keeper_config.xml
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    networks:
      - infra-net

  ch1:
    image: clickhouse/clickhouse-server:25.3
    container_name: ch1
    hostname: ch1
    restart: unless-stopped
    user: "101:101"
    depends_on:
      - keeper1
      - keeper2
      - keeper3
    mem_limit: 10g
    cpus: "5.0"
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - /data/clickhouse/ch1:/var/lib/clickhouse
      - ./ch1/config.d:/etc/clickhouse-server/config.d
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    environment:
      CLICKHOUSE_USER: default
      CLICKHOUSE_PASSWORD: ${CLICKHOUSE_PASSWORD}
    networks:
      - infra-net

  ch2:
    image: clickhouse/clickhouse-server:25.3
    container_name: ch2
    hostname: ch2
    restart: unless-stopped
    user: "101:101"
    depends_on:
      - keeper1
      - keeper2
      - keeper3
    mem_limit: 10g
    cpus: "5.0"
    ports:
      - "8124:8123"
      - "9002:9000"
    volumes:
      - /data/clickhouse/ch2:/var/lib/clickhouse
      - ./ch2/config.d:/etc/clickhouse-server/config.d
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    environment:
      CLICKHOUSE_USER: default
      CLICKHOUSE_PASSWORD: ${CLICKHOUSE_PASSWORD}
    networks:
      - infra-net

networks:
  infra-net:
    name: infra-net
    driver: bridge
```
</details>
</br>

И в дополнение файл окружения `.env`:


<details>
<summary>infra/ch-cluster/.env</summary>

```bash
CLICKHOUSE_PASSWORD=*************
```
</details>
</br>

---

🚀 11. Порядок запуска

```bash
docker compose up -d keeper1 keeper2 keeper3
```
Проверка:
```bash
docker exec -it keeper1 bash -c "echo stat | nc localhost 9181"
```
Должен быть один leader и два follower.

Затем:
```bash
docker compose up -d ch1 ch2
```

🧪 Проверка кластера
```bash
docker exec -it ch1 clickhouse-client
```
В Clickhouse клиенте

```sql
SELECT * FROM system.clusters;

SELECT * FROM system.zookeeper WHERE path = '/';
```

#### Развёртывание Portainer (опционально)
Делаем, если удобнее визульный интерфейс мониторинга контейнеров

```bash
docker volume create portainer_data

docker run -d \
  -p 9001:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```
После этого Portainer доступен по: `http://<SERVER_IP>:9001`

---

#### Структура каталогов кластера:
```txt
.
├── ch1
│   └── config.d
│       ├── listen.xml
│       ├── macros.xml
│       ├── memory_limits.xml
│       ├── remote_servers.xml
│       ├── users_override.xml
│       └── zookeeper.xml
├── ch2
│   └── config.d
│       ├── listen.xml
│       ├── macros.xml
│       ├── memory_limits.xml
│       ├── remote_servers.xml
│       ├── users_override.xml
│       └── zookeeper.xml
├── docker-compose.yml
├── .env
├── keeper1
│   └── config.xml
├── keeper2
│   └── config.xml
└── keeper3
    └── config.xml

7 directories, 17 files
```


#### Результат шага
 - развернут основной сервер проекта на OS Ubuntu Server 22.04 LTS
 - на сервере создана основная часть архитектуры - кластер базы данных Clickhouse (2 ноды) в контейнерах Docker в единой локальной сети `infra-net` с управлением через Clickhouse-Keeper (3 ноды)


---

### Шаг 1 — Инфраструктура данных (Bootstrap Layer)

🎯 Цель:

Подготовить систему к работе с данными.

---

#### 1.1 Установка и настройка Kafka


* контейнер Kafka, контейнер Zookeeper
  
Подготовка каталогов
```bash
mkdir -p ~/infra/kafka/kafka-data
cd ~/infra/kafka
```
Файл `docker-compose.yaml`:

<details>
<summary>infra/kafka/docker-compose.yaml</summary>

```yml
version: "3.8"

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: kafka-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    restart: unless-stopped
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

      # внутри docker-сети:
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT

      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

      # retention 72h
      KAFKA_LOG_RETENTION_HOURS: 72
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1

    volumes:
      - ./kafka-data:/var/lib/kafka/data
    restart: unless-stopped
    networks:
      - infra-net

networks:
  infra-net:
    external: true
```
</details>
</br>

Запуск
```bash
cd ~/infra/kafka
docker compose up -d
```
* topic для сделок
  
Создание топиков (stream + batch)

<details>
<summary>Команды для создания</summary>

```bash
docker exec -it kafka kafka-topics \
  --create \
  --topic binance_trades_stream \
  --bootstrap-server kafka:9092 \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=259200000 \
  --config segment.ms=3600000
```
```bash
docker exec -it kafka kafka-topics \
  --create \
  --topic binance_trades_batch \
  --bootstrap-server kafka:9092 \
  --partitions 6 \
  --replication-factor 1 \
  --config retention.ms=259200000 \
  --config segment.ms=3600000
```
</details>
</br>

Проверка
```bash
docker exec -it kafka kafka-topics --list --bootstrap-server kafka:9092
```

* проверка доступности Kafka из ClickHouse (ch1/ch2)

<details>
<summary>Команды для проверки</summary>

```bash
docker exec -it ch1 bash -lc '</dev/tcp/kafka/9092' && echo "ch1 -> kafka OK"
docker exec -it ch2 bash -lc '</dev/tcp/kafka/9092' && echo "ch2 -> kafka OK"
```
</details>
</br>


* ClickHouse: таблицы Kafka Engine + MV -> ReplicatedMergeTree

<details>
<summary>Запросы для создания базы данных demo и таблиц для вставки batch и streaming</summary>

```sql
CREATE DATABASE IF NOT EXISTS demo
ON CLUSTER replicated_cluster;

-- ==============
-- RAW (replicated)
-- ==============
CREATE TABLE IF NOT EXISTS demo.binance_aggtrades_raw
ON CLUSTER replicated_cluster

(
    symbol LowCardinality(String),
    event_time DateTime64(3, 'UTC'),
    agg_trade_id UInt64,
    price Float64,
    quantity Float64,
    first_trade_id UInt64,
    last_trade_id UInt64,
    is_buyer_maker UInt8,

    ingest_ts DateTime64(3, 'UTC') DEFAULT now64(3),
    source LowCardinality(String) DEFAULT 'kafka'
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/demo/binance_aggtrades_raw', '{replica}')
PARTITION BY toDate(event_time)
ORDER BY (symbol, event_time, agg_trade_id);

-- ========================
-- Kafka engines (2 topics)
-- ========================
CREATE TABLE IF NOT EXISTS demo.kafka_binance_stream
ON CLUSTER replicated_cluster
(
    symbol String,
    event_time DateTime64(3, 'UTC'),
    agg_trade_id UInt64,
    price Float64,
    quantity Float64,
    first_trade_id UInt64,
    last_trade_id UInt64,
    is_buyer_maker UInt8
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'binance_trades_stream',
    kafka_group_name = 'ch_binance_stream',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 4,
    kafka_max_block_size = 10000,
    kafka_skip_broken_messages = 1;

CREATE TABLE IF NOT EXISTS demo.kafka_binance_batch
ON CLUSTER replicated_cluster
(
    symbol String,
    event_time DateTime64(3, 'UTC'),
    agg_trade_id UInt64,
    price Float64,
    quantity Float64,
    first_trade_id UInt64,
    last_trade_id UInt64,
    is_buyer_maker UInt8
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:9092',
    kafka_topic_list = 'binance_trades_batch',
    kafka_group_name = 'ch_binance_batch',
    kafka_format = 'JSONEachRow',
    kafka_num_consumers = 2,
    kafka_max_block_size = 50000,
    kafka_skip_broken_messages = 1;

-- ==========================
-- MV: Kafka -> Replicated RAW
-- ==========================
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_stream_to_raw
ON CLUSTER replicated_cluster
TO demo.binance_aggtrades_raw
AS
SELECT
    symbol,
    event_time,
    agg_trade_id,
    price,
    quantity,
    first_trade_id,
    last_trade_id,
    is_buyer_maker,
    now64(3) AS ingest_ts,
    'stream' AS source
FROM demo.kafka_binance_stream;

CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_batch_to_raw
ON CLUSTER replicated_cluster
TO demo.binance_aggtrades_raw
AS
SELECT
    symbol,
    event_time,
    agg_trade_id,
    price,
    quantity,
    first_trade_id,
    last_trade_id,
    is_buyer_maker,
    now64(3) AS ingest_ts,
    'batch' AS source
FROM demo.kafka_binance_batch;
```
</details>
</br>

Пояснение структуры:
  - `symbol` — торговая пара
  - `event_time` — время сделки
  - `agg_trade_id` — агрегированный идентификатор сделки
  - `price` — цена
  - `quantity` — объём
  - `first_trade_id`, `last_trade_id` — диапазон идентификаторов
  - `is_buyer_maker` — флаг стороны инициатора


* Быстрый тест: отправка 1 сообщения в stream topic

<details>
<summary>Команды для отправки сообщения и проверки данных в таблице</summary>

```bash
docker exec -i kafka bash -lc 'cat > /tmp/one.json <<JSON
{"symbol":"BTCUSDT","event_time":"2023-07-01 00:00:00.123","agg_trade_id":1,"price":30000.1,"quantity":0.01,"first_trade_id":1,"last_trade_id":1,"is_buyer_maker":0}
JSON
kafka-console-producer --bootstrap-server kafka:9092 --topic binance_trades_stream < /tmp/one.json'

sleep 2
docker exec -it ch1 clickhouse-client --user default --password mypassword -q \
  "SELECT symbol, event_time, agg_trade_id, source FROM demo.binance_aggtrades_raw ORDER BY ingest_ts DESC LIMIT 5;"
```
</details>
</br>

---

#### 1.2 Установка Prometheus - Я СЕЙЧАС НА ЭТОМ ШАГЕ


	•	сбор метрик:
	•	ClickHouse
	•	Kafka
	•	Node Exporter
	•	Docker

#### 1.3 Установка Grafana
	•	базовые dashboards:
	•	CPU
	•	RAM
	•	Disk
	•	ClickHouse
	•	Kafka

#### 1.4 Развертывание MinIO (cold storage) - опционально
	•	S3-compatible storage
	•	бакет для архивов



#### Результат шага

---

### Шаг 2 — Bootstrap загрузка данных через Airflow

🎯 Цель:

Создать source-of-truth датасет (июль 2023).

#### 2.1 Развернуть Airflow
	•	docker-compose
	•	Postgres backend
	•	volume для dags
	•	публичный доступ через Nginx

#### 2.2 DAG: Загрузка Binance July 2023
	•	загрузка через API
	•	запись в staging table
	•	экспорт в Parquet
	•	сохранение в:
	•	локальный volume
	•	MinIO (опционально)

#### 2.3 DAG: Загрузка погоды
	•	загрузка hourly
	•	создание reference table

#### 2.4 DAG: Загрузка электроэнергии
	•	загрузка hourly
	•	создание reference table

#### Результат шага

---

### Шаг 3 — Проектирование схемы ClickHouse

🎯 Цель:

Создать правильную архитектуру таблиц.

#### 3.1 Raw Layer
- replicated raw table
- TTL или manual cleanup

#### 3.2 Kafka Engine Table
- таблица для streaming ingestion

#### 3.3 Materialized Views
- из Kafka → Raw
- агрегации (1m, 15m, 1h, 1d)

#### 3.4 Data Mart
- таблицы с JOIN:
- trades
- weather
- electricity

#### Результат шага

---

### Шаг 4 — Replay Service (ключевой этап)

🎯 Цель:

Создать управляемый генератор нагрузки.

#### 4.1 Контейнер Replay Service
- чтение Parquet
- TIME_SCALE
- INTENSITY_K
- batch + streaming

#### 4.2 Kafka Producer
- acks=all
- idempotence=true

#### 4.3 Batch Engine
- MIN_BATCH_SIZE
- MAX_BATCH_INTERVAL

#### 4.4 State Control
- start / stop / pause
- loop July

#### Результат шага

---

### Шаг  5 — Мониторинг лаборатории

🎯 Цель:

Сделать систему наблюдаемой.

#### 5.1 ClickHouse metrics
- insert rate
- parts
- merges backlog

#### 5.2 Kafka metrics
- lag
- throughput

#### 5.3 Replay metrics
- events/sec
- batch size
- error rate

#### 5.4 Grafana dashboards
- unified ingestion experiment dashboard

#### 5.5 Пользователи `admin` и `demo`

- создание пользователя `demo` с правами просмотра:
  * все таблицы Clickhouse
  * дашборды Grafana
  * интерфейс Airflow
  * ssh доступ к Ubuntu Server с правами просмотра каталога проекта

#### Результат шага

---

### Шаг 6 — Визуализация

🎯 Цель:

Сравнение __batch__ vs __streaming__.

#### 6.1 Развернуть DataLens

(или подключить к существующему)

#### 6.2 Два идентичных dashboard
- batch source
- streaming source

#### 6.3 Синхронизированные фильтры

#### Результат шага
---

### Шаг  7 — Управление жизненным циклом

🎯 Цель:

#### 7.1 TTL или TRUNCATE после цикла

#### 7.2 DAG для очистки

#### 7.3 DAG для запуска нового эксперимента

#### Результат шага
---

### Шаг 8 — Backup & Recovery

🎯 Цель:

#### 8.1 clickhouse-backup

#### 8.2 backup в MinIO

#### 8.3 восстановление кластера

#### Результат шага
---

### Шаг 9 — Документация и репозиторий

🎯 Цель:

#### 9.1 README.md
	•	архитектура
	•	схемы
	•	инструкции запуска

#### 9.2 Скриншоты
	•	Portainer
	•	Grafana
	•	DataLens
	•	нагрузка

#### 9.3 Описание эксперимента
	•	при INTENSITY_K = 1
	•	при INTENSITY_K = 5
	•	при INTENSITY_K = 10
#### Результат шага
---

### Шаг 10 — Презентация (10–15 минут)
1.	Проблема
2.	Архитектура
3.	Инженерное решение
4.	Демонстрация
5.	Результаты
6.	Ограничения
7.	Что можно улучшить
8.  Перспектива проекта, следующие шаги





## 2. Реализация проекта


Проектирование хранилища данных (Data Warehouse)

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

1. Пайплайны обработки данных

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