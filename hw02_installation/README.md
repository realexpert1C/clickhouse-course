Домашнее задание: Развертывание и базовая конфигурация, интерфейсы и инструменты 

⸻

Подготовка к работе. Базовая инфраструктура

Установленные компоненты
	•	Ubuntu 22.04 LTS (облачный сервер)
	•	Docker, Docker Compose
	•	Git

Созданные контейнеры
	•	ClickHouse
	•	MariaDB
	•	PostgreSQL 15 (внешняя)
	•	PostgreSQL 13 (внутренняя БД Airflow)
	•	Airflow 3.0

⸻

Конфигурация сетей Docker

Для корректной работы всех контейнеров и взаимодействия между ними используется кастомная bridge-сеть infra-net:

docker network create infra-net

Подключение контейнеров

Каждому контейнеру в docker-compose.yaml добавляю сеть:

networks:
  - infra-net

Также в корне docker-compose.yaml:

networks:
  infra-net:
    external: true


⸻

Дальнейшие действия
	•	Подключение Portainer для визуального управления контейнерами
	•	Проверка подключения к ClickHouse через WEB и через clickhouse-client


Шаг 1: Установка и запуск ClickHouse

ClickHouse запускается с пробросом трёх портов:

8123 — HTTP-интерфейс ClickHouse. Этот порт используется для подключения через браузер, утилиты типа clickhouse-client, а также большинством BI-систем.

9000 — нативный порт ClickHouse (TCP-протокол). Используется для подключения через драйверы ClickHouse, например, в DBeaver, Python (через clickhouse-driver) и других приложениях.

9009 — внутренний порт для взаимодействия между нодами (например, при репликации и шардинге), также используется мониторингом и профилированием.

Команда запуска контейнера ClickHouse с подключением к сети infra-net

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

Эти шаги обеспечивают доступ к ClickHouse через веб-интерфейс и инструменты анализа, а также интеграцию с другими контейнерами в проектной сети.

Установка clickhouse-client
sudo apt install clickhouse-client -y

Подключение к серверу
clickhouse-client --host 127.0.0.1 --port 9000 --user default --password my_pass


Шаг 2: Загрузка датасета

Для проверки загружен датасет UK Price Paid через HTTP по инструкции из документации.
https://clickhouse.com/docs/getting-started/example-datasets/uk-price-paid

Создана таблица через WEB интерфейс 

CREATE TABLE default.uk_price_paid
(
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
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2);

В таблицу загружены данные
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


К базе данных сделан ряд запросов. 
Все запросы отработали корректно.

Шаг 3. Скриншоты проверки подключени к Clickhouse и отработки SELECT

Фото подключения к Clickhouse. см. Скриншот1. Скриншот2. Скриншот3. Скриншот4.
Фото отработки запросов. см. Скриншот5. Скриншот6. Скриншот7. Скриншот8.


Шаг 4: Тестирование производительности

Тест 1. Конфигурация по умолчанию
Тестовый запрос. 
echo "SELECT * FROM default.uk_price_paid LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark --port 9000 --user default --password my_pass -i 10

- Результаты Теста 1.

Loaded 1 queries.

Queries executed: 1 (10%).

localhost:9000, queries: 1, QPS: 0.806, RPS: 16133829.266, MiB/s: 682.567, result RPS: 8064953.236, result MiB/s: 318.990.

0%		1.234 sec.	
10%		1.234 sec.	
20%		1.234 sec.	
30%		1.234 sec.	
40%		1.234 sec.	
50%		1.234 sec.	
60%		1.234 sec.	
70%		1.234 sec.	
80%		1.234 sec.	
90%		1.234 sec.	
95%		1.234 sec.	
99%		1.234 sec.	
99.9%		1.234 sec.	
99.99%		1.234 sec.	



Queries executed: 2 (20%).

localhost:9000, queries: 1, QPS: 0.974, RPS: 19484323.920, MiB/s: 824.778, result RPS: 9739793.242, result MiB/s: 385.443.

