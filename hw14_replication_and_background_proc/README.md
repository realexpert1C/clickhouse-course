# Домашнее задание №14 — Репликация и удаление

### 📄 Текст задания

> Репликация и удаление  
>
> **Цель:**
> - преобразовать таблицу в реплицируемую;
> - настроить реплики в ClickHouse;
> - работать с данными в распределённой системе;
>
> **Описание / шаги:**
> 1. Взять демонстрационный dataset: https://clickhouse.com/docs/en/getting-started/example-datasets.
> 2. Конвертировать таблицу в реплицируемую, используя макрос `replica`.
> 3. Добавить 2 реплики.
> 4. Выполнить запросы и отдать результаты как 2 файла:
>    ```sql
>    SELECT getMacro('replica'), *
>    FROM remote('replica1,replica2,replica3', system.parts)
>    FORMAT JSONEachRow;
>
>    SELECT * FROM system.replicas
>    FORMAT JSONEachRow;
>    ```
> 5. Добавить или выбрать колонку с типом `Date` в таблице.
>    Добавить TTL на таблицу: хранить последние 7 дней.
> 6. Отправить результат запроса `SHOW CREATE TABLE <таблица>`.
>
> **Критерии оценки:**
> - Таблица реплицирована;
> - Добавлены реплики;
> - Выполнены запросы с результатами;
> - Настроен TTL с проверкой `SHOW CREATE TABLE`.

---

## ШАГ 1 Взять демонстрационный датасет, загрузить таблицу

### Цель шага
Подготовить таблицу для репликации

Для этого разверну ClickHouse LTS версии 25.3 на одной ноде `ch1`,  
проверю ее работоспособность и готовность к дальнейшим шагам (загрузка данных, репликация, добавление Keeper и реплик). Далее загружу тестовый датасет.

---

### 1.1 Подготовка структуры каталогов

В корне проекта создаю отдельную директорию для инфраструктуры ClickHouse-кластера, так удобнее - файлы конфигурации
всех реплик и кипера в одной папке на хосте. Это для учебных целей, в реальных задачах могут быть все каталоги на разных хостах:

```bash
mkdir -p infra/ch-cluster/ch1/data
```
Структура каталогов:
<pre>
infra/&nbsp;
&nbsp;└── ch-cluster/
    └── ch1/
        ├── docker-compose.yml
        └── data/
</pre>
---

### 1.2 Файл конфигурации для создания контейнера __docker-compose.yaml__ для ClickHouse (ch1)

Файл: infra/ch-cluster/ch1/docker-compose.yaml

```xml
version: "3.8"

services:
  ch1:
    image: clickhouse/clickhouse-server:25.3
    container_name: ch1
    hostname: ch1
    ports:
      - "8123:8123"   # HTTP интерфейс
      - "9000:9000"   # Native интерфейс
    volumes:
      - ./data:/var/lib/clickhouse
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
```

---

### 1.3 Запуск контейнера ClickHouse

Переход в каталог и запуск контейнера:

```bash
cd infra/ch-cluster/ch1
docker compose up -d
```

Проверка состояния контейнера:

```bash
docker ps
```

Ожидаемый результат:
- контейнер ch1 в статусе Up

---

### 1.4 Запуск ClickHouse-клиента внутри контейнера

Подключение к ClickHouse с помощью встроенного клиента:

```bash
docker exec -it ch1 clickhouse-client
```

Ожидаемый результат:
	•	успешное подключение к серверу
	•	отображение версии клиента и сервера

📸 Результат - успешное подключение clickhouse-client
![hw14_ch_client](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_ch_client.PNG)

---

### 1.5 Проверка работоспособности ClickHouse

Внутри clickhouse-client выполняются проверочные запросы.

1.5.1 Проверка версии и аптайма

```sql
SELECT
    version() AS version,
    uptime()  AS uptime_seconds;
```

Ожидаемый результат:
	•	версия ClickHouse 25.3.x
	•	uptime_seconds > 0

