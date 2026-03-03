
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

### 1.8 Мониторинг и наблюдаемость системы

#### Цель мониторинга

В рамках проекта реализуется полноценная система наблюдаемости (observability), позволяющая:
- контролировать корректность ingestion (batch и streaming),
- анализировать поведение ClickHouse под нагрузкой,
- отслеживать деградацию при burst-нагрузке,
- выявлять узкие места,
- сравнивать эффективность двух способов загрузки данных.


#### Архитектура мониторинга

Используются:
* Prometheus — сбор метрик
* Grafana — визуализация и алертинг

Общая схема:
```txt
Replay Service
        ↓
      Kafka
        ↓
    ClickHouse
        ↓
    Prometheus
        ↓
Grafana Dashboards
```

#### 1.8.1. Мониторинг Replay Service

Replay Service является активным участником системы и подлежит мониторингу наравне с ClickHouse и Kafka.

__Метрики производительности__

Позволяют контролировать фактическую скорость генерации данных:
* replay_events_total
* replay_events_per_second
* kafka_messages_sent_total
* batch_flush_total
* batch_size_current

Позволяют определить:
* идут ли события корректно,
* соответствует ли скорость ожидаемой при текущем TIME_SCALE,
* применился ли INTENSITY_K,
* нет ли падения throughput.


__Метрики ошибок__

Критически важны для стабильности эксперимента:
* kafka_send_errors_total
* clickhouse_insert_errors_total
* retry_count_total

Используются для алертинга при превышении допустимого уровня ошибок.

__Метрики задержек__

Позволяют выявлять деградацию системы:
* batch_flush_duration_seconds
* kafka_send_latency_seconds
* replay_processing_lag_seconds

Позволяют увидеть:
- замедление batch-вставок,
- зависание Kafka producer,
- накопление внутреннего лага.

---

#### 1.8.2. Мониторинг ClickHouse

Контроль состояния кластера и поведения при нагрузке.

Основные метрики:
- Insert rate
- PartsNumber (количество частей)
- BackgroundMergesAndMutationsPoolTask
- ReplicasMaxQueueSize
- DelayedInserts
- QueryDuration

Анализируется:
* рост числа мелких частей,
* backlog merge-процессов,
* задержка вставок,
* влияние нагрузки на выполнение аналитических запросов.

---

#### 1.8.3. Мониторинг Kafka

Контроль потоковой составляющей ingestion.

Основные метрики:
* consumer lag
* messages in per second
* messages out per second
* producer error rate

Позволяет выявлять:
- накопление lag,
- перегрузку топиков,
- проблемы доставки сообщений.

---

#### 1.8.4. Системные метрики

Собираются через Node Exporter:
* CPU usage
* RAM usage
* IO wait
* Disk write rate
* Disk space usage

Необходимы для анализа:
- утилизации ресурсов,
- достижения предельной нагрузки,
- влияния batch vs streaming на железо.

---

#### 1.8.5. Алертинг

__Алерты для Replay Service__
* отсутствие новых событий > 60 секунд
* batch не выполнялся > 20 минут
* error rate > 1%
* изменение INTENSITY_K без роста throughput

__Алерты для ClickHouse__
* parts > 10 000
* merges backlog превышает порог
* insert queue delay выше допустимого значения
* рост replication queue


__Алерты для Kafka__
* consumer lag растёт более 5 минут
* producer error

---

#### 1.8.6. Логирование Replay Service

Replay Service ведёт структурированное логирование:
* формат JSON
* уровни: INFO / WARNING / ERROR
* вывод в stdout (Docker logs)

Опционально возможно:
* интеграция с Promtail + Loki
* централизованное хранение логов

---

#### 1.8.7. Единый Dashboard эксперимента

В Grafana формируется отдельный dashboard для анализа ingestion-эксперимента.

В него включаются:
* Replay rate
* Batch insert rate
* Kafka throughput
* ClickHouse insert rate
* Parts count
* Merge backlog
* CPU / RAM
* Disk IO

Мониторинг позволяет наблюдать поведение всей системы как в моменты загрузки данных, так и переключения между batch и streaming дашбордами в DataLens. Другими словами получить то, что является целью настоящего проекта - состояние и реакцию системы при изменении нагрузки.

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

### 1.12. Архитектурная ценность проекта

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
 - развернут основной сервер проекта
 - на сервере создана основная часть архитектуры - кластер базы данных Clickhouse в контейнерах Docker в единой локальной сети `infra-net`


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