0%		1.023 sec.	
10%		1.023 sec.	
20%		1.023 sec.	
30%		1.023 sec.	
40%		1.023 sec.	
50%		1.023 sec.	
60%		1.023 sec.	
70%		1.023 sec.	
80%		1.023 sec.	
90%		1.023 sec.	
95%		1.023 sec.	
99%		1.023 sec.	
99.9%		1.023 sec.	
99.99%		1.023 sec.	



Queries executed: 3 (30%).

localhost:9000, queries: 1, QPS: 0.971, RPS: 19433603.527, MiB/s: 822.847, result RPS: 9714439.212, result MiB/s: 384.837.

0%		1.026 sec.	
10%		1.026 sec.	
20%		1.026 sec.	
30%		1.026 sec.	
40%		1.026 sec.	
50%		1.026 sec.	
60%		1.026 sec.	
70%		1.026 sec.	
80%		1.026 sec.	
90%		1.026 sec.	
95%		1.026 sec.	
99%		1.026 sec.	
99.9%		1.026 sec.	
99.99%		1.026 sec.	



Queries executed: 4 (40%).

localhost:9000, queries: 1, QPS: 0.814, RPS: 16278067.960, MiB/s: 689.103, result RPS: 8137055.048, result MiB/s: 321.418.

0%		1.225 sec.	
10%		1.225 sec.	
20%		1.225 sec.	
30%		1.225 sec.	
40%		1.225 sec.	
50%		1.225 sec.	
60%		1.225 sec.	
70%		1.225 sec.	
80%		1.225 sec.	
90%		1.225 sec.	
95%		1.225 sec.	
99%		1.225 sec.	
99.9%		1.225 sec.	
99.99%		1.225 sec.	



Queries executed: 5 (50%).

localhost:9000, queries: 1, QPS: 0.838, RPS: 16763776.421, MiB/s: 709.421, result RPS: 8379850.231, result MiB/s: 331.233.

0%		1.188 sec.	
10%		1.188 sec.	
20%		1.188 sec.	
30%		1.188 sec.	
40%		1.188 sec.	
50%		1.188 sec.	
60%		1.188 sec.	
70%		1.188 sec.	
80%		1.188 sec.	
90%		1.188 sec.	
95%		1.188 sec.	
99%		1.188 sec.	
99.9%		1.188 sec.	
99.99%		1.188 sec.	



Queries executed: 6 (60%).

localhost:9000, queries: 1, QPS: 0.680, RPS: 13610270.473, MiB/s: 575.334, result RPS: 6803480.630, result MiB/s: 270.422.

0%		1.464 sec.	
10%		1.464 sec.	
20%		1.464 sec.	
30%		1.464 sec.	
40%		1.464 sec.	
50%		1.464 sec.	
60%		1.464 sec.	
70%		1.464 sec.	
80%		1.464 sec.	
90%		1.464 sec.	
95%		1.464 sec.	
99%		1.464 sec.	
99.9%		1.464 sec.	
99.99%		1.464 sec.	



Queries executed: 7 (70%).

localhost:9000, queries: 1, QPS: 0.732, RPS: 14641099.283, MiB/s: 617.572, result RPS: 7318769.717, result MiB/s: 289.418.

0%		1.361 sec.	
10%		1.361 sec.	
20%		1.361 sec.	
30%		1.361 sec.	
40%		1.361 sec.	
50%		1.361 sec.	
60%		1.361 sec.	
70%		1.361 sec.	
80%		1.361 sec.	
90%		1.361 sec.	
95%		1.361 sec.	
99%		1.361 sec.	
99.9%		1.361 sec.	
99.99%		1.361 sec.	



Queries executed: 8 (80%).

localhost:9000, queries: 1, QPS: 0.825, RPS: 16512765.941, MiB/s: 698.157, result RPS: 8247620.698, result MiB/s: 324.560.

0%		1.209 sec.	
10%		1.209 sec.	
20%		1.209 sec.	
30%		1.209 sec.	
40%		1.209 sec.	
50%		1.209 sec.	
60%		1.209 sec.	
70%		1.209 sec.	
80%		1.209 sec.	
90%		1.209 sec.	
95%		1.209 sec.	
99%		1.209 sec.	
99.9%		1.209 sec.	
99.99%		1.209 sec.	



