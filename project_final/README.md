
__Проектная работа по курсу обучения OTUS "Clickhouse для инженеров и архитекторов БД"__

Студент: Богаченков Александр Викторович

# Тема. Аналитика в реальном времени на базе ClickHouse. Сравнительный анализ batch и streaming загрузки данных при неравномерной нагрузке

# 📑 Оглавление

- [1. Обзор проекта](#1-обзор-проекта)
  - [1.1. Цель проекта](#11-цель-проекта)
  - [1.2. Моделируемая производственная проблема](#12-моделируемая-производственная-проблема)
  - [1.3. Архитектурная концепция проекта](#13-архитектурная-концепция-проекта)
  - [1.4. Источник данных (Source of Truth)](#14-источник-данных-source-of-truth)
  - [1.5. Архитектура потоков данных](#15-архитектура-потоков-данных)
  - [1.6. Streaming Pipeline](#16-streaming-pipeline)
  - [1.7. Batch Pipeline](#17-batch-pipeline)
  - [1.8. Архитектура хранения данных](#18-архитектура-хранения-данных)
  - [1.9. Визуализация](#19-визуализация)
  - [1.10. Мониторинг и наблюдаемость системы](#110-мониторинг-и-наблюдаемость-системы)
  - [1.11. Циклический режим работы](#111-циклический-режим-работы)
  - [1.12. Кластер ClickHouse](#112-кластер-clickhouse)
  - [1.13. Инфраструктура и аппаратная конфигурация](#113-инфраструктура-и-аппаратная-конфигурация-deployment-environment)
  - [1.14. Архитектурная ценность проекта](#114-архитектурная-ценность-проекта)

- [2. Реализация проекта](#2-реализация-проекта)
  - [Шаг 0 — Базовая инфраструктура](#шаг-0---базовая-инфраструктура)
  - [Шаг 1 — Инфраструктура данных](#шаг-1--инфраструктура-данных-bootstrap-layer)
  - [Шаг 2 — Загрузка данных через Airflow](#шаг-2--загрузка-данных-через-airflow)
  - [Шаг 3 — Проектирование схемы ClickHouse](#шаг-3--проектирование-схемы-clickhouse)
  - [Шаг 4 — Мониторинг лаборатории](#шаг-4--мониторинг-лаборатории)
  - [Шаг 5 — Визуализация BI](#шаг-5--визуализация-bi)
  - [Шаг 6 — Управление жизненным циклом](#шаг-6--управление-жизненным-циклом)
  - [Шаг 7 — Выводы](#шаг-7---выводы)

- [Ссылки на сервисы](#ссылки-на-сервисы)

- [Скриншоты](#скриншоты)

- [Что далее?](#что-далее)



## 1. Обзор проекта

### 1.1. Цель проекта

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

Для демонстрации нагрузки используется исторический набор событий, который воспроизводится с сохранением исходного временного профиля.

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

Формат хранения: `Parquet`

Файл размещается в: `S3-совместимое хранилище MinIO`

Это обеспечивает:
- воспроизводимость экспериментов
- независимость от Binance API
- возможность повторного запуска эксперимента
- контроль над нагрузкой

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
<summary>Основная схема потоков данных</summary>

```mermaid
flowchart TD

A[Historical Dataset<br>Binance Trades<br>July 2023]
B[Parquet Storage]
C[MinIO S3 Storage]
D[Apache Airflow<br>Pipeline Orchestrator]

%% SOURCE CHAIN
A --> B
B --> C
C --> D

%% PIPELINE SPLIT
D --> E[Streaming Pipeline]
D --> F[Batch Pipeline]

%% STREAMING PIPELINE
E --> G[Apache Kafka]
G --> I[ClickHouse Kafka Engine]
I --> K[Raw Streaming Table<br>ReplicatedMergeTree]
K --> M1[Streaming Aggregation Layer<br>Materialized Views]
M1 --> N1[Streaming Data Mart<br>Aggregated Tables]

%% BATCH PIPELINE
F --> H[Batch Buffer<br>Airflow DAG]
H --> J[INSERT INTO ClickHouse]
J --> L[Raw Batch Table<br>ReplicatedMergeTree]
L --> M2[Batch Aggregation Layer<br>Materialized Views]
M2 --> N2[Batch Data Mart<br>Aggregated Tables]

%% REFERENCE DATA
O[Electricity Consumption<br>Reference Table]

N1 --> O
N2 --> O

%% BI LAYER
P[Yandex DataLens]

O --> P

P --> Q[Streaming Dashboard]
P --> R[Batch Dashboard]
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

`Airflow → Kafka topic → Kafka Engine → Materialized View → INSERT в Raw таблицу`

Особенности streaming ingestion:
- события поступают практически в реальном времени (каждую секунду)
- вставки выполняются небольшими блоками (максимальный размер блока 5000 строк)
- создаётся большое количество data parts (части)

Это позволяет наблюдать:
- рост числа `parts`
- нагрузку на `merge`-процессы
- влияние частых вставок на производительность системы.

---

### 1.7. Batch Pipeline

__Batch__ ingestion моделирует классический ETL-подход.

Airflow формирует буфер данных и выполняет вставку в ClickHouse при выполнении одного из условий:
- накоплено 50 000 строк
- прошло 10 минут с момента предыдущей вставки

Таким образом:
- минимальный размер вставки — 50 000 строк
- вставки выполняются не чаще 6 раз в час

Поток данных:

`Airflow (Batch Buffer) → INSERT INTO ClickHouse → Raw Batch Table`

Это приводит к:
- меньшему количеству частей данных
- более стабильной работе `merge`-процессов
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
- Active Parts 
- Merge backlog 
- Query Latency 

Анализируется:

- количество INSERT-запросов в секунду на каждой ноде
- рост количества частей
- загрузка merge-процессов
- задержки вставок

#### 1.10.2. Мониторинг Kafka

Основные метрики:
- Consumer Lag
- Throughput

Позволяет выявлять:
- накопление lag
- скорость вставки данных

#### 1.10.3. Системные метрики

Собираются через Node Exporter.

Метрики:
- CPU usage
- RAM usage
- Disk write rate
- Disk space usage

Позволяют оценивать влияние различных способов загрузки данных на ресурсы сервера.

#### 1.10.4. Мониторинг Apache Airflow

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



---

### 1.11. Циклический режим работы

После завершения воспроизведения исторического датасета:
1.	выполняется очистка таблиц
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


✅ Представленная архитектура проекта:
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

[⬆️ Наверх](#-оглавление)

## 2. Реализация проекта

### Шаг 0 - Базовая инфраструктура 

🎯 Цель:

Подготовить сервер, сеть и ClickHouse кластер.

Что входит:
#### Установка OS Ubuntu Server 22.04 LTS 
  Операционная система развернута из стандартного дистрибутива с сайта (https://releases.ubuntu.com/22.04/). Имя сервера - `ch-lab`
#### Установка Docker и Docker Compose на `ch-lab`

<details>
<summary>Команды для установки</summary>

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

# Проверка:
docker --version
docker compose version
```

</details>
</br>

---

#### Развертывание кластера ClickHouse cluster (1 shard, 2 replicas) и ClickHouse Keeper (3 nodes)
На сервере `ch-lab`:

🧱 1. Создание структуры каталогов

<details>
<summary>Команды для создания директорий</summary>

```bash
mkdir -p ~/infra/ch-cluster
cd ~/infra/ch-cluster

mkdir -p ch1/config.d
mkdir -p ch2/config.d
mkdir -p ch1/users.d
mkdir -p ch2/users.d

mkdir -p keeper1
mkdir -p keeper2
mkdir -p keeper3

mkdir -p /data/clickhouse/ch1
mkdir -p /data/clickhouse/ch2

mkdir -p /data/keeper/keeper1
mkdir -p /data/keeper/keeper2
mkdir -p /data/keeper/keeper3
```

</details>
</br>

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

🔑 8. Файл `default.xml` - создание `default` пользователя (одинаковый на нодах ch1 и ch2)

<details>
<summary>infra/ch-cluster/ch1/users.d/default.xml</summary>

```xml
<clickhouse>
    <users>
        <default>
            <access_management>1</access_management> <!-- дает право на создание ролей -->
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
      - ./ch1/users.d:/etc/clickhouse-server/users.d
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
      - ./ch2/users.d:/etc/clickhouse-server/users.d
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
Делаем, если удобнее визуальный интерфейс мониторинга контейнеров

<details>
<summary>Команды для Portainer</summary>

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

</details>
</br>

---

#### Структура каталогов кластера:
```txt
.
├── ch1
│   ├── config.d
│   │   ├── listen.xml
│   │   ├── macros.xml
│   │   ├── memory_limits.xml
│   │   ├── prometheus.xml
│   │   ├── remote_servers.xml
│   │   └── zookeeper.xml
│   └── users.d
│       └── default.xml
├── ch2
│   ├── config.d
│   │   ├── listen.xml
│   │   ├── macros.xml
│   │   ├── memory_limits.xml
│   │   ├── prometheus.xml
│   │   ├── remote_servers.xml
│   │   └── zookeeper.xml
│   └── users.d
│       └── default.xml
├── docker-compose.yml
├── .env
├── keeper1
│   └── config.xml
├── keeper2
│   └── config.xml
└── keeper3
    └── config.xml

9 directories, 19 files
```


#### Результат шага
 - развернут основной сервер проекта на OS Ubuntu Server 22.04 LTS
 - на сервере создана основная часть архитектуры - кластер базы данных Clickhouse (2 ноды) в контейнерах Docker в единой локальной сети `infra-net` с управлением через Clickhouse-Keeper (3 ноды)

[⬆️ Наверх](#-оглавление)

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
  
Создание топика для stream

<details>
<summary>Команда для создания</summary>

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

---


__Сбор метрик:__
- ClickHouse
- Kafka
- Apache Airflow
- Node Exporter (системные метрики)

#### 1.2 Установка Prometheus и Grafana

Для мониторинга состояния системы используется стек:
- Prometheus — сбор метрик
- Grafana — визуализация метрик

Создаю в папке `infra` каталоги для хранения конфигурации и данных мониторинга.

<details>
<summary>Команды для создания директорий</summary>

```bash
mkdir -p monitoring/prometheus
mkdir -p monitoring/prometheus/data

mkdir -p monitoring/grafana
mkdir -p monitoring/grafana/data

mkdir -p monitoring/node-exporter

mkdir -p monitoring/logs

# Назначаю права доступа.

sudo chown -R 65534:65534 monitoring/prometheus/data
sudo chown -R 472:472 monitoring/grafana/data
sudo chmod -R 775 monitoring/logs
```
</details>
</br>



---

__Prometheus__

Prometheus разворачивается в контейнере Docker и подключается к общей сети проекта `infra-net`

<details>
<summary>Файл: monitoring/prometheus/docker-compose.yml</summary>

```yml
version: '3.8'

services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped

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

</details>
</br>

---

__Конфигурация Prometheus__

Prometheus опрашивает метрики различных компонентов системы с интервалом 5 секунд.

<details>
<summary>Файл конфигурации: monitoring/prometheus/prometheus.yml</summary>

```yml
global:
  scrape_interval: 5s

scrape_configs:

  - job_name: clickhouse
    static_configs:
      - targets:
          - ch1:9363
          - ch2:9363

  - job_name: node
    static_configs:
      - targets:
          - node_exporter:9100
```

</details>
</br>

---

__Включение Prometheus метрик в ClickHouse__

ClickHouse поддерживает встроенный экспорт метрик Prometheus.

Для этого добавляю конфигурацию Prometheus в настройки сервера.

<details>
<summary>infra/ch-cluster/ch1/config.d/prometheus.xml</summary>

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

</details>
</br>


Аналогичный файл добавляется для второй ноды.

<details>
<summary>infra/ch-cluster/ch2/config.d/prometheus.xml</summary>

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

</details>
</br>


После добавления конфигурации перезапускаю контейнеры ClickHouse
```bash
docker restart ch1 ch2
```

Проверка метрик ClickHouse - открыть интерфейс Prometheus:

http://SERVER_IP:9090

Перейти:

Graph → Execute

Примеры запросов:

- Количество запросов в секунду

rate(ClickHouseProfileEvents_Query[1m])


- Количество активных частей

ClickHouseMetrics_PartsActive


---

__Установка Node Exporter__

Node Exporter используется для сбора системных метрик сервера:
- CPU
- RAM
- Disk
- IO
- Network

Контейнер Node Exporter подключается к той же сети `infra-net`.

<details>
<summary>Файл: monitoring/node-exporter/docker-compose.yml</summary>

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

</details>
</br>


Проверка системных метрик. Примеры запросов Prometheus:

- CPU загрузка

rate(node_cpu_seconds_total{mode!="idle"}[1m])

- Использование памяти

node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

---

__Настройка Grafana__

Grafana используется для визуализации метрик системы.

<details>
<summary>Файл: monitoring/grafana/docker-compose.yml</summary>

```yml
version: '3.8'

services:

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped

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

</details>
</br>


Проверка доступа. Grafana доступна по адресу:

http://SERVER_IP:3000

---

__Подключение Prometheus в Grafana__

В интерфейсе Grafana:

1️⃣ Connections → Data sources
2️⃣ Add data source
3️⃣ Выбрать Prometheus

URL: http://prometheus:9090

---

__Результат__

После выполнения данных шагов система мониторинга позволяет отслеживать:
- загрузку CPU и памяти сервера
- состояние ClickHouse кластера
- количество частей данных
- нагрузку на merge-процессы
- скорость выполнения запросов
- влияние streaming и batch ingestion на ресурсы системы

Полученные метрики используются для анализа поведения системы в рамках эксперимента.

#### 1.3 Развертывание MinIO (cold storage)

Для хранения исходного исторического датасета используется объектное хранилище MinIO, совместимое с протоколом Amazon S3.

MinIO используется как уровень хранения COLD, где размещается исходный датасет в формате Parquet.

__Создание каталогов для MinIO__

```bash
mkdir -p ~/infra/minio/data
```

__Запуск MinIO__ 

MinIO разворачивается в контейнере Docker и подключается к общей сети проекта `infra-net`

<details>
<summary>Файл: infra/minio/docker-compose.yml</summary>

```yml
version: "3.8"

services:

  minio:
    image: quay.io/minio/minio:latest
    container_name: minio
    restart: unless-stopped

    networks:
      - infra-net

    ports:
      - "9003:9000"
      - "9102:9201"

    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: AdminLab_2025!

    volumes:
      - ./data:/data

    command: server /data --console-address ":9201"

networks:
  infra-net:
    external: true
```

</details>
</br>



Веб-интерфейс MinIO доступен по адресу: http://SERVER_IP:9102



__Создание bucket для хранения датасета__

После входа в консоль MinIO необходимо создать bucket для хранения исходных данных.

Имя bucket: `clickhouse-lab-datasets`

Формат хранения: Parquet

Пример пути к файлу:

`s3://clickhouse-lab-datasets/binance/july_2023/trades.parquet`

__Результат шага:__

- развернуто объектное хранилище MinIO
- создан bucket для хранения датасета
- обеспечено централизованное хранение исторических данных
- подготовлена инфраструктура для воспроизводимого запуска эксперимента

[⬆️ Наверх](#-оглавление)

---

### Шаг 2 — Загрузка данных через Airflow

🎯 Цель:

Создать source-of-truth датасет (июль 2023).

#### 2.1 Развернуть Apache Airflow

Apache Airflow устанавливается в контейнерах в сети `infra-net`

__Подготовка директории и настройка прав__

<details>
<summary>Файл: infra/minio/docker-compose.yml</summary>

```bash
mkdir -p ~/infra/airflow
cd ~/infra/airflow

mkdir -p dags data logs plugins

sudo chown -R 50000:0 dags data logs plugins
sudo chmod -R 775 dags logs plugins
```

</details>
</br>


__Создание docker-compose.yaml__

<details>
<summary>Файл: infra/airflow/docker-compose.yml</summary>

```yml
# docker-compose.yaml
# Airflow 3.1.7 (api-server) + LocalExecutor + Postgres
# network: infra-net (external)

services:
  postgres:
    image: postgres:16
    container_name: airflow-postgres
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s
    restart: unless-stopped
    networks:
      - infra-net

  airflow-init:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: airflow-init
    env_file:
      - .env
    user: "0:0"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - infra-net
    entrypoint: /bin/bash
    command:
      - -c
      - |
        set -e
        echo "Creating directories..."
        mkdir -p /opt/airflow/{dags,logs,plugins}
        echo "Fix ownership to ${AIRFLOW_UID}:0 ..."
        chown -R "${AIRFLOW_UID}:0" /opt/airflow/{dags,logs,plugins}

        echo "Airflow version:"
        /entrypoint airflow version

        echo "DB migrate..."
        /entrypoint airflow db migrate

        echo "Create admin user (idempotent)..."
        /entrypoint airflow users create \
          --username "${_AIRFLOW_WWW_USER_USERNAME}" \
          --password "${_AIRFLOW_WWW_USER_PASSWORD}" \
          --firstname "${_AIRFLOW_WWW_USER_FIRSTNAME}" \
          --lastname "${_AIRFLOW_WWW_USER_LASTNAME}" \
          --role Admin \
          --email "${_AIRFLOW_WWW_USER_EMAIL}" \
        || true

        echo "Init done."

    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/data:/opt/airflow/data
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
      

  airflow-apiserver:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: airflow-apiserver
    hostname: airflow-apiserver
    env_file:
      - .env
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow

      # для чтения логов через log server scheduler-а
      AIRFLOW__LOGGING__WORKER_LOG_SERVER_HOST: airflow-scheduler
      AIRFLOW__LOGGING__WORKER_LOG_SERVER_PORT: 8793
      AIRFLOW__CORE__LOAD_EXAMPLES: "false"
      AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: "true"

    ports:
      - "8080:8080"
    command: api-server
    restart: unless-stopped
    networks:
      - infra-net
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/data:/opt/airflow/data
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins

  airflow-scheduler:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: airflow-scheduler
    hostname: airflow-scheduler
    env_file:
      - .env
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow

      # КРИТИЧНО: base URL с /execution/
      AIRFLOW__CORE__EXECUTION_API_SERVER_URL: http://airflow-apiserver:8080/execution/

      # для UI ссылок/внутренних URL — оставляем так (не влияет на открытие UI)
      AIRFLOW__API__BASE_URL: http://airflow-apiserver:8080

      # log server scheduler-а
      AIRFLOW__LOGGING__WORKER_LOG_SERVER_HOST: airflow-scheduler
      AIRFLOW__LOGGING__WORKER_LOG_SERVER_PORT: 8793

    command: scheduler
    restart: unless-stopped
    networks:
      - infra-net
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/data:/opt/airflow/data
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins

  airflow-dag-processor:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: airflow-dag-processor
    hostname: airflow-dag-processor
    env_file:
      - .env
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow

    command: dag-processor
    restart: unless-stopped
    networks:
      - infra-net
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/data:/opt/airflow/data
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins

  airflow-triggerer:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: airflow-triggerer
    hostname: airflow-triggerer
    env_file:
      - .env
    depends_on:
      airflow-init:
        condition: service_completed_successfully
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__CORE__AUTH_MANAGER: airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow

    command: triggerer
    restart: unless-stopped
    networks:
      - infra-net
    volumes:
      - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
      - ${AIRFLOW_PROJ_DIR:-.}/data:/opt/airflow/data
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins

volumes:
  postgres-db-volume:

networks:
  infra-net:
    external: true
```

</details>
</br>

__Создание окружения__
Дополнительно в каталоге airflow где будет запускаться контейнер создаю файл окружения .env:

<details>
<summary>Файл: infra/airflow/.env</summary>

```text
AIRFLOW_IMAGE_NAME=apache/airflow:3.1.7-python3.10
AIRFLOW_UID=50000
AIRFLOW_PROJ_DIR=.
AIRFLOW__API__AUTH_BACKEND=airflow.api.auth.backend.session

AIRFLOW__CORE__FERNET_KEY=_EEHzk5SgyDJXHrmphwavn4rDwNQigD8Oq0J9zwWxSY=
AIRFLOW__WEBSERVER__SECRET_KEY=9fH2kLmQ8xV4zRtY7uNp3wSa6BcDeFgHiJkLmNoPqRsTuVwXyZa1234567890AB
AIRFLOW__API__SECRET_KEY=410c44973d9cf1ff2cc337511fd6fae26aed1939bbe956f756f457df8d27d05d
AIRFLOW__API_AUTH__JWT_SECRET=19215a8c77489e3d34d2b480caacac7d5a5238d4e416ada843862b8b9b8921a9

# Инициализация БД
_AIRFLOW_DB_MIGRATE=true

# Создание администратора при первом запуске
_AIRFLOW_WWW_USER_CREATE=true
_AIRFLOW_WWW_USER_USERNAME=admin
_AIRFLOW_WWW_USER_PASSWORD=AdminLab_2025!
_AIRFLOW_WWW_USER_FIRSTNAME=Admin
_AIRFLOW_WWW_USER_LASTNAME=User
_AIRFLOW_WWW_USER_EMAIL=admin@example.com
```

</details>
</br>


__Установка зависимостей Python__

Для получения возможности установки дополнительных библиотек (которых возможно нет в стандартном контейнере Airflow)

Создаю Dockerfile:

<details>
<summary>Файл: infra/airflow/Dockerfile</summary>


```bash
FROM apache/airflow:3.1.7-python3.10

USER airflow

RUN pip install --no-cache-dir \
    clickhouse-connect==0.7.8 \
    kafka-python==2.0.2 \
    minio==7.2.7 \
    requests==2.31.0 \
    pyarrow==15.0.2

```
</details>
</br>


<details>
<summary>Изменяю image (если не сделал этого ранее) в compose на build для всех сервисов Airflow:</summary>

```yaml
build:
  context: .
  dockerfile: Dockerfile
```

</details>
</br>

<details>
<summary>Команды пересборки и запуска контейнеров</summary>

```bash
docker compose down
docker compose build
docker compose up -d

# Полные пересборка и перезапуск при необходимости :

docker compose down -v
docker volume prune -f
docker compose up airflow-init
docker compose up -d --build
```

</details>
</br>


__Запуск Airflow__

Инициализация БД: `docker compose up airflow-init --build`

<details>
<summary>Команда создания пользователя admin</summary>


```bash
docker compose exec airflow-apiserver airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email admin@example.com \
  --password AdminLab_2025!
```

</details>
</br>

Запуск сервисов: `docker compose up -d --build`

__Вход в Web UI__

http://<IP_сервера>:8080

__Результат__
Развернуты и готовы к работе Apache Airflow 3.1.7:
- airflow apiserver 
- airflow dag processor
- airflow scheduler
- airflow trigger

#### 2.2 DAG: Загрузка Binance July 2023

<details>
<summary>DAG: infra/airflow/dags/binance_bootstrap_dataset.py</summary>


```python
from airflow import DAG
from airflow.providers.standard.operators.python import PythonOperator
from datetime import datetime
import zipfile
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import os
from minio import Minio


# -----------------------------------------
# CONFIG
# -----------------------------------------

DATA_DIR = "/opt/airflow/data/binance"

FILES = [
    "BTCUSDT-aggTrades-2023-07.zip",
    "ETHUSDT-aggTrades-2023-07.zip",
    "BTCUSDC-aggTrades-2023-07.zip",
    "ETHUSDC-aggTrades-2023-07.zip",
]

LOCAL_STREAM_PARQUET = "/opt/airflow/data/binance/trades_stream.parquet"
LOCAL_BATCH_PARQUET = "/opt/airflow/data/binance/trades_batch.parquet"

MINIO_BUCKET = "clickhouse-lab-datasets"

MINIO_STREAM_OBJECT = "binance/july_2023/trades_stream.parquet"
MINIO_BATCH_OBJECT = "binance/july_2023/trades_batch.parquet"


minio_client = Minio(
    "minio:9000",
    access_key="admin",
    secret_key="AdminLab_2025!",
    secure=False
)


# -----------------------------------------
# TASK
# -----------------------------------------

def build_dataset():

    dfs = []

    for filename in FILES:

        symbol = filename.split("-")[0]

        zip_path = os.path.join(DATA_DIR, filename)

        print(f"Processing {zip_path}")

        with zipfile.ZipFile(zip_path) as z:

            csv_name = z.namelist()[0]

            with z.open(csv_name) as f:

                df = pd.read_csv(
                    f,
                    header=None,
                    names=[
                        "agg_trade_id",
                        "price",
                        "quantity",
                        "first_trade_id",
                        "last_trade_id",
                        "event_time",
                        "is_buyer_maker",
                        "is_best_match"
                    ]
                )

        # -----------------------------------------
        # Удаляю ненужную колонку Binance
        # -----------------------------------------

        df = df.drop(columns=["is_best_match"])

        # -----------------------------------------
        # Нормализация timestamp
        # -----------------------------------------

        df["event_time"] = pd.to_numeric(df["event_time"], errors="coerce")

        df = df.dropna(subset=["event_time"])

        df["event_time"] = pd.to_datetime(
            df["event_time"].astype("int64"),
            unit="ms",
            utc=True
        )

        # -----------------------------------------
        # Добавляю symbol
        # -----------------------------------------

        df["symbol"] = symbol

        dfs.append(df)

    # -----------------------------------------
    # Объединяю все пары
    # -----------------------------------------

    full_df = pd.concat(dfs, ignore_index=True)

    print("Dataset rows:", len(full_df))

    # -----------------------------------------
    # Сортировка по времени (важно для replay)
    # -----------------------------------------

    full_df = full_df.sort_values("event_time")

    # -----------------------------------------
    # Оптимизация типов
    # -----------------------------------------

    full_df["is_buyer_maker"] = full_df["is_buyer_maker"].astype("int8")

    # -----------------------------------------
    # Конвертация в Parquet
    # -----------------------------------------

    print("Converting to Parquet")

    table = pa.Table.from_pandas(full_df, preserve_index=False)

    pq.write_table(table, LOCAL_STREAM_PARQUET)
    pq.write_table(table, LOCAL_BATCH_PARQUET)

    # -----------------------------------------
    # Upload STREAM dataset
    # -----------------------------------------

    print("Uploading STREAM dataset to MinIO")

    minio_client.fput_object(
        MINIO_BUCKET,
        MINIO_STREAM_OBJECT,
        LOCAL_STREAM_PARQUET
    )

    # -----------------------------------------
    # Upload BATCH dataset
    # -----------------------------------------

    print("Uploading BATCH dataset to MinIO")

    minio_client.fput_object(
        MINIO_BUCKET,
        MINIO_BATCH_OBJECT,
        LOCAL_BATCH_PARQUET
    )

    print("Datasets uploaded successfully")


# -----------------------------------------
# DAG
# -----------------------------------------

default_args = {
    "owner": "airflow",
    "retries": 1
}

with DAG(

    dag_id="binance_build_parquet_dataset",

    start_date=datetime(2025, 1, 1),

    schedule=None,

    catchup=False,

    default_args=default_args,

    tags=["dataset", "binance"]

) as dag:

    build = PythonOperator(
        task_id="build_parquet_dataset",
        python_callable=build_dataset
    )
```

</details>
</br>

__Результат__

Датасет о сделках с основными валютами на бирже Binance загружен в S3-совместимое хранилище и готов к использованию в качестве источника данных. Для удобства построения параллельных потоков созданы два одинаковых датасета с разными названиями `trades_stream.parquet` и `trades_batch.parquet`

---

#### 2.3 DAG: Загрузка данных потребления электроэнергии в США в июле 2023 года

Создаю таблицу в Clickhouse для вставки данных
<details>
<summary>Таблица electricity</summary>


```sql

CREATE TABLE IF NOT EXISTS demo.electricity_hourly
ON CLUSTER replicated_cluster
(
    timestamp DateTime('UTC'),
    region String,
    demand_mw Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/electricity_hourly',
    '{replica}'
)
PARTITION BY toDate(timestamp)
ORDER BY (region, timestamp);

```

</details>
</br>

---
С помощью DAG в Apache Airflow загружаю данные в таблицу в Clickhouse:

<details>
<summary>Файл:/infra/airflow/dags/load_electricity_dataset.py</summary>



```python
from datetime import datetime  
import pandas as pd  
import clickhouse_connect  

from airflow import DAG  
from airflow.providers.standard.operators.python import PythonOperator  
import os

# в .env предварительно задать CLICKHOUSE_PASSWORD= 
DATA_FILE = "/opt/airflow/data/electricity.csv"
PASSWORD = os.getenv("CLICKHOUSE_PASSWORD")

#=============================================================#
#   вариант подключения с использованием connections в Airflow
from airflow.hooks.base import BaseHook

conn = BaseHook.get_connection("clickhouse_demo")

host = conn.host
port = conn.port
user = conn.login
password = conn.password
database = conn.schema
#=============================================================#


def load_electricity():

    print("Reading EIA electricity dataset")

    df = pd.read_csv(DATA_FILE)

    # перевод времени
    df["timestamp"] = pd.to_datetime(df["UTC Time at End of Hour"])

    # фильтр July 2023
    df = df[
        (df["timestamp"] >= "2023-07-01") &
        (df["timestamp"] < "2023-08-01")
    ]

    # нужные колонки
    df = df[["timestamp", "Region", "Demand (MW)"]]

    df = df.rename(columns={
        "Region": "region",
        "Demand (MW)": "demand_mw"
    })

    df = df.dropna()

    rows = df.values.tolist()

    print("Rows to insert:", len(rows))
#    вариант подключения с использованием пароля, заданного в окружении
#    client = clickhouse_connect.get_client(
#        host="ch1",
#        port=8123,
#        username="default",
#        password=PASSWORD,
#        database="demo"
#    )

#   вариант подключения с использованием connections в Apache Airflow
    client = clickhouse_connect.get_client(
        host=host,
        port=port,
        username=user,
        password=password,
        database=database
    )

    print("Cleaning table")

    client.command("""
    TRUNCATE TABLE demo.electricity_hourly
    ON CLUSTER replicated_cluster
    """)

    print("Inserting data")

    client.insert(
        "electricity_hourly",
        rows,
        column_names=["timestamp", "region", "demand_mw"]
    )

    print("Inserted rows:", len(rows))


with DAG(
    dag_id="load_electricity_dataset",
    start_date=datetime(2024,1,1),
    schedule=None,
    catchup=False,
) as dag:

    load_task = PythonOperator(
        task_id="load_electricity_data",
        python_callable=load_electricity
    )
```


</details>
</br>

__Результат__

Датасет потребления электроэнергии загружен в Clickhouse и готов к использованию в качестве таблицы для JOIN.

---

#### 2.4. Пользователи и роли ClickHouse

Для ограничения доступа в системе создаётся отдельная роль только для чтения данных.

Такой пользователь используется BI-инструментами и аналитическими приложениями, которым требуется доступ к данным без возможности их изменения.

Подключение к серверу ClickHouse: ```docker exec -it ch1 clickhouse-client```

Создание роли reader. Роль позволяет выполнять только операции чтения.
```sql
CREATE ROLE IF NOT EXISTS reader
ON CLUSTER replicated_cluster;

GRANT SELECT ON *.* TO reader;
```
Создание пользователя demo, который будет использоваться для подключения к системе аналитики.
```sql
CREATE USER IF NOT EXISTS demo
ON CLUSTER replicated_cluster
IDENTIFIED WITH sha256_password BY 'DemoLab_2025!';
```
Назначение роли пользователю
```sql
GRANT reader TO demo
ON CLUSTER replicated_cluster;

ALTER USER demo
ON CLUSTER replicated_cluster
DEFAULT ROLE reader;
```
Проверка прав пользователя
```sql
SHOW GRANTS FOR demo;
```
Пользователь demo может выполнять только операции чтения. Попытка изменить данные завершится ошибкой:
```sql
DROP TABLE demo.electricity_hourly;

Not enough privileges
```
Таким образом обеспечивается безопасный доступ к данным без риска их изменения.


#### Результат шага:
На данном этапе подготовлены исходные для работы датасеты. Создан пользователь для просмотра данных и аналитики

[⬆️ Наверх](#-оглавление)

---

### Шаг 3 — Проектирование схемы ClickHouse

🎯 Цель:

Создать правильную архитектуру таблиц.

#### 3.1. ClickHouse: таблицы Streaming RAW, Batch RAW, MV

<details>
<summary>Запросы для создания базы данных demo и таблиц для вставки batch и streaming</summary>

```sql
CREATE DATABASE IF NOT EXISTS demo
ON CLUSTER replicated_cluster;

-- ==============================================
-- Streaming RAW (replicated) ReplicatedMergeTree
-- ==============================================

CREATE TABLE IF NOT EXISTS demo.binance_aggtrades_stream
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

    ingest_ts DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/demo/binance_aggtrades_stream', '{replica}')
PARTITION BY toDate(event_time)
ORDER BY (symbol, event_time, agg_trade_id);

-- ===================================
-- Streaming Kafka Engine (replicated)
-- ===================================

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
    kafka_max_block_size = 5000,
    kafka_skip_broken_messages = 1;

-- =====================================
-- Streaming MV: Kafka -> Replicated RAW
-- =====================================

CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_stream_to_raw
ON CLUSTER replicated_cluster
TO demo.binance_aggtrades_stream
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
    now64(3) AS ingest_ts
FROM demo.kafka_binance_stream;

-- =================================================
-- Batch RAW (replicated) ReplicatedMergeTree engine
-- =================================================

CREATE TABLE IF NOT EXISTS demo.binance_aggtrades_batch
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

    ingest_ts DateTime64(3, 'UTC') DEFAULT now64(3)
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/demo/binance_aggtrades_batch', '{replica}')
PARTITION BY toDate(event_time)
ORDER BY (symbol, event_time, agg_trade_id);
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
docker exec -it kafka bash -lc 'cat > /tmp/one.json <<JSON
{"symbol":"BTCUSDT","event_time":"2023-07-01 00:00:00.123","agg_trade_id":1,"price":30000.1,"quantity":0.01,"first_trade_id":1,"last_trade_id":1,"is_buyer_maker":0}
JSON
kafka-console-producer --bootstrap-server kafka:9092 --topic binance_trades_stream < /tmp/one.json'

sleep 2
docker exec -it ch1 clickhouse-client --user default --password mypassword -q \
  "SELECT symbol, event_time, agg_trade_id, source FROM demo.binance_trades_stream ORDER BY ingest_ts DESC LIMIT 5;"
```
</details>
</br>


#### 3.2 DAG: Параллельные pipelines из источника в RAW таблицы Streaming и Batch

Создаю файл DAG, который реализует два параллельных потока данных их источника (хранилище Minio)в целевые RAW таблицы в Clickhouse согласно архитектруре проекта batch и streaming загрузка


<details>
<summary>Файл: ~/infra/airflow/dags/binance_replay_pipelines.py</summary>

```python
from airflow import DAG
from airflow.providers.standard.operators.python import PythonOperator
from airflow.hooks.base import BaseHook

from datetime import datetime
import pandas as pd
import time
import logging
import threading
from queue import Queue

from minio import Minio
from kafka import KafkaProducer
import clickhouse_connect


# =========================
# CONFIG
# =========================

DATA_DIR = "/opt/airflow/data"

STREAM_FILE = f"{DATA_DIR}/trades_stream.parquet"
BATCH_FILE = f"{DATA_DIR}/trades_batch.parquet"

KAFKA_TOPIC = "binance_trades_stream"

STREAM_BLOCK = 5000
BATCH_BLOCK = 50000
BATCH_TIMEOUT = 600  # 10 минут

START_TIME = pd.Timestamp("2023-07-01 00:00:00", tz="UTC")
TOTAL_SECONDS = 30 * 24 * 3600


# =========================
# LOGGING
# =========================

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


# =========================
# CLICKHOUSE
# =========================

def get_ch_client():
    conn = BaseHook.get_connection("clickhouse_demo")

    return clickhouse_connect.get_client(
        host=conn.host,
        port=conn.port,
        username=conn.login,
        password=conn.password,
        database=conn.schema
    )


# =========================
# MINIO
# =========================

minio_client = Minio(
    "minio:9000",
    access_key="admin",
    secret_key="AdminLab_2025!",
    secure=False
)


# =========================
# KAFKA
# =========================

import json

producer = KafkaProducer(
    bootstrap_servers="kafka:9092",
    value_serializer=lambda v: json.dumps(v).encode("utf-8")
)


# =========================
# DOWNLOAD
# =========================

def download():

    logger.info("Downloading datasets")

    minio_client.fget_object(
        "clickhouse-lab-datasets",
        "binance/july_2023/trades_stream.parquet",
        STREAM_FILE
    )

    minio_client.fget_object(
        "clickhouse-lab-datasets",
        "binance/july_2023/trades_batch.parquet",
        BATCH_FILE
    )

    logger.info("Download completed")


# =========================
# STREAM TASK
# =========================

def stream_task():

    logger.info("Stream started")

    df = pd.read_parquet(STREAM_FILE)
    df["event_time"] = pd.to_datetime(df["event_time"], utc=True)
    df = df.sort_values("event_time")

    next_tick = time.time()

    for i in range(TOTAL_SECONDS):

        current_second = START_TIME + pd.Timedelta(seconds=i)
        next_second = current_second + pd.Timedelta(seconds=1)

        df_sec = df[
            (df["event_time"] >= current_second) &
            (df["event_time"] < next_second)
        ]

        if not df_sec.empty:

            rows = df_sec.to_dict("records")

            for j in range(0, len(rows), STREAM_BLOCK):
                chunk = rows[j:j + STREAM_BLOCK]

                for r in chunk:
                    producer.send(KAFKA_TOPIC, {
                        "symbol": r["symbol"],
                        "event_time": r["event_time"].strftime("%Y-%m-%d %H:%M:%S.%f")[:-3],
                      # "event_time": str(r["event_time"]),
                        "agg_trade_id": int(r["agg_trade_id"]),
                        "price": float(r["price"]),
                        "quantity": float(r["quantity"]),
                        "first_trade_id": int(r["first_trade_id"]),
                        "last_trade_id": int(r["last_trade_id"]),
                        "is_buyer_maker": int(r["is_buyer_maker"])
                    })

                producer.flush()

            logger.info(f"[STREAM] {current_second} rows={len(rows)}")

        # синхронизация по времени
        next_tick += 1
        sleep_for = next_tick - time.time()

        if sleep_for > 0:
            time.sleep(sleep_for)
        else:
            logger.warning(f"[STREAM LAG] {-sleep_for:.2f} sec")

    logger.info("Stream finished")


# =========================
# BATCH TASK
# =========================

def batch_task():

    logger.info("Batch started")

    df = pd.read_parquet(BATCH_FILE)
    df["event_time"] = pd.to_datetime(df["event_time"], utc=True)
    df = df.sort_values("event_time")

    client = get_ch_client()

    buffer = []
    flush_queue = Queue()

    last_flush_time = time.time()

    # =========================
    # BACKGROUND INSERTER
    # =========================

    def inserter():

        while True:
            batch = flush_queue.get()

            if batch is None:
                break

            logger.info(f"[INSERT] rows={len(batch)}")

            client.insert(
                "binance_aggtrades_batch",
                batch,
                column_names=[
                    "symbol",
                    "event_time",
                    "agg_trade_id",
                    "price",
                    "quantity",
                    "first_trade_id",
                    "last_trade_id",
                    "is_buyer_maker"
                ]
            )

            flush_queue.task_done()

    thread = threading.Thread(target=inserter, daemon=True)
    thread.start()

    # =========================
    # MAIN LOOP
    # =========================

    next_tick = time.time()

    for i in range(TOTAL_SECONDS):

        current_second = START_TIME + pd.Timedelta(seconds=i)
        next_second = current_second + pd.Timedelta(seconds=1)

        df_sec = df[
            (df["event_time"] >= current_second) &
            (df["event_time"] < next_second)
        ]

        if not df_sec.empty:

            for _, r in df_sec.iterrows():
                buffer.append([
                    r["symbol"],
                    r["event_time"],
                    int(r["agg_trade_id"]),
                    float(r["price"]),
                    float(r["quantity"]),
                    int(r["first_trade_id"]),
                    int(r["last_trade_id"]),
                    int(r["is_buyer_maker"])
                ])

        now = time.time()

        # flush по размеру
        if len(buffer) >= BATCH_BLOCK:
            flush_queue.put(buffer.copy())
            buffer.clear()
            last_flush_time = now

        # flush по времени
        elif (now - last_flush_time) >= BATCH_TIMEOUT and buffer:
            flush_queue.put(buffer.copy())
            buffer.clear()
            last_flush_time = now

        # синхронизация по времени
        next_tick += 1
        sleep_for = next_tick - time.time()

        if sleep_for > 0:
            time.sleep(sleep_for)
        else:
            logger.warning(f"[BATCH LAG] {-sleep_for:.2f} sec")

    # финальный flush
    if buffer:
        flush_queue.put(buffer.copy())

    flush_queue.put(None)
    thread.join()

    logger.info("Batch finished")


# =========================
# DAG
# =========================

with DAG(
    dag_id="binance_replay_time_driven",
    start_date=datetime(2025, 1, 1),
    schedule=None,
    catchup=False,
    tags=["binance", "time-driven"]
) as dag:

    download_op = PythonOperator(
        task_id="download",
        python_callable=download
    )

    stream_op = PythonOperator(
        task_id="stream",
        python_callable=stream_task
    )

    batch_op = PythonOperator(
        task_id="batch",
        python_callable=batch_task
    )

    download_op >> [stream_op, batch_op]
```

</details>
</br>

Логика работы DAG: ОПИСАТЬ

<details>
<summary>Прверочный запрос о количестве строк в RAW таблицах</summary>


```sql
SELECT
    countIf(source = 'stream') AS stream_rows,
    countIf(source = 'batch') AS batch_rows,
    batch_rows - stream_rows AS diff
FROM
(
    SELECT 'stream' AS source FROM demo.binance_aggtrades_stream
    UNION ALL
    SELECT 'batch' AS source FROM demo.binance_aggtrades_batch
);
```
</details>
</br>

<details>
<summary>Прверочный запрос о количестве партов в таблицах</summary>


```sql
SELECT
    table,
    count() parts
FROM system.parts
WHERE database='demo'
GROUP BY table;
```
</details>
</br>

__Результат__
Настроены два параллельных pipilines, которые моделируют загрузку данных из хранилища в Clickhouse двумя способами:

- пакетная загрузка (batch ingestion) по 50000 строк или не реже, чем каждые 15 минут
- потоковая загрузка (streaming ingestion) ежесекундно, но блоками не более, чем 5000 строк за раз
При этом данные из источника считываются один раз в секунду - имитация реальной работы приложения

Источники данных это два одинаковых датасета в хранилище Minio: `trades_stream.parquet` и `trades_batch.parquet`
Целевые RAW таблицы в Clickhouse: `demo.binance_trades_stream` и `demo.binance_trades_batch`


--- 

#### 3.3 Aggregations and Materialized Views

На данном этапе для каждого потока данных создаю по две агрегированных по временным интервалам 15 минут и 1 час таблицы:

- `trades_stream_15m`
- `trades_batch_15m`
- `trades_stream_1h`
- `trades_batch_1h`
  
И создаю 4 Materialized View для организации вставки данных из RAW таблиц:

- `RAW STREAM`  ──► `stream_15m`
- `RAW BATCH`   ──► `batch_15m`

- `RAW STREAM`  ──► `stream_1h`
- `RAW BATCH`   ──► `batch_1h`


3.3.1. 
<details>
<summary>Таблица trades_stream_15m</summary>


```sql
CREATE TABLE IF NOT EXISTS demo.trades_stream_15m
ON CLUSTER replicated_cluster
(
    symbol LowCardinality(String),
    interval_start DateTime('UTC'),

    trades_count UInt64,
    volume Float64,

    min_price Float64,
    max_price Float64,
    avg_price Float64,
    vwap Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/trades_stream_15m',
    '{replica}'
)
PARTITION BY toDate(interval_start)
ORDER BY (symbol, interval_start);
```
</details>
</br>

3.3.2.
<details>
<summary>Таблица trades_batch_15m</summary>


```sql
CREATE TABLE IF NOT EXISTS demo.trades_batch_15m
ON CLUSTER replicated_cluster
(
    symbol LowCardinality(String),
    interval_start DateTime('UTC'),

    trades_count UInt64,
    volume Float64,

    min_price Float64,
    max_price Float64,
    avg_price Float64,
    vwap Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/trades_batch_15m',
    '{replica}'
)
PARTITION BY toDate(interval_start)
ORDER BY (symbol, interval_start);
```
</details>
</br>

3.3.3.
<details>
<summary>Таблица trades_stream_1h</summary>


```sql
CREATE TABLE IF NOT EXISTS demo.trades_stream_1h
ON CLUSTER replicated_cluster
(
    symbol LowCardinality(String),
    interval_start DateTime('UTC'),

    trades_count UInt64,
    volume Float64,

    min_price Float64,
    max_price Float64,
    avg_price Float64,
    vwap Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/trades_stream_1h',
    '{replica}'
)
PARTITION BY toDate(interval_start)
ORDER BY (symbol, interval_start);
```
</details>
</br>

3.3.4
<details>
<summary>Таблица trades_batch_1h</summary>


```sql
CREATE TABLE IF NOT EXISTS demo.trades_batch_1h
ON CLUSTER replicated_cluster
(
    symbol LowCardinality(String),
    interval_start DateTime('UTC'),

    trades_count UInt64,
    volume Float64,

    min_price Float64,
    max_price Float64,
    avg_price Float64,
    vwap Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/trades_batch_1h',
    '{replica}'
)
PARTITION BY toDate(interval_start)
ORDER BY (symbol, interval_start);
```
</details>
</br>
---

3.3.5.
<details>
<summary>Materialized View mv_stream_15m</summary>


```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_stream_15m
ON CLUSTER replicated_cluster
TO demo.trades_stream_15m
AS
SELECT
    symbol,
    toStartOfInterval(event_time, INTERVAL 15 MINUTE) AS interval_start,

    count() AS trades_count,
    sum(quantity) AS volume,

    min(price) AS min_price,
    max(price) AS max_price,
    avg(price) AS avg_price,

    sum(price * quantity) / sum(quantity) AS vwap

FROM demo.binance_aggtrades_stream
GROUP BY
    symbol,
    interval_start;
```
</details>
</br>

3.3.6.
<details>
<summary>Materialized View mv_batch_15m</summary>


```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_batch_15m
ON CLUSTER replicated_cluster
TO demo.trades_batch_15m
AS
SELECT
    symbol,
    toStartOfInterval(event_time, INTERVAL 15 MINUTE) AS interval_start,

    count() AS trades_count,
    sum(quantity) AS volume,

    min(price) AS min_price,
    max(price) AS max_price,
    avg(price) AS avg_price,

    sum(price * quantity) / sum(quantity) AS vwap

FROM demo.binance_aggtrades_batch
GROUP BY
    symbol,
    interval_start;
```
</details>
</br>

3.3.7.
<details>
<summary>Materialized View mv_stream_1h</summary>


```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_stream_1h
ON CLUSTER replicated_cluster
TO demo.trades_stream_1h
AS
SELECT
    symbol,
    toStartOfHour(event_time) AS interval_start,

    count() AS trades_count,
    sum(quantity) AS volume,

    min(price) AS min_price,
    max(price) AS max_price,
    avg(price) AS avg_price,

    sum(price * quantity) / sum(quantity) AS vwap

FROM demo.binance_aggtrades_stream
GROUP BY
    symbol,
    interval_start;
```
</details>
</br>


3.3.8.
<details>
<summary>Materialized View mv_batch_1h</summary>


```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_batch_1h
ON CLUSTER replicated_cluster
TO demo.trades_batch_1h
AS
SELECT
    symbol,
    toStartOfHour(event_time) AS interval_start,

    count() AS trades_count,
    sum(quantity) AS volume,

    min(price) AS min_price,
    max(price) AS max_price,
    avg(price) AS avg_price,

    sum(price * quantity) / sum(quantity) AS vwap

FROM demo.binance_aggtrades_batch
GROUP BY
    symbol,
    interval_start;
```
</details>
</br>

---

#### 3.4 Data Mart
Таблицы с JOIN trades + electricity: 
- `trades_energy_stream`
- `trades_energy_batch`
  
и две MW для вставки данных

3.4.1.
<details>
<summary>Таблица витрины STREAM - trades_energy_stream</summary>


```sql
CREATE TABLE IF NOT EXISTS demo.trades_energy_stream
ON CLUSTER replicated_cluster
(
    interval_start DateTime('UTC'),

    symbol LowCardinality(String),

    trades_count UInt64,

    volume Float64,

    avg_price Float64,

    vwap Float64,

    electricity_mw Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/trades_energy_stream',
    '{replica}'
)
PARTITION BY toDate(interval_start)
ORDER BY (symbol, interval_start);
```
</details>
</br>

3.4.2.
<details>
<summary>Таблица витрины BATCH - trades_energy_batch</summary>


```sql
CREATE TABLE IF NOT EXISTS demo.trades_energy_batch
ON CLUSTER replicated_cluster
(
    interval_start DateTime('UTC'),

    symbol LowCardinality(String),

    trades_count UInt64,

    volume Float64,

    avg_price Float64,

    vwap Float64,

    electricity_mw Float64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/demo/trades_energy_batch',
    '{replica}'
)
PARTITION BY toDate(interval_start)
ORDER BY (symbol, interval_start);
```
</details>
</br>

3.4.3.
<details>
<summary>Materialized View для STREAM - mv_trades_energy_stream</summary>


```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_trades_energy_stream
ON CLUSTER replicated_cluster
TO demo.trades_energy_stream
AS
SELECT
    t.interval_start,
    t.symbol,
    t.trades_count,
    t.volume,
    t.avg_price,
    t.vwap,
    e.electricity_mw
FROM demo.trades_stream_1h t
LEFT JOIN
(
    SELECT
        timestamp,
        sum(demand_mw) AS electricity_mw
    FROM demo.electricity_hourly
    GROUP BY timestamp
) e
ON t.interval_start = e.timestamp;
```
</details>
</br>

3.4.4.
<details>
<summary>Materialized View для BATCH - mv_trades_energy_batch</summary>


```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS demo.mv_trades_energy_batch
ON CLUSTER replicated_cluster
TO demo.trades_energy_batch
AS
SELECT
    t.interval_start,
    t.symbol,
    t.trades_count,
    t.volume,
    t.avg_price,
    t.vwap,
    e.electricity_mw
FROM demo.trades_batch_1h t
LEFT JOIN
(
    SELECT
        timestamp,
        sum(demand_mw) AS electricity_mw
    FROM demo.electricity_hourly
    GROUP BY timestamp
) e
ON t.interval_start = e.timestamp;
```
</details>
</br>


__Результаты шага__

- Спроектирована и реализована многослойная архитектура хранения данных в ClickHouse: RAW → Aggregation → Data Mart.
- Созданы RAW-таблицы для потоковой и пакетной загрузки данных и настроены Materialized Views для автоматического формирования агрегатов (15 минут и 1 час).
- На уровне Data Mart построены витрины данных с объединением торговых данных и почасового потребления электроэнергии.

Полученная схема обеспечивает подготовку данных для аналитики и последующей визуализации в BI-инструменте.

[⬆️ Наверх](#-оглавление)

---

### Шаг 4 — Мониторинг лаборатории

🎯 Цель:

Сделать систему наблюдаемой.

Добавляю в __Grafana__ дашборд `Batch vs Streaming`, в котором вывожу следующие метрики

#### 4.1 ClickHouse metrics
- `Insert rate` Количество INSERT-запросов в секунду по каждой ноде ClickHouse
- `Active Parts` - Количество активных частей на каждой ноде ClickHouse
- `Merge backlog` — Количество задач в очереди слияния частей данных (merge)
- `Query Latency` - Длительность выполнения запросов (в секундах)

---

#### 4.2 Kafka metrics
- `consumer lag`
- `throughput`
Kafka сам не отдаёт метрики в формате Prometheus. Поэтому используется специальный сервис: `Kafka Exporter`. Он читает Kafka и публикует метрики: http://kafka-exporter:9308/metrics . Prometheus потом их собирает.

__Развернуть Kafka Exporter__

На локальном сервере создаю каталог: ``` mkdir -p ~/infra/monitoring/kafka-exporter```
В этом каталоге 

<details>
<summary>Файл: docker-compose.yml</summary>

```yml
services:

  kafka-exporter:
    image: danielqsj/kafka-exporter:latest
    container_name: kafka-exporter

    command:
      - --kafka.server=kafka:9092

    ports:
      - "9308:9308"

    restart: unless-stopped

    networks:
      - infra-net


networks:
  infra-net:
    external: true
```
</details>
</br>


Что означает конфигурация `--kafka.server=kafka:9092`

Exporter подключается к Kafka внутри docker сети.

`9308` - это порт метрик Prometheus.

Запускаю контейнер `kafka-exporter`

Проверить метрики - на сервере: `curl http://localhost:9308/metrics | head`

Ожидается вывод примерно такого вида:
```bash
# HELP kafka_brokers Number of Brokers in the Kafka Cluster.
# TYPE kafka_brokers gauge
kafka_brokers 1
```

__Подключить Kafka exporter к Prometheus__

Открыть конфиг Prometheus на сервере: `nano ~/infra/monitoring/prometheus/prometheus.yml`
Добавить новый job

В конец scrape_configs:
```yml
  - job_name: kafka_exporter
    static_configs:
      - targets:
          - kafka-exporter:9308
```
Перезапустить Prometheus: `docker restart prometheus`
Проверить targets. Открыть в браузере: `http://<SERVER_IP>:9090`

Дальше: `Status → Targets`. Ожидается новый target: `kafka-exporter:9308` и статус: `UP`

__Панели Kafka metrics в Grafana:__

- Consumer lag — это отставание потребителя Kafka от продюсера (количество еще не обработанных сообщений)
- Throughput - Количество сообщений в секунду через Kafka

--- 

#### 4.3. Node Exporter (System)

- CPU usage % - загрузка процессора сервера
- RAM usage % - использование оперативной памяти
- Disk Write Throughput - сколько данных сервер записывает на диск в секунду
- Disk Usage % - процент заполнения жесткого диска
  
#### 4.4. Проверочные SQL-запросы (ClickHouse)

##### Query 1. ДЕГРАДАЦИЯ: parts vs merges (pressure snapshot)

🎯 Цель: понять — система переваривает поток или захлебывается


<details>
<summary>Query 1</summary>

```SQL
SELECT
    p.table,
    count() AS active_parts,
    countIf(m.table IS NOT NULL) AS active_merges,
    round(count() / nullIf(countIf(m.table IS NOT NULL), 0), 2) AS parts_to_merges_ratio
FROM system.parts p
LEFT JOIN system.merges m
    ON p.database = m.database
   AND p.table = m.table
WHERE p.database = 'demo'
  AND p.active = 1
  AND p.table IN (
      'binance_aggtrades_stream',
      'binance_aggtrades_batch')
GROUP BY p.table
ORDER BY parts_to_merges_ratio DESC;
```
</details>
</br>


🧠 Как читать

`parts_to_merges_ratio` — моментальный индикатор давления на merge subsystem.
Метрика интерпретируется только вместе с active_merges:
- если active_merges > 0, можно оценивать текущую интенсивность схлопывания parts;
- если active_merges = 0, высокий ratio не означает деградацию сам по себе, а лишь показывает, что в момент запроса merge-процессы не выполнялись.


##### Query 2. ПРОБЛЕМА РЕПЛИКАЦИИ (NO_REPLICA_HAS_PART)

🎯 Цель: найти ситуации, когда: одна реплика уже имеет part, а другая — нет


<details>
<summary>Query 2</summary>

Основной запрос

```SQL
SELECT
    hostName() AS host,
    database,
    table,
    replica_name,
    total_replicas,
    active_replicas,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    log_max_index - log_pointer AS replication_lag
FROM clusterAllReplicas('replicated_cluster', system.replicas)
WHERE database = 'demo';
```

Альтернатива
```SQL
SELECT
    database,
    table,
    is_readonly,
    is_session_expired,
    queue_size,
    inserts_in_queue,
    merges_in_queue,
    log_max_index - log_pointer AS replication_lag
FROM system.replicas
WHERE database = 'demo';
```
</details>
</br>

🧠 Как читать

поле	|смысл
------|-----
queue_size|очередь репликации
inserts_in_queue|вставки не применены
replication_lag|отставание

👉 если > 0 → уже есть проблема

##### Query 3. ЗАВИСШИЕ ЗАПРОСЫ / ОЧЕРЕДИ

🎯 Цель: увидеть, где “застряло”


🔴 Очередь репликации (самое важное)
<details>
<summary>Query 3</summary>

```SQL
SELECT
    hostName() AS host,
    database,
    table,
    replica_name,
    type,
    create_time,
    required_quorum,
    source_replica,
    new_part_name,
    is_currently_executing,
    num_tries,
    num_postponed,
    postpone_reason,
    last_exception,
    last_exception_time,
    last_attempt_time
FROM clusterAllReplicas('replicated_cluster', system.replication_queue)
WHERE database = 'demo'
ORDER BY create_time ASC;
```
</details>
</br>

🧠 Как читать

поле	|значение
------|--------
is_currently_executing = 0|не выполняется
num_tries > 0|уже были ошибки
last_exception|причина
старый create_time|завис

Если вывод запроса пустой, то очередь `replication_queue` пустая - все на момент запроса выполено

<details>
<summary>🔥 Важный индикатор</summary>

```SQL
SELECT
    replica_name,
    count() AS stuck_tasks
FROM clusterAllReplicas('replicated_cluster', system.replication_queue)
WHERE database = 'demo' 
  AND is_currently_executing = 0
  AND num_tries > 0
GROUP BY replica_name
ORDER BY stuck_tasks DESC;
```
</details>
</br>

👉 сразу видно “проблемную ноду”

---

#### 4.5. Пользователь `demo` для публичного просмотра сервисов проекта

Cоздание пользователей `demo` (пароль `DemoLab_2025!`) с правами только просмотра:
  * все таблицы Clickhouse - пользователь `demo` создан на шаге 2.4 . Доступ к Clickhouse осуществляется:
  - через WEB GUI по адресу clickhouse.myclickcourse.ru/play
  - через подключение (например DBeaver), параметры подключения: Хост - clickhouse.myclickcourse.ru, Порт - 443, Пользователь - demo, use_server_time_zone=false, ssl=true, protocol=https
  * дашборды Grafana - пользователь `demo` задается через соответствующее меню в графическом интерфейсе 
  * интерфейс Airflow - пользователь создается с помощью
  <details>
<summary>команды</summary>

```bash
docker exec -it airflow-apiserver airflow users create \
  --username demo \
  --firstname Demo \
  --lastname User \
  --role Viewer \
  --email demo@example.com \
  --password DemoLab_2025!
```
</details>
</br>

* ssh доступ к Ubuntu Server с правами просмотра каталога проекта реализован через настройки конфигураций облачного и локального серверов следущим образом: пользователь `demo` получает доступ к папке проекта через команду `ssh demo@ssh.myclickcourse.ru` - дважды ввести пароль. Для упрощения доступа пользователя `demo` реализован SSH jump host через VDS. Пользователь подключается к публичному домену, после чего автоматически проксируется на внутренний сервер в приватной сети WireGuard. Путь к директории проекта `/home/alex/infra`, для удобства просмотра можно использовать `tree`.

* доступ к дашборду Datalens осуществляется по публичной ссылке и не требует авторизации

--- 


#### 4.6. Apache Airflow


__Airflow Metrics__

В ходе выполнения работы выявилось нецелесообразность этой визуализации - нечего выводить для пользы проекта, поэтому данные метрики в окончательном виде не реализовывались. Параметры и логи работы DAG`s можно увидеть через веб-интерфейс Apache Airflow. Ниже - объем проделанной работы по настройке мониторинга Airflow.

<details>
<summary>Команды и код для настройки мониторинга Apache Airflow</summary>

__Поднять StatsD exporter для мониторинга Airflow__
Создать папку на сервере: ```mkdir -p ~/infra/monitoring/airflow-metrics``` и в ней -

<details>
<summary>Файл: docker-compose.yml</summary>

```yml
services:

  statsd-exporter:
    image: prom/statsd-exporter
    container_name: statsd-exporter

    ports:
      - "9103:9102"
      - "9125:9125/udp"

    restart: unless-stopped

    networks:
      - infra-net

networks:
  infra-net:
    external: true
```
</details>
</br>

Запустить контейнер и проверить endpoint ```curl http://localhost:9103/metrics | head```

Должны появиться метрики Prometheus.

__Включить StatsD в Airflow__

В файл `.env` Airflow в раздел метрик добавить строки и перезапустить Airflow:

```bash
AIRFLOW__METRICS__STATSD_ON=True
AIRFLOW__METRICS__STATSD_HOST=statsd-exporter
AIRFLOW__METRICS__STATSD_PORT=9125
AIRFLOW__METRICS__STATSD_PREFIX=airflow
```
Перезапуск:
```bash
docker compose down
docker compose up -d --build
```
Проверить, что метрики появились: `curl http://localhost:9103/metrics | grep airflow`

__Подключить statsd-exporter в Prometheus__

Открыть конфиг Prometheus `nano ~/infra/monitoring/prometheus/prometheus.yml`

В блоке `scrape_configs` добавить новый job:
```yml
- job_name: airflow_exporter
  static_configs:
    - targets:
        - statsd-exporter:9102
```
Важно: использовать внутренний порт контейнера `9102`. Перезапустить `docker restart prometheus`
</details>
</br>

---


#### Результат шага

Развернута система мониторинга на базе Prometheus и Grafana, обеспечивающая наблюдаемость потоковой и пакетной загрузки данных. Настроены метрики ClickHouse, Kafka и системных ресурсов сервера, а также реализованы SQL-запросы для диагностики деградации, репликации и очередей. Созданы публичные доступы для демонстрации системы через Grafana, ClickHouse и Airflow. Это позволяет в реальном времени анализировать состояние инфраструктуры и сравнивать поведение batch и streaming ingestion.

[⬆️ Наверх](#-оглавление)

---

### Шаг 5 — Визуализация BI

🎯 Цель:

Сравнение __batch__ vs __streaming__.

#### 5.1 Подключение к DataLens

Настройки подключения к Clickhouse:
- Хост = `clickhouse.myclickcourse.ru`
- Порт = `443`
- Пользователь = `demo`
- Пароль = `DemoLab_2025!`
- TLS = `вкл`
- `Разрешить подзапросы в датасетах` = `вкл`

#### 5.2 Дашборд: Batch vs Streaming

Реализованы три вкладки:
1.	Batch ingestion
	* Источник: batch pipeline (ClickHouse)
	* KPI: количество строк, сделок, объём, средняя цена
	* *Графики: динамика по часам, интервалы 15m и 1h
2.	Streaming ingestion
	* Полностью аналогична Batch
	* Отличие: источник — streaming pipeline
	* Позволяет напрямую сравнивать поведение ingestion
3.	Показатели ClickHouse
	* Системные метрики:
	* количество parts, строк и размер таблиц
	* соотношение parts / merges
	* состояние репликации
	* зависшие очереди

#### 5.3 Синхронизированные фильтры

- Селектор по валютной паре (symbol)
- Применяется ко всем графикам на вкладках Batch и Streaming
- Обеспечивает единый анализ по выбранному инструменту


#### Результат шага:
Построен BI-дашборд для сравнения batch и streaming загрузки данных.
Реализованы ключевые метрики бизнес-уровня и системного уровня.
Обеспечена возможность интерактивного анализа через селектор валютных пар.
Дашборд позволяет визуально сравнивать поведение pipeline и состояние ClickHouse в режиме, близком к production.

[⬆️ Наверх](#-оглавление)

---

### Шаг 6 — Управление жизненным циклом

🎯 Цель:

Обеспечить контролируемый запуск и завершение экспериментов, а также подготовку системы к повторным запускам без накопления данных и искажения результатов.


#### 6.1 TRUNCATE после цикла

После завершения эксперимента все данные, записанные в ClickHouse, должны быть удалены, чтобы:
	•	исключить влияние предыдущих запусков на новые эксперименты
	•	сохранить корректность метрик (insert rate, parts, latency)
	•	обеспечить воспроизводимость результатов

Очистка выполняется с помощью команды:

TRUNCATE TABLE <table_name>;

Очистке подлежат:
- RAW таблицы:
	* binance_aggtrades_stream
	* binance_aggtrades_batch
- агрегаты:
  * trades_stream_15m
  * trades_stream_1h
  * trades_batch_15m
  * trades_batch_1h
- витрины:
	* trades_energy_stream
	* trades_energy_batch

Справочные данные:
- `electricity_hourly` не чистим, потому что эти данные не меняются и на результат экспериментов не влияют

---

#### 6.2 DAG для очистки

Реализован отдельный DAG: binance_cleanup

Назначение:
- централизованная очистка всех таблиц
- возможность запуска вручную перед экспериментом
- логирование процесса очистки

Логика DAG:

Airflow → PythonOperator → TRUNCATE всех таблиц

<details>
<summary>Файл binance_cleanup.py</summary>

```SQL
from airflow import DAG
from airflow.providers.standard.operators.python import PythonOperator
from airflow.hooks.base import BaseHook

from datetime import datetime
import clickhouse_connect
import logging


# =========================
# LOGGING
# =========================

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)


# =========================
# CLICKHOUSE CONNECTION
# =========================

def get_ch_client():

    conn = BaseHook.get_connection("clickhouse_demo")

    return clickhouse_connect.get_client(
        host=conn.host,
        port=conn.port,
        username=conn.login,
        password=conn.password,
        database=conn.schema
    )


# =========================
# CLEANUP TASK
# =========================

def cleanup_clickhouse():

    logger.info("Starting ClickHouse cleanup")

    client = get_ch_client()

    # electricity_hourly не чистим
    tables = [
        "binance_aggtrades_batch",
        "binance_aggtrades_stream",
        "kafka_binance_stream",
        "trades_batch_1h",
        "trades_batch_15m",
        "trades_energy_batch",
        "trades_energy_stream",
        "trades_stream_1h",
        "trades_stream_15m"
    ]

    for table in tables:

        try:
            query = f"TRUNCATE TABLE {table}"

            logger.info(f"Truncating table: {table}")

            client.command(query)

            logger.info(f"Table cleared: {table}")

        except Exception as e:
            logger.error(f"Error truncating {table}: {e}")

    logger.info("Cleanup finished")


# =========================
# DAG
# =========================

with DAG(
    dag_id="binance_cleanup",
    start_date=datetime(2025, 1, 1),
    schedule=None,
    catchup=False,
    tags=["binance", "cleanup"]
) as dag:

    cleanup = PythonOperator(
        task_id="cleanup_clickhouse",
        python_callable=cleanup_clickhouse
    )
```
</details>
</br>

Особенности:
- используется подключение через Airflow Connections
- обработка ошибок (логирование при сбое очистки)
- выполняется быстро (метаданные очищаются без полного удаления файлов)

#### 6.3. DAG для подготовки следующего эксперимента с интенсивностью х5

Подготовлен DAG, который:
- считывает данные из первичных исторических датасетов
- генерирует новые данные (х5), увеличивая количество строк в 5 раз, заполнение данных осуществляется на основе соседних по метке времени строк с небольшими отклонениями с сохранением типов и форматов данных
- записывает данные в хранилище Minio в датасеты `trades_stream_X5.parquet` и `trades_batch_X5.parquet` отедльно для каждого потока данных 

<details>
<summary>Файл binance_build_parquet_datasetX5.py</summary>

```python
from airflow import DAG
from airflow.providers.standard.operators.python import PythonOperator
from datetime import datetime
import zipfile
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq
import os
import numpy as np
from minio import Minio


# -----------------------------------------
# CONFIG
# -----------------------------------------

DATA_DIR = "/opt/airflow/data/binance"

FILES = [
    "BTCUSDT-aggTrades-2023-07.zip",
    "ETHUSDT-aggTrades-2023-07.zip",
    "BTCUSDC-aggTrades-2023-07.zip",
    "ETHUSDC-aggTrades-2023-07.zip",
]

LOCAL_STREAM_PARQUET = "/opt/airflow/data/binance/trades_stream_X5.parquet"
LOCAL_BATCH_PARQUET = "/opt/airflow/data/binance/trades_batch_X5.parquet"

MINIO_BUCKET = "clickhouse-lab-datasets"

MINIO_STREAM_OBJECT = "binance/july_2023/trades_stream_X5.parquet"
MINIO_BATCH_OBJECT = "binance/july_2023/trades_batch_X5.parquet"


minio_client = Minio(
    "minio:9000",
    access_key="admin",
    secret_key="AdminLab_2025!",
    secure=False
)


# -----------------------------------------
# GENERATE X5 DATA
# -----------------------------------------

def generate_x5(df: pd.DataFrame) -> pd.DataFrame:
    dfs = [df]

    for i in range(4):  # исходный + 4 копии = x5
        tmp = df.copy()

        # небольшой случайный сдвиг времени
        shift_ms = np.random.randint(1, 50, size=len(tmp))
        tmp["event_time"] = tmp["event_time"] + pd.to_timedelta(shift_ms, unit="ms")

        # небольшой шум по цене и количеству
        tmp["price"] = tmp["price"] * (1 + np.random.normal(0, 0.0005, len(tmp)))
        tmp["quantity"] = tmp["quantity"] * (1 + np.random.normal(0, 0.001, len(tmp)))

        # уникализируем ID
        tmp["agg_trade_id"] = tmp["agg_trade_id"] + (i + 1) * 10_000_000_000
        tmp["first_trade_id"] = tmp["first_trade_id"] + (i + 1) * 10_000_000_000
        tmp["last_trade_id"] = tmp["last_trade_id"] + (i + 1) * 10_000_000_000

        dfs.append(tmp)

    result = pd.concat(dfs, ignore_index=True)
    return result


# -----------------------------------------
# TASK
# -----------------------------------------

def build_dataset_x5():

    dfs = []

    for filename in FILES:

        symbol = filename.split("-")[0]
        zip_path = os.path.join(DATA_DIR, filename)

        print(f"Processing {zip_path}")

        with zipfile.ZipFile(zip_path) as z:
            csv_name = z.namelist()[0]

            with z.open(csv_name) as f:
                df = pd.read_csv(
                    f,
                    header=None,
                    names=[
                        "agg_trade_id",
                        "price",
                        "quantity",
                        "first_trade_id",
                        "last_trade_id",
                        "event_time",
                        "is_buyer_maker",
                        "is_best_match"
                    ]
                )

        # удаляю лишнюю колонку
        df = df.drop(columns=["is_best_match"])

        # нормализация timestamp
        df["event_time"] = pd.to_numeric(df["event_time"], errors="coerce")
        df = df.dropna(subset=["event_time"])

        df["event_time"] = pd.to_datetime(
            df["event_time"].astype("int64"),
            unit="ms",
            utc=True
        )

        # добавляю symbol
        df["symbol"] = symbol

        # генерирую X5
        df_x5 = generate_x5(df)

        dfs.append(df_x5)

    # объединяю все пары
    full_df = pd.concat(dfs, ignore_index=True)

    print("Dataset rows (X5):", len(full_df))

    # сортировка по времени
    full_df = full_df.sort_values("event_time")

    # оптимизация типов
    full_df["is_buyer_maker"] = full_df["is_buyer_maker"].astype("int8")

    # защита от отрицательных значений после шума
    full_df["price"] = full_df["price"].clip(lower=0.00000001)
    full_df["quantity"] = full_df["quantity"].clip(lower=0.00000001)

    # конвертация в parquet
    print("Converting to Parquet")

    table = pa.Table.from_pandas(full_df, preserve_index=False)

    pq.write_table(table, LOCAL_STREAM_PARQUET)
    pq.write_table(table, LOCAL_BATCH_PARQUET)

    # -----------------------------------------
    # Upload STREAM dataset
    # -----------------------------------------

    print("Uploading STREAM X5 dataset to MinIO")

    minio_client.fput_object(
        MINIO_BUCKET,
        MINIO_STREAM_OBJECT,
        LOCAL_STREAM_PARQUET
    )

    # -----------------------------------------
    # Upload BATCH dataset
    # -----------------------------------------

    print("Uploading BATCH X5 dataset to MinIO")

    minio_client.fput_object(
        MINIO_BUCKET,
        MINIO_BATCH_OBJECT,
        LOCAL_BATCH_PARQUET
    )

    print("Datasets X5 uploaded successfully")


# -----------------------------------------
# DAG
# -----------------------------------------

default_args = {
    "owner": "airflow",
    "retries": 1
}

with DAG(
    dag_id="binance_build_parquet_dataset_x5",
    start_date=datetime(2025, 1, 1),
    schedule=None,
    catchup=False,
    default_args=default_args,
    tags=["dataset", "binance", "x5"]
) as dag:

    build = PythonOperator(
        task_id="build_parquet_dataset_x5",
        python_callable=build_dataset_x5
    )
```
</details>
</br>

#### 6.4. Для повторения эксперимента следует запустить DAG:
- `binance_cleanup` - очистки данных 
- `binance_replay_sync` - загрузки данных в Clickhouse по двум потокам batch и streaming, перед стартом изменить источник данных на `//clickhouse-lab-datasets/binance/july_2023/tradesX5.parquet`

#### Результат шага

Реализован механизм управления жизненным циклом данных, включающий:
- очистку всех слоев хранилища (RAW, AGG, Data Mart)
- отдельный DAG для подготовки новых данных
- возможность повторного запуска экспериментов без ручного вмешательства

Это обеспечивает:
- воспроизводимость экспериментов
- корректность метрик
- удобство эксплуатации системы

[⬆️ Наверх](#-оглавление)

---

### Шаг 7 - Выводы
по результатам проделанной работы

🎯 Цель этапа:
Оценить результаты работы и подтвердить достижение целей проекта.


#### Достижение цели проекта

В рамках проекта был успешно реализован MVP аналитической платформы на базе ClickHouse, позволяющий сравнить два подхода загрузки данных:
* Batch ingestion
* Streaming ingestion

Смоделированы условия, приближенные к реальной эксплуатации:
* неравномерный поток данных
* параллельная загрузка
* увеличенная интенсивность (×5)

#### Ключевые наблюдения

Streaming ingestion:
- создаёт большое количество мелких parts
- увеличивает нагрузку на merge-процессы
- чувствителен к пиковым нагрузкам
- обеспечивает near real-time доступ к данным

Batch ingestion:
- формирует крупные блоки данных
- снижает нагрузку на систему
- работает стабильно при высокой нагрузке
- имеет задержку в доступности данных

#### Результаты визуализации

В Yandex DataLens реализованы:
- два идентичных дашборда (Batch vs Streaming)
- синхронизированные фильтры (symbol)
- метрики загрузки и агрегации
- системные показатели ClickHouse

Это позволило:
- визуально сравнить поведение пайплайнов
- выявить различия в нагрузке и производительности
- подтвердить гипотезы эксперимента

#### MVP проекта

Реализован минимально жизнеспособный продукт (MVP), включающий:
- ingestion layer (Kafka + Airflow)
- storage layer (ClickHouse cluster)
- processing layer (Materialized Views)
- data mart (агрегированные витрины)
- BI слой (DataLens dashboards)
- monitoring (Prometheus + Grafana)

MVP полностью воспроизводим и позволяет повторять эксперимент с разной нагрузкой.

При использовании оригинального датасета осуществлялась вставка данных в объеме около 900 000 строк в сутки на поток. При используемой конфигурации сервера и кластера Clickhouse такая плотность данных не оказала заметного влияния на снижение производительности, поэтому как только это стало понятно, были запущены потоки с увелчиенной в пять раз плотностью потока данных.

---

🏁 Итог

Проект:
1. Демонстрирует реальные различия между batch и streaming подходами
2. Показывает влияние архитектурных решений на производительность
3. Подтверждает важность балансировки ingestion-стратегии

👉 Основной вывод:
__Streaming__ даёт скорость, __Batch__ — стабильность. Оптимальная архитектура требует их осознанного сочетания.

## Ссылки на сервисы

- [clickhouse.myclickcourse.ru/play веб интерфейс Clickhouse](clickhouse.myclickcourse.ru/play)
- [airflow.myclickcourse.ru веб интерфейс Apache Airflow](airflow.myclickcourse.ru)
- [grafana.myclickcourse.ru веб интерфейс Grafana](grafana.myclickcourse.ru)
- ssh доступ: ssh.myclickcourse.ru

Доступ ко всем сервисам: пользователь `demo` пароль `DemoLab_2025!`

## Скриншоты
состояния системы при запуске загрузки из оригинального датасета (без генерации х5)

- ![Скриншот 1](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen1.png)
- ![Скриншот 2](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen2.png)
- ![Скриншот 3](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen3.png)
- ![Скриншот 4](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen4.png)
- ![Скриншот 5](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen5.png)
- ![Скриншот 6](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen6.png)
- ![Скриншот 7](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen7.png)
- ![Скриншот 8](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen8.png)
- ![Скриншот 9](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen9.png)
- ![Скриншот 10](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen10.png)
- ![Скриншот 11](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen11.png)
- ![Скриншот 12](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen12.png)
- ![Скриншот 13](https://github.com/realexpert1C/clickhouse-course/blob/22e6481cf709f131d5794cbe044963d306657816/images/pf_screen13.png)

## Что далее?
В настоящее время время основной DAG проекта, моделирующий два параллельных потока данных запущен в демонстрационных целях на 30 дней с генерацией дополнительных строк с коэффициентом 5 к оригиналу.

Возможные направения развиия платформы:
- Настройка ежедневного запуска DAG для моделирования сбоев при загрузке и догоняющей загрузки данных после сбоев на сервере
- Добавление управляемой настроки плотности потока данных - произвольное изменение коэффициента генерации строк для выявления баланса после которого наступает деградация системы
- Добавление возможности изменять не только количество строк данных, но и ширину (количество столбцов) RAW таблиц
- Добавление возможности раздельного запуска потоков загрузки
- Добавление алертов по ключевым показателям (зависшие запросы, ошибки NO_REPLICA_HAS_PART и др.) с выводом сообщений в Телеграмм-бот, настройка запуска DAG через Телеграм-бот
- Синхронизация содержимого airflow/dags c Git для возможности создания дагов в удаленном режиме без прмой правки на сервере и обеспечения контроля версий
- Создание полноценной системы резервного копирования данных
- Прогон тестовых данных с разными настройками Clickhouse (размер блока вставки, число параллельных потоков вставки, optimize_on_insert = 1/0)
- Реализовать полный  Product Overview аналитической платформы


[⬆️ Наверх](#-оглавление)