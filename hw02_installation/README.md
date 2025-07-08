# 📘 Конспект: Развертывание и базовая конфигурация ClickHouse

## 🎯 Цели вебинара
- Познакомиться с вариантами установки ClickHouse
- Изучить конфигурацию и основные настройки
- Рассмотреть интерфейсы доступа и инструменты

---

## ⚙️ Варианты установки ClickHouse
1. **ClickHouse Cloud** – SaaS-решение
2. **Managed ClickHouse** – Яндекс, VK Cloud
3. **Quick Install** – `curl https://clickhouse.com/ | sh`
4. **Docker образ** – `clickhouse/clickhouse-server`
5. **DEB/RPM пакеты** – для Linux-дистрибутивов
6. **ClickHouse Keeper** – замена ZooKeeper для репликации
7. **Kubernetes-кластеры** – ClickHouse Operator

---

## 🔧 Базовая конфигурация
- Файл `config.xml` — системные настройки
- Файл `users.xml` — пользователи и сессии
- Пользовательские настройки: `config.d/` и `users.d/`

### Пример:
```xml
<clickhouse>
  <users>
    <default>
      <profile>default</profile>
      <networks><ip>::/0</ip></networks>
      <password>...</password>
    </default>
  </users>
</clickhouse>
```
⸻

⚙️ Настройки производительности
	•	max_memory_usage
	•	max_threads
	•	max_execution_time
	•	merge_tree_* параметры
	•	Ограничения на JOIN, GROUP BY, и внешнюю память

⸻

🧪 Рекомендации после установки
	1.	Настройка лимитов ulimit, kernel.shmmax
	2.	Отключить transparent_hugepage
	3.	Использовать noop или deadline планировщик
	4.	Мониторинг через встроенные метрики и system.tables, system.metrics

⸻

🧩 Интерфейсы доступа

Сетевые:
	•	HTTP (8123), TCP (9000), gRPC

CLI:
```xml 
clickhouse-client -h localhost -d default --query="SELECT 1"

⸻

HTTP:
```xml 
curl 'http://localhost:8123/?query=SELECT%201'

JDBC/ODBC:
	•	ClickHouse JDBC и ODBC для BI-инструментов (Tableau, Excel, PowerBI)

⸻

📥 Вставка данных

```xml
cat file.csv | clickhouse-client --query="INSERT INTO table FORMAT CSV"

⸻

📦 Подключение внешних источников (JDBC)
	•	Поддержка jdbc() и ENGINE=JDBC
	•	Использование ClickHouse JDBC Bridge

⸻

🧠 Автоопределение схемы
	•	Поддержка DESC format(...)
	•	Кэширование схем system.schema_inference_cache

⸻

📌 Домашнее задание
	1.	Установить ClickHouse
	2.	Загрузить тестовый датасет
	3.	Выполнить бенчмаркинг
	4.	Провести конфигурирование
	5.	Сравнить производительность до/после
	6.	Оформить отчет (PDF/Git)

⸻

🧠 Полезные ссылки
	•	Документация ClickHouse
	•	Инсталляция ClickHouse
	•	ClickHouse Keeper




## 📄 Домашнее задание: Развертывание и базовая конфигурация, интерфейсы и инструменты

---

### ⚖️ Подготовка к работе. Базовая инфраструктура

#### Установленные компоненты

* Ubuntu 22.04 LTS (облачный сервер)
* Docker, Docker Compose
* Git

#### Созданные контейнеры

* ClickHouse
* MariaDB
* PostgreSQL 15 (внешняя)
* PostgreSQL 13 (внутренняя БД Airflow)
* Airflow 3.0

---

### ⚙️ Конфигурация сетей Docker

Создание кастомной bridge-сети `infra-net`:

```bash
docker network create infra-net
```

Подключение контейнеров к сети:

```yaml
networks:
  - infra-net
```

Определение сети в `docker-compose.yaml`:

```yaml
networks:
  infra-net:
    external: true