Queries executed: 10 (100%).

localhost:9000, queries: 10, QPS: 0.813, RPS: 16269366.674, MiB/s: 688.080, result RPS: 8131040.631, result MiB/s: 321.809.

0%		1.023 sec.	
10%		1.026 sec.	
20%		1.129 sec.	
30%		1.188 sec.	
40%		1.209 sec.	
50%		1.225 sec.	
60%		1.225 sec.	
70%		1.234 sec.	
80%		1.278 sec.	
90%		1.361 sec.	
95%		1.464 sec.	
99%		1.464 sec.	
99.9%		1.464 sec.	
99.99%		1.464 sec.



🛠 Шаг 5: Изучение конфигурационных файлов БД


Изучение конфигурации

⸻

📂 Основные конфигурационные директории и файлы:
	1.	/etc/clickhouse-server/config.xml — главный файл конфигурации сервера.
	2.	/etc/clickhouse-server/users.xml — пользователи, права, пароли.
	3.	/etc/clickhouse-server/config.d/ — дополнительные фрагменты конфигурации.
	4.	/etc/clickhouse-server/users.d/ — дополнительные фрагменты пользовательской конфигурации.

⸻
Просмотр содержимого:
docker exec -it clickhouse cat /etc/clickhouse-server/config.xml
docker exec -it clickhouse ls /etc/clickhouse-server/config.d/
docker exec -it clickhouse cat /etc/clickhouse-server/config.d/<имя_файла>


Ключевые параметры конфигурации ClickHouse, которые можно изменить для повышения производительности:
	1.	max_threads — количество потоков обработки запроса. Лучше всего выставить в значение, равное количеству физических ядер CPU.
	2.	max_memory_usage — максимальный объём памяти, доступный одному запросу.
	3.	max_memory_usage_for_all_queries — общий лимит памяти на все запросы.
	4.	max_network_bandwidth — ограничение пропускной способности по сети.
	5.	max_concurrent_queries — ограничение одновременных запросов.
	6.	merge_tree_max_rows_to_use_cache / merge_tree_max_bytes_to_use_cache — контроль использования кеша для MergeTree таблиц.
	7.	read_backoff_min_latency_ms / read_backoff_min_rows — параметры для адаптивного управления нагрузкой при чтении с диска.
	8.	compression / min_compress_block_size / max_compress_block_size — настройки компрессии.
	9.	storage_configuration — оптимизация для дисков, включая SSD/HDD, S3 и другие типы хранилищ.
	10.	background_pool_size / background_merges_mutations_concurrency — управление параллелизмом фоновых операций.

Дополнительно стоит использовать профили настроек (<profiles>) в users.xml, чтобы задать разные режимы работы (например, production, test и пр.) .


🛠 Шаг 6: Оптимизация и повторное тестирование

⸻

Создание кастомного конфигурационного файла
nano /etc/clickhouse-server/config.d/performance_tuning.xml

конфиг с ограничением 80%
<clickhouse>
    <max_server_memory_usage>10307921510</max_server_memory_usage> 
    <max_memory_usage>1030792151</max_memory_usage> 
    <max_threads>6</max_threads> 
    <max_concurrent_queries>10</max_concurrent_queries>
    <background_pool_size>6</background_pool_size>
    <background_buffer_flush_schedule_pool_size>2<background_buffer_flush_schedule_pool_size>
    <background_fetches_pool_size>3</background_fetches_pool_size>
<clickhouse>

Перезапускаю clickhouse: docker restart clickhouse

- Результаты Теста 2.

Loaded 1 queries.

Queries executed: 1 (10%).

localhost:9000, queries: 1, QPS: 0.607, RPS: 12176151.301, MiB/s: 515.519, result RPS: 6066720.793, result MiB/s: 240.574.

0%		1.640 sec.	
10%		1.640 sec.	
20%		1.640 sec.	
30%		1.640 sec.	
40%		1.640 sec.	
50%		1.640 sec.	
60%		1.640 sec.	
70%		1.640 sec.	
80%		1.640 sec.	
90%		1.640 sec.	
95%		1.640 sec.	
99%		1.640 sec.	
99.9%		1.640 sec.	
99.99%		1.640 sec.	



