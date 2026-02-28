# Домашнее задание 25 - Визуализация данных в Apache Superset

### Цель работы
- Развернуть Apache Superset
- Подключить Superset к ClickHouse
- Подготовить витрины данных
- Построить дашборд из 5 различных визуализаций
- Проверить корректность отображения данных

Визуализации строятся на основе:
* demo.binance_aggtrades_raw
* demo.weather_hourly

Данные уже загружены в ClickHouse.

---

## Шаг 1. Развертывание Apache Superset

Superset развернут в Docker в сети infra-net.

Создаю рабочую папку infra/superset, в которой создаю файл

`docker-compose.yaml` (superset):

```yml
version: "3.8"

services:
  superset:
    image: apache/superset:latest
    container_name: superset
    ports:
      - "8088:8088"
    environment:
      - SUPERSET_SECRET_KEY=XA8L/n8Q+eO6m9jMILzGsir/Kw0MfjDphOjRtySFkczFyKu3m9s86CNX
    volumes:
      - ./superset_home:/app/superset_home
    networks:
      - infra-net

networks:
  infra-net:
    external: true
```
И дополинтельно папку для тома данных mkdir infra/superset/superset_home

Выполняю миграцию db:
```bash
docker compose run --rm superset superset db upgrade
```

Создаю админа:
```bash
docker compose run --rm superset superset fab create-admin \
  --username admin \
  --firstname Admin \
  --lastname User \
  --email admin@example.com \
  --password admin
```

Инициализация:
```bash
docker compose run --rm superset superset init
```
Или все одной командой:
```bash
sudo docker exec -it superset superset fab create-admin \
               --username admin \
               --firstname Superset \
               --lastname Admin \
               --email admin@admin.com \
               --password admin; \
sudo docker exec -it superset superset db upgrade; \
sudo docker exec -it superset superset load_examples; \
sudo docker exec -it superset superset init;
```

Запуск:
```bash
docker compose up -d
```

Superset доступен по адресу:

http://<SERVER_IP>:8088

![📸 Скриншот 1: Страница логина Superset](https://github.com/realexpert1C/clickhouse-course/blob/e68c0a695a816bd4183230124d07f20e1cd5a422/images/hw25_superset_start.png)

---

## Шаг 2. Подключение Superset к ClickHouse

В Superset:

Settings → Database Connections → Add Database

Connection string:

clickhouse+http://default:default123@ch1:8123/demo

или 

clickhouse+http://default:default123@172.21.0.3:8123/demo

С первого раза подключение не успешно, потому что в контейнере Superset не установлен ClickHouse driver + SQLAlchemy dialect

Создаю /infra/superset/Dockerfile:
```bash
FROM apache/superset:3.1.3

USER root

RUN pip install --no-cache-dir apache-superset[clickhouse]

USER superset
```

И в docker-compose.yaml вместо image использую build:
```yml
superset:
  build: .
  container_name: superset
```

Пересобираю образ:
```bash
docker compose down
docker image rm apache/superset:latest
docker compose build --no-cache
docker compose up -d
```
И снова тестирую подключение - успешно!
![📸 Скриншот 2: Успешное подключение к ClickHouse](https://github.com/realexpert1C/clickhouse-course/blob/e68c0a695a816bd4183230124d07f20e1cd5a422/images/hw25_ch_connect.png)

---

## Шаг 3. Подготовка витрин

Для корректной аналитики созданы агрегированные витрины.

### 3.1 Витрина торгов по минутам
```sql
CREATE OR REPLACE VIEW demo.v_binance_minute AS
SELECT
    symbol,
    toStartOfMinute(event_time) AS minute,
    count() AS trades_count,
    sum(quantity) AS volume,
    sum(price * quantity) / sum(quantity) AS vwap,
    sumIf(quantity, is_buyer_maker = 0) AS buy_volume,
    sumIf(quantity, is_buyer_maker = 1) AS sell_volume
FROM demo.binance_aggtrades_raw
GROUP BY symbol, minute
ORDER BY minute;
```
---

### 3.2 Витрина погодных данных по часам
```sql
CREATE OR REPLACE VIEW demo.v_weather_hourly AS
SELECT
    location,
    event_time,
    temperature_2m,
    relative_humidity_2m,
    precipitation
FROM demo.weather_hourly
ORDER BY event_time;
```
![📸 Скриншот 3: Проверка витрин через SELECT](https://github.com/realexpert1C/clickhouse-course/blob/e68c0a695a816bd4183230124d07f20e1cd5a422/images/hw25_crypto_view.png)
![📸 Скриншот 3-1: Проверка витрин через SELECT](https://github.com/realexpert1C/clickhouse-course/blob/e68c0a695a816bd4183230124d07f20e1cd5a422/images/hw25_weth_view.png)

---

## Шаг 4. Создание датасетов в Superset

В Superset:

Data → Datasets → Add Dataset

Добавлены:
- demo.v_binance_minute
- demo.v_weather_hourly

![Скриншот 4: Список датасетов](https://github.com/realexpert1C/clickhouse-course/blob/2e3083e0833871f71150f73c6c78bbc76bd37f3e/images/hw25_datasets_list_v2.png)

---

## Шаг 5. Построение дашборда

Создан дашборд:

“BTC Market & Weather Overview”

---

__📊 Визуализация 1 — Volume per Minute__

Тип: Time-series Line Chart
Источник: v_binance_minute
Метрика: sum(volume)
Временная колонка: minute

---

__📈 Визуализация 2 — VWAP__

Тип: Time-series Line Chart
Метрика: avg(vwap)

---

__📉 Визуализация 3 — Buy/Sell Imbalance__

Тип: Stacked Area Chart

Метрики:
- sum(buy_volume)
- sum(sell_volume)

---

__🌡 Визуализация 4 — Temperature__

Тип: Time-series Line Chart
Источник: v_weather_hourly
Метрика: avg(temperature_2m)

---

__🌧 Визуализация 5 — Precipitation__

Тип: Bar Chart
Метрика: sum(precipitation)


![📸 Скриншот 5: Полный дашборд с 5 визуализациями]()

---

## Шаг 6. Связь временных шкал

Обе витрины используют:
* UTC timezone
* единую временную ось

Это позволяет:
- анализировать корреляцию погодных условий и рыночной активности
- наблюдать поведение объема при изменении погодных параметров

---

Итог

В рамках задания:

- ✔ Развернут Apache Superset
- ✔ Подключен к ClickHouse
- ✔ Подготовлены аналитические витрины
- ✔ Создан дашборд из 5 визуализаций
- ✔ Данные корректно отображаются