```

---

### 🚀 Дальнейшие действия

* Подключение Portainer для управления контейнерами
* Проверка подключения к ClickHouse через WEB и clickhouse-client

---

### ✅ Шаг 1: Установка и запуск ClickHouse

Запуск контейнера ClickHouse с пробросом портов:

```bash
docker run -d \
  --name clickhouse \
  --network infra-net \
  -p 8123:8123 \
  -p 9000:9000 \
  -p 9009:9009 \
  -v clickhouse_data:/var/lib/clickhouse \
  -v clickhouse_logs:/var/log/clickhouse-server \
  -e CLICKHOUSE_DB=default \
  -e CLICKHOUSE_USER=default \
  -e CLICKHOUSE_PASSWORD=my_pass \
  clickhouse/clickhouse-server:latest
```

Установка клиента ClickHouse:

```bash
sudo apt install clickhouse-client -y
```

Подключение к серверу:

```bash
clickhouse-client --host 127.0.0.1 --port 9000 --user default --password my_pass
```

---

### ✅ Шаг 2: Загрузка датасета

Скачивание и загрузка UK Price Paid: Инструкция: [ClickHouse Dataset Docs](https://clickhouse.com/docs/getting-started/example-datasets/uk-price-paid)

Создание таблицы:

```sql
CREATE TABLE default.uk_price_paid (
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8('terraced' = 1, 'semi-detached' = 2, 'detached' = 3, 'flat' = 4, 'other' = 0),
    is_new UInt8,
    duration Enum8('freehold' = 1, 'leasehold' = 2, 'unknown' = 0),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
) ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2);
```

Загрузка данных из CSV:

```sql
INSERT INTO uk.uk_price_paid
SELECT
    toUInt32(price_string) AS price,
    parseDateTimeBestEffortUS(time) AS date,
    splitByChar(' ', postcode)[1] AS postcode1,
    splitByChar(' ', postcode)[2] AS postcode2,
    transform(a, ['T', 'S', 'D', 'F', 'O'], ['terraced', 'semi-detached', 'detached', 'flat', 'other']) AS type,
    b = 'Y' AS is_new,
    transform(c, ['F', 'L', 'U'], ['freehold', 'leasehold', 'unknown']) AS duration,
    addr1,
    addr2,
    street,
    locality,
    town,
    district,
    county
FROM url(
    'http://prod1.publicdata.landregistry.gov.uk.s3-website-eu-west-1.amazonaws.com/pp-complete.csv',
    'CSV',
    'uuid_string String,
    price_string String,
    time String,
    postcode String,
    a String,
    b String,
    c String,
    addr1 String,
    addr2 String,
    street String,
    locality String,
    town String,
    district String,
    county String,
    d String,
    e String'
) SETTINGS max_http_get_redirects=10;
```

К базе данных сделан ряд запросов. 
Все запросы отработали корректно.

---

### ✅ Шаг 3: Скриншоты проверки

* Подключение к ClickHouse: [Скриншот1](https://github.com/realexpert1C/clickhouse-course/blob/main/images/connect1.png), [Скриншот2](https://github.com/realexpert1C/clickhouse-course/blob/main/images/connect2.png), [Скриншот3](https://github.com/realexpert1C/clickhouse-course/blob/main/images/connect3.png), [Скриншот4](https://github.com/realexpert1C/clickhouse-course/blob/main/images/connect4.png)
* Запросы: [Скриншот5](https://github.com/realexpert1C/clickhouse-course/blob/main/images/select1.png), [Скриншот6](https://github.com/realexpert1C/clickhouse-course/blob/main/images/select2.png), [Скриншот7](https://github.com/realexpert1C/clickhouse-course/blob/main/images/select3.png), [Скриншот8](https://github.com/realexpert1C/clickhouse-course/blob/main/images/select4.png)

---

### ✅ Шаг 4: Тестирование производительности

Запуск:

```bash
echo "SELECT * FROM default.uk_price_paid LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark --port 9000 --user default --password my_pass -i 10
```

  
#### Тест 1: Конфигурация по умолчанию

```bash
Loaded 1 queries.

Queries executed: 10 (100%).

localhost:9000, queries: 10, QPS: 0.813, RPS: 16269366.674, MiB/s: 688.080, result RPS: 8131040.631, result MiB/s: 321.809.