Queries executed: 2 (20%).

localhost:9000, queries: 1, QPS: 0.809, RPS: 16177291.053, MiB/s: 684.424, result RPS: 8086678.846, result MiB/s: 320.020.

0%		1.226 sec.	
10%		1.226 sec.	
20%		1.226 sec.	
30%		1.226 sec.	
40%		1.226 sec.	
50%		1.226 sec.	
60%		1.226 sec.	
70%		1.226 sec.	
80%		1.226 sec.	
90%		1.226 sec.	
95%		1.226 sec.	
99%		1.226 sec.	
99.9%		1.226 sec.	
99.99%		1.226 sec.	



Queries executed: 3 (30%).

localhost:9000, queries: 1, QPS: 0.680, RPS: 13611338.050, MiB/s: 575.413, result RPS: 6795665.805, result MiB/s: 268.499.

0%		1.465 sec.	
10%		1.465 sec.	
20%		1.465 sec.	
30%		1.465 sec.	
40%		1.465 sec.	
50%		1.465 sec.	
60%		1.465 sec.	
70%		1.465 sec.	
80%		1.465 sec.	
90%		1.465 sec.	
95%		1.465 sec.	
99%		1.465 sec.	
99.9%		1.465 sec.	
99.99%		1.465 sec.	



Queries executed: 4 (40%).

localhost:9000, queries: 1, QPS: 0.656, RPS: 13163513.528, MiB/s: 555.760, result RPS: 6564028.636, result MiB/s: 259.183.

0%		1.513 sec.	
10%		1.513 sec.	
20%		1.513 sec.	
30%		1.513 sec.	
40%		1.513 sec.	
50%		1.513 sec.	
60%		1.513 sec.	
70%		1.513 sec.	
80%		1.513 sec.	
90%		1.513 sec.	
95%		1.513 sec.	
99%		1.513 sec.	
99.9%		1.513 sec.	
99.99%		1.513 sec.	



Queries executed: 5 (50%).

localhost:9000, queries: 1, QPS: 0.844, RPS: 16900126.032, MiB/s: 714.452, result RPS: 8441095.196, result MiB/s: 334.974.

0%		1.178 sec.	
10%		1.178 sec.	
20%		1.178 sec.	
30%		1.178 sec.	
40%		1.178 sec.	
50%		1.178 sec.	
60%		1.178 sec.	
70%		1.178 sec.	
80%		1.178 sec.	
90%		1.178 sec.	
95%		1.178 sec.	
99%		1.178 sec.	
99.9%		1.178 sec.	
99.99%		1.178 sec.	



Queries executed: 6 (60%).

localhost:9000, queries: 1, QPS: 0.685, RPS: 13693500.027, MiB/s: 579.281, result RPS: 6845085.289, result MiB/s: 271.479.

0%		1.453 sec.	
10%		1.453 sec.	
20%		1.453 sec.	
30%		1.453 sec.	
40%		1.453 sec.	
50%		1.453 sec.	
60%		1.453 sec.	
70%		1.453 sec.	
80%		1.453 sec.	
90%		1.453 sec.	
95%		1.453 sec.	
99%		1.453 sec.	
99.9%		1.453 sec.	
99.99%		1.453 sec.	



Queries executed: 7 (70%).

localhost:9000, queries: 1, QPS: 0.724, RPS: 14478975.727, MiB/s: 612.754, result RPS: 7237727.648, result MiB/s: 286.159.

0%		1.376 sec.	
10%		1.376 sec.	
20%		1.376 sec.	
30%		1.376 sec.	
40%		1.376 sec.	
50%		1.376 sec.	
60%		1.376 sec.	
70%		1.376 sec.	
80%		1.376 sec.	
90%		1.376 sec.	
95%		1.376 sec.	
99%		1.376 sec.	
99.9%		1.376 sec.	
99.99%		1.376 sec.	



Queries executed: 8 (80%).