📸 Результат запроса SELECT version(), uptime()
![hw14_ch1_check1](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_ch1_check1.PNG)

---

1.5.2. Базовая проверка выполнения запросов

```sql
SHOW DATABASES;

SELECT 1;
```

Ожидаемый результат:
	•	список системных баз данных
	•	корректный ответ на простой запрос

📸 Результат 
![hw14_ch1_check2](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_ch1_check2.PNG)

---

### 1.6.Создание таблицы MergeTree и загрузка датасета `UK Price Paid`

Скачивание и загрузка UK Price Paid: Инструкция: [ClickHouse Dataset Docs](https://clickhouse.com/docs/getting-started/example-datasets/uk-price-paid)

Создаю таблицу на ch1:

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
Загрузка датасета:

```sql
INSERT INTO default.uk_price_paid
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

Скриншоты создания таблицы и загрузки датасета:
![hw14_create_tbl](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_create_tbl.PNG), ![hw14_insert_tbl1](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_insert_tbl1.PNG), ![hw14_insert_tbl2](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_insert_tbl2.PNG)

### Результат выполнения шага

На данном этапе:
* ClickHouse 25.3 LTS успешно запущен
* контейнер ch1 работает стабильно
* клиент подключается без ошибок
* сервер корректно отвечает на запросы
* тестовая таблица на движке MergeTree создана, данные в нее успешно загружены. Таблица для репликации готова
  

## ШАГ 2 — Конвертируйте таблицу в реплицируемую, используя макрос `replica`.

### Цель шага - сделать таблицу реплицируемой

Чтобы сделать тестовую таблицу реплицируемой, необходимо сначала установить координатор, который будет управлять репликацией. Традиционно используется Zookeeper. Но для него необходим отдельный контейнер в Docker. Попробую другой вариант - это Clickhouse Keeper - который может быть установлен совместно с Clickhouse-Server в одном контейнере. После установки Keeper изменю конфигурационные файлы ch1 и зарегистрирую таблицу в Clickhouse Keeper. Для этого, помимо изменений в конфигурации ch1, я фактически создам новую таблицу с той же структурой, что у тестовой, но с использованием движка ReplicatedMergeTree и перелью данные из первоначальной таблицы в реплицируемую.

### 2.1 Установка Clickhouse Keeper

__Монтирование конфигурации__
Clickhouse Keeper входит в состав дистрибутива Clickhouse, поэтому для его добавления нужно изменить файл конфигурации контейнера и пересоздать его. В файл __docker-compose.yaml__ создания контейнера Clickhouse с именем `ch1`, который уже запущен, добавляю строку для монитирования папки дополнительных конфигураций `config.d`:

```yaml
volumes:
  - ./config.d:/etc/clickhouse-server/config.d
```
📌 ClickHouse автоматически подхватывает все *.xml из config.d.

--- 

__Конфигурация Keeper__

В папке /ch1/config.d/
cоздаю файл `config.d/keeper_server.xml`:

```xml
<clickhouse>
    <keeper_server>
        <!-- Клиентский порт (аналог ZooKeeper 2181) -->
        <tcp_port>9181</tcp_port>
        <listen_host>0.0.0.0</listen_host>

        <!-- Уникальный ID ноды -->
        <server_id>1</server_id>

        <!-- Хранилища -->
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
        </coordination_settings>

        <!-- Raft-конфигурация -->
        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>ch1</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```

---

__Конфигурация ZooKeeper-клиента ClickHouse__

и файл `config.d/zookeeper.xml`:

```xml
<clickhouse>
    <zookeeper>
        <node>
            <host>127.0.0.1</host>
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```

⚠️ Даже если используется Clickhouse Keeper, ClickHouse по-прежнему читает <zookeeper>.


__Запускаю__

В папке /ch-cluster/ch1:

```bash
docker compose down
docker compose up -d
```

Проверка логов:

docker logs ch1 | grep -E "Keeper|9181|9234"

В логах должно быть:

```bash
Listening for Keeper (tcp): 127.0.0.1:9181
Listening for Keeper (tcp): [::1]:9181
```

Скриншот выполнения команд создания контейнера и проверки логов:
![hw14_create_keeper](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_create_keeper.PNG)

---

Проверка работы Keeper (SQL)

Захожу в контейнер ch1 и запускаю Clickhouse client

```bash
docker exec -it ch1 clickhouse-client
```

Выполняю запрос

```sql
SELECT *
FROM system.zookeeper_connection;
```

Ожидаемый результат
* connected = 1
* host = 127.0.0.1
* port = 9181
* session_uptime увеличивается
* enabled_feature_flags не пустой

Скриншот выполнения запроса:
![hw14_check_keeper](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_check_keeper.png)

Результат запроса показывает, что Keeper работает корректно.

### 2.2 Подготовка ClickHouse-ноды ch1 к репликации (макросы + реплицируемая таблица)

Определение макросов ноды (shard / replica)

Зачем это нужно?
- `ReplicatedMergeTree` обязательно использует макросы {shard} и {replica};
- именно они формируют путь в Keeper и уникальность реплики;
- без макросов таблица не может быть реплицируемой.

На хосте создаю файл `infra/ch-cluster/ch1/config.d/macros.xml`:

```xml
<clickhouse>
    <macros>
        <shard>01</shard>
        <replica>ch1</replica>
    </macros>
</clickhouse>
```

📌 Важно:
* `shard` — логическая группа (у нас один шард);
* `replica` — уникальное имя ноды (должно отличаться на других нодах).

Проверяю, что макросы подхватились внутри ch1

```sql
SELECT
    getMacro('shard')   AS shard,
    getMacro('replica') AS replica;
```
Ожидаемый результат
```text
┌─shard─┬─replica─┐
│ 01    │ ch1     │
└───────┴─────────┘
```

Скриншот выполнения запроса:
![hw14_check_repl1](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_check_repl1.PNG)

### 2.3 Конвертация таблицы `uk_price_paid` в реплицируемую

ClickHouse не умеет менять движок таблицы напрямую.
Поэтому используется безопасный паттерн:

Создать новую таблицу → перенести данные → переименовать

Создаю новую реплицируемую таблицу с такой же структурой как у `uk_price_paid`, выполняю в ch1:

```sql
CREATE TABLE uk_price_paid_r
(
    `price` UInt32,
    `date` Date,
    `postcode1` LowCardinality(String),
    `postcode2` LowCardinality(String),
    `type` Enum8(
        'terraced' = 1,
        'semi-detached' = 2,
        'detached' = 3,
        'flat' = 4,
        'other' = 0
    ),
    `is_new` UInt8,
    `duration` Enum8(
        'freehold' = 1,
        'leasehold' = 2,
        'unknown' = 0
    ),
    `addr1` String,
    `addr2` String,
    `street` LowCardinality(String),
    `locality` LowCardinality(String),
    `town` LowCardinality(String),
    `district` LowCardinality(String),
    `county` LowCardinality(String)
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/default/uk_price_paid',
    '{replica}'
)
ORDER BY (postcode1, postcode2, addr1, addr2);
```
📌 Что важно:
* путь в Keeper одинаковый для всех реплик;
* {replica} подставляется из макросов;
* Keeper фиксирует таблицу, даже если реплика пока одна.

Переношу данные из старой таблицы:

```sql
INSERT INTO uk_price_paid_r
SELECT * FROM uk_price_paid;
```
📌 На этом этапе:
* данные физически копируются;
* Keeper фиксирует появление первой реплики.

Проверяю полноту данных, сравнивая длину таблиц:

```sql
SELECT count() FROM uk_price_paid;

SELECT count() FROM uk_price_paid_r;
```

Замена таблицы:

```sql
DROP TABLE uk_price_paid;

RENAME TABLE uk_price_paid__r TO uk_price_paid;
```
📌 Теперь:
* таблица называется как раньше;
* но движок — `ReplicatedMergeTree`.


Проверяю результат
```sql
SHOW CREATE TABLE uk_price_paid;
```
А также проверяю, что таблица видна Киперу как реплицируемая:

```sql
SELECT
    database,
    table,
    replica_name,
    is_readonly,
    total_replicas,
    active_replicas
FROM system.replicas;
```

Ожидаемый результат

```text
┌─database─┬─table──────────┬─replica_name─┬─is_readonly─┬─total_replicas─┬─active_replicas─┐
│ default  │ uk_price_paid  │ ch1          │ 0           │ 1              │ 1               │
└──────────┴────────────────┴──────────────┴─────────────┴────────────────┴─────────────────┘
```

Скриншоты выполнения запросов:
[hw14_change_tbl1](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_change_tbl1.PNG), [hw14_change_tbl2](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_change_tbl2.PNG), [hw14_change_tbl3](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_change_tbl3.PNG), [hw14_change_tbl4](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_change_tbl4.PNG)

### Результат выполнения шага

* ✅ Keeper запущен;
* ✅ макросы {shard} и {replica} настроены;
* ✅ таблица `uk_price_paid` реплицируемая, но реплика пока одна;
* ✅ Таблица была сконвертирована в ReplicatedMergeTree с использованием макроса {replica}.


---

## ШАГ 3 - Добавить две реплики

### 3.1. Поднимаю ClickHouse-ноду ch2

3.1.1. `docker-compose` для `ch2`

Создаю каталог:

`infra/ch-cluster/ch2`

и в нем файл
__docker-compose.yaml__ (аналогичен ch1, но с другими портами):

```yaml
services:
  ch2:
    image: clickhouse/clickhouse-server:25.3
    container_name: ch2
    hostname: ch2

    ports:
      - "8124:8123"
      - "9009:9000"

    volumes:
      - ./data:/var/lib/clickhouse
      - ./config.d:/etc/clickhouse-server/config.d

    networks:
      - infra-net

networks:
  infra-net:
    external: true
```
---

3.1.2. Конфигурация Keeper и ZooKeeper

Копирую без изменений из ch1:

```bash
cp -r ../ch1/config.d infra/ch-cluster/ch2/
```
❗ Важно:
* keeper_server.xml нужен только на ch1
* на ch2 и ch3 Keeper НЕ запускается

Поэтому на ch2 удаляю файл `keeper_server.xml`:
```bash
rm infra/ch-cluster/ch2/config.d/keeper_server.xml
```


3.1.3. Макросы для `ch2`

Редактирую config.d/macros.xml:
```xml
<clickhouse>
  <macros>
    <shard>01</shard>
    <replica>ch2</replica>
  </macros>
</clickhouse>
```
---
3.1.4. Запуск `ch2`

```bash
cd infra/ch-cluster/ch2
docker compose up -d
```
---

3.2. Создаю реплицируемую таблицу на `ch2`

Подключаюсь к ch2 через Clickhouse-client и выполняю `CREATE TABLE` БЕЗ дальнейшего копирования данных - они подтянутся автоматически:

```sql
CREATE TABLE uk_price_paid
(
    price UInt32,
    date Date,
    postcode1 LowCardinality(String),
    postcode2 LowCardinality(String),
    type Enum8(
        'terraced' = 1,
        'semi-detached' = 2,
        'detached' = 3,
        'flat' = 4,
        'other' = 0
    ),
    is_new UInt8,
    duration Enum8(
        'freehold' = 1,
        'leasehold' = 2,
        'unknown' = 0
    ),
    addr1 String,
    addr2 String,
    street LowCardinality(String),
    locality LowCardinality(String),
    town LowCardinality(String),
    district LowCardinality(String),
    county LowCardinality(String)
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/default/uk_price_paid',
    '{replica}'
)
ORDER BY (postcode1, postcode2, addr1, addr2);
```

📌 Данные подтянутся автоматически из ch1.

Контроль:

```sql
SELECT count() FROM uk_price_paid;
```

---

3.3. Повторяю такие же шаги для `ch3`

Полностью аналогично ch2, только:

Макросы:

<replica>ch3</replica>

Порты:

- "8125:8123"
- "9010:9000"


---

3.4. Проверка репликации

На любой ноде:

```sql
SELECT
    database,
    table,
    replica_name,
    is_readonly,
    total_replicas,
    active_replicas
FROM system.replicas;
```

Ожидаемо:
- total_replicas = 3
- active_replicas = 3
- replica_name: ch1, ch2, ch3

📌 Скриншот выполнения запроса
![hw14_check_repls](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_check_repls.PNG)

### Результат выполнения шага

* Добавлены две реплики на нодах `ch2` и `ch3`
* На каждой реплике создана реплицируемая таблица   `uk_price_paid` 


## ШАГ 4 - Выполните запросы и отдайте результаты как 2 файла

При выполнении запросов по заданию будут созданы файлы на хосте. Я их заберу для дальнейшей публикации с помощью команды:
```bash
scp admin@185.207.65.197:/home/admin/infra/ch-cluster/ch1/parts.json ./Downloads
```
```bash
scp admin@185.207.65.197:/home/admin/infra/ch-cluster/ch1/replicas.json ./Downloads
```


### 4.1 Запрос 1

```sql
SELECT
getMacro(‘replica’),
*
FROM remote(’разделенный запятыми список реплик’,system.parts)
FORMAT JSONEachRow;
```

Выполняю запрос командой:
```bash
docker exec -i ch1 clickhouse-client --query "
SELECT 
getMacro('replica'),
*
FROM remote('ch1,ch2,ch3',system.parts)

FORMAT JSONEachRow
" > parts.json
```
[Ссылка на parts.json](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/files/parts.json)

---

### 4.2 Запрос 2

```sql
SELECT * FROM system.replicas FORMAT JSONEachRow;
```

Выполняю запрос командой:
```bash
docker exec -i ch1 clickhouse-client --query "
SELECT *
FROM system.replicas
FORMAT JSONEachRow
" > replicas.json
```
[Ссылка на replicas.json](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/files/replicas.json)


### Результат выполнения шага
* выполнены запросы, 
* представлены ссылки для скачивания двух файлов

---

## ШАГ 5 - Добавьте или выберите колонку с типом Date в таблице, добавьте TTL на таблицу «хранить последние 7 дней». На проверку отправьте результат запроса «SHOW CREATE TABLE таблица»

### 5.1 — Проверяю, что колонка с типом Date есть

```sql
DESCRIBE TABLE uk_price_paid;
```

Вижу колнку `date` с типом `Date`

---

### 5.2 — Добавляю TTL «7 дней»

```sql
ALTER TABLE uk_price_paid
MODIFY TTL date + INTERVAL 7 DAY;
```

📌 Что это означает:
- каждая строка живёт 7 дней от значения в колонке date
- после этого попадает под удаление
- удаление происходит фоновыми мерджами, не мгновенно

Почему не ON CLUSTER?

Таблица использует движок `ReplicatedMergeTree`, поэтому изменения метаданных, включая TTL, распространяются на все реплики через ClickHouse Keeper без необходимости выполнения ALTER ON CLUSTER.

---

### 5.3 — Проверяю запросом

```sql
SHOW CREATE TABLE uk_price_paid;
```

📸 Скриншот выполнения запросов
![hw14_step5](https://github.com/realexpert1C/clickhouse-course/blob/d7cf08dfd89935b1b874fad83f8e296875ead635/images/hw14_step5.png)

### Результат выполнения шага
* выбрана колонка с типом `Date` в таблице,
* добавлен TTL на таблицу «хранить последние 7 дней» 
* на проверку представлен результат запроса «SHOW CREATE TABLE таблица»
---