0%        1.023 sec.  
10%       1.026 sec.  
20%       1.129 sec.  
30%       1.188 sec.  
40%       1.209 sec.  
50%       1.225 sec.  
60%       1.225 sec.  
70%       1.234 sec.  
80%       1.278 sec.  
90%       1.361 sec.  
95%       1.464 sec.  
99%       1.464 sec.  
99.9%     1.464 sec.  
99.99%    1.464 sec.
```

---



### 🛠️ Шаг 5: Изучение конфигурации ClickHouse

#### Основные файлы

* `/etc/clickhouse-server/config.xml`
* `/etc/clickhouse-server/users.xml`
* `/etc/clickhouse-server/config.d/`
* `/etc/clickhouse-server/users.d/`

#### Просмотр

```bash
docker exec -it clickhouse cat /etc/clickhouse-server/config.xml
```

---

### 🛠️ Шаг 6: Оптимизация и повторное тестирование

Создал кастомный конфиг в двух вариантах
#### Тест 2: Кастомный конфиг (ограничение использования ресурсов до 80%)

```xml
<clickhouse>
    <max_server_memory_usage>10307921510</max_server_memory_usage> 
    <max_memory_usage>1030792151</max_memory_usage> 
    <max_threads>6</max_threads> 
    <max_concurrent_queries>10</max_concurrent_queries>
    <background_pool_size>6</background_pool_size>
    <background_buffer_flush_schedule_pool_size>2</background_buffer_flush_schedule_pool_size>
    <background_fetches_pool_size>3</background_fetches_pool_size>
</clickhouse>
```

```bash
Queries executed: 10 (100%).

localhost:9000, queries: 10, QPS: 0.730, RPS: 14619446.917, MiB/s: 618.571, result RPS: 7300472.300, result MiB/s: 289.173.

0%        1.162 sec.  
10%       1.178 sec.  
20%       1.220 sec.  
30%       1.226 sec.  
40%       1.275 sec.  
50%       1.376 sec.  
60%       1.376 sec.  
70%       1.453 sec.  
80%       1.465 sec.  
90%       1.513 sec.  
95%       1.640 sec.  
99%       1.640 sec.  
99.9%     1.640 sec.  
99.99%    1.640 sec.
```


#### Тест 3: Кастомный конфиг 2 (расширенные фоны и кэш)

```xml
<clickhouse>
    <merge_tree>
        <number_of_free_entries_in_pool_to_execute_mutation>0</number_of_free_entries_in_pool_to_execute_mutation>
    </merge_tree>

    <background_pool_size>16</background_pool_size>
    <background_buffer_flush_schedule_pool_size>4</background_buffer_flush_schedule_pool_size>
    <background_fetches_pool_size>6</background_fetches_pool_size>

    <query_cache>
        <size_in_bytes>1073741824</size_in_bytes>
        <max_entries>4096</max_entries>
    </query_cache>

    <max_threads>0</max_threads>
    <max_concurrent_queries>100</max_concurrent_queries>
</clickhouse>
```

```bash
Queries executed: 10 (100%).

localhost:9000, queries: 10, QPS: 0.870, RPS: 17412431.214, MiB/s: 736.929, result RPS: 8699823.673, result MiB/s: 344.112.

0%        0.909 sec.  
10%       0.967 sec.  
20%       0.997 sec.  
30%       1.057 sec.  
40%       1.170 sec.  
50%       1.181 sec.  
60%       1.181 sec.  
70%       1.189 sec.  
80%       1.252 sec.  
90%       1.260 sec.  
95%       1.331 sec.  
99%       1.331 sec.  
99.9%     1.331 sec.  
99.99%    1.331 sec.
```

---

## 🧐 Шаг 7: Выводы

### 📊 Сравнение конфигураций ClickHouse

| № | Конфигурация                  | QPS   | RPS         | MiB/s | Среднее время запроса |
| - | ----------------------------- | ----- | ----------- | ----- | --------------------- |
| 1 | По умолчанию                  | 0.813 | \~16.27 млн | \~688 | 1.162–1.464 сек       |
| 2 | С ограничением ресурсов (80%) | 0.730 | \~14.62 млн | \~618 | 1.162–1.640 сек       |
| 3 | Оптимизация без ограничений   | 0.870 | \~17.41 млн | \~737 | 0.909–1.331 сек       |


**Вывод:** ClickHouse изначально оптимизирован и работает эффективно на настройках по умолчанию. Жесткое ограничение ресурсов заметно снижает производительность, а грамотная ручная настройка может улучшить показатели, особенно при интенсивной нагрузке и сложных запросах.