localhost:9000, queries: 1, QPS: 0.814, RPS: 16328907.600, MiB/s: 691.849, result RPS: 8142462.637, result MiB/s: 322.579.

0%		1.220 sec.	
10%		1.220 sec.	
20%		1.220 sec.	
30%		1.220 sec.	
40%		1.220 sec.	
50%		1.220 sec.	
60%		1.220 sec.	
70%		1.220 sec.	
80%		1.220 sec.	
90%		1.220 sec.	
95%		1.220 sec.	
99%		1.220 sec.	
99.9%		1.220 sec.	
99.99%		1.220 sec.	



Queries executed: 10 (100%).

localhost:9000, queries: 10, QPS: 0.730, RPS: 14619446.917, MiB/s: 618.571, result RPS: 7300472.300, result MiB/s: 289.173.

0%		1.162 sec.	
10%		1.178 sec.	
20%		1.220 sec.	
30%		1.226 sec.	
40%		1.275 sec.	
50%		1.376 sec.	
60%		1.376 sec.	
70%		1.453 sec.	
80%		1.465 sec.	
90%		1.513 sec.	
95%		1.640 sec.	
99%		1.640 sec.	
99.9%		1.640 sec.	
99.99%		1.640 sec.

Вижу, что результат хуже, чем по умолчанию. Видимо слишком большое ограничение по производительности поставил
в 80%, уберу. Сделаю еще одно тестирование с кастомным конфигом, но с ограничеием производительности по умолчанию.

Напоминаю запрос, который используется echo "SELECT * FROM default.uk_price_paid LIMIT 10000000 OFFSET 10000000" | clickhouse-benchmark --port 9000 --user default --password my_pass -i 10

Кастомный конфиг 2
<clickhouse>
    <!-- Увеличение числа фонов задач для слияния и сжатия данных -->
    <merge_tree>
        <number_of_free_entries_in_pool_to_execute_mutation>0</number_of_free_entries_in_pool_to_execute_mutation>
    </merge_tree>

    <!-- Расширение пула фоновых потоков -->
    <background_pool_size>16</background_pool_size>
    <background_buffer_flush_schedule_pool_size>4</background_buffer_flush_schedule_pool_size>
    <background_fetches_pool_size>6</background_fetches_pool_size>

    <!-- Увеличение размера кэша запросов -->
    <query_cache>
        <size_in_bytes>1073741824</size_in_bytes> <!-- 1 GB -->
        <max_entries>4096</max_entries>
    </query_cache>

    <!-- Расширенные лимиты для параллельных потоков -->
    <max_threads>0</max_threads> <!-- Автоопределение -->
    <max_concurrent_queries>100</max_concurrent_queries>
</clickhouse>


- Результаты Теста 3.

Loaded 1 queries.

Queries executed: 1 (10%).

localhost:9000, queries: 1, QPS: 0.939, RPS: 18778666.697, MiB/s: 794.729, result RPS: 9387050.418, result MiB/s: 370.367.

0%		1.057 sec.	
10%		1.057 sec.	
20%		1.057 sec.	
30%		1.057 sec.	
40%		1.057 sec.	
50%		1.057 sec.	
60%		1.057 sec.	
70%		1.057 sec.	
80%		1.057 sec.	
90%		1.057 sec.	
95%		1.057 sec.	
99%		1.057 sec.	
99.9%		1.057 sec.	
99.99%		1.057 sec.	



Queries executed: 2 (20%).

localhost:9000, queries: 1, QPS: 0.843, RPS: 16870149.417, MiB/s: 714.227, result RPS: 8433023.797, result MiB/s: 333.206.

0%		1.170 sec.	
10%		1.170 sec.	
20%		1.170 sec.	
30%		1.170 sec.	
40%		1.170 sec.	
50%		1.170 sec.	
60%		1.170 sec.	
70%		1.170 sec.	
80%		1.170 sec.	
90%		1.170 sec.	
95%		1.170 sec.	
99%		1.170 sec.	
99.9%		1.170 sec.	
99.99%		1.170 sec.	



Queries executed: 3 (30%).

localhost:9000, queries: 1, QPS: 0.790, RPS: 15816785.593, MiB/s: 669.726, result RPS: 7899999.837, result MiB/s: 312.581.

0%		1.252 sec.	
10%		1.252 sec.	
20%		1.252 sec.	
30%		1.252 sec.	
40%		1.252 sec.	
50%		1.252 sec.	
60%		1.252 sec.	
70%		1.252 sec.	
80%		1.252 sec.	
90%		1.252 sec.	
95%		1.252 sec.	
99%		1.252 sec.	
99.9%		1.252 sec.	
99.99%		1.252 sec.	



Queries executed: 5 (50%).

localhost:9000, queries: 2, QPS: 0.917, RPS: 18346301.877, MiB/s: 776.553, result RPS: 9167166.612, result MiB/s: 363.533.

0%		0.909 sec.	
10%		0.909 sec.	
20%		0.909 sec.	
30%		0.909 sec.	
40%		0.909 sec.	
50%		1.260 sec.	
60%		1.260 sec.	
70%		1.260 sec.	
80%		1.260 sec.	
90%		1.260 sec.	
95%		1.260 sec.	
99%		1.260 sec.	
99.9%		1.260 sec.	
99.99%		1.260 sec.	



Queries executed: 6 (60%).

localhost:9000, queries: 1, QPS: 0.748, RPS: 14954247.976, MiB/s: 632.932, result RPS: 7475305.994, result MiB/s: 296.008.

0%		1.331 sec.	
10%		1.331 sec.	
20%		1.331 sec.	
30%		1.331 sec.	
40%		1.331 sec.	
50%		1.331 sec.	
60%		1.331 sec.	
70%		1.331 sec.	
80%		1.331 sec.	
90%		1.331 sec.	
95%		1.331 sec.	
99%		1.331 sec.	
99.9%		1.331 sec.	
99.99%		1.331 sec.	



Queries executed: 7 (70%).

localhost:9000, queries: 1, QPS: 0.843, RPS: 16857927.169, MiB/s: 713.493, result RPS: 8426914.159, result MiB/s: 333.066.

0%		1.181 sec.	
10%		1.181 sec.	
20%		1.181 sec.	
30%		1.181 sec.	
40%		1.181 sec.	
50%		1.181 sec.	
60%		1.181 sec.	
70%		1.181 sec.	
80%		1.181 sec.	
90%		1.181 sec.	
95%		1.181 sec.	
99%		1.181 sec.	
99.9%		1.181 sec.	
99.99%		1.181 sec.	



Queries executed: 10 (100%).

localhost:9000, queries: 10, QPS: 0.870, RPS: 17412431.214, MiB/s: 736.929, result RPS: 8699823.673, result MiB/s: 344.112.

0%		0.909 sec.	
10%		0.967 sec.	
20%		0.997 sec.	
30%		1.057 sec.	
40%		1.170 sec.	
50%		1.181 sec.	
60%		1.181 sec.	
70%		1.189 sec.	
80%		1.252 sec.	
90%		1.260 sec.	
95%		1.331 sec.	
99%		1.331 sec.	
99.9%		1.331 sec.	
99.99%		1.331 sec.	


🧠 Шаг 6: Выводы

⸻

📊 Сравнение конфигураций ClickHouse по результатам clickhouse-benchmark

№	Конфигурация	QPS (запросов/сек)	RPS (строк/сек)	MiB/s	Сред. время запроса
1	По умолчанию	0.813	~16.27 млн	~688 MiB/s	1.162–1.464 сек
2	С ограничением ресурсов (80%)	0.730	~14.62 млн	~618 MiB/s	1.162–1.640 сек
3	Оптимизация без ограничений	0.870	~17.41 млн	~737 MiB/s	0.909–1.331 сек

Проведенные тесты показали различие в производительности, при чем ограничение потребления ресурсов системы снизило ее, а возвращение к значениям по умолчанию при одновременной подстройке конфигурации под параметры оборудования сервера повысило. Однако, я бы не назвал данные отклонения критическими. Очевидно, что и с настройками по умолчанию Clickhouse достаточно оптимально сконфигурирован.