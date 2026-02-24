# Домашнее задание 23 - Оркестраторы и DI Tools

### Цель работы
* Развернуть ETL-инструмент (Apache Airflow)
* Подключить его к ClickHouse
* Настроить пайплайн загрузки данных из внешнего API
* Реализовать регулярную загрузку данных

---

### Архитектура решения

Инфраструктура
- ClickHouse Cluster:
- 1 shard
- 4 replicas (ch1, ch2, ch3, ch4)
- Replication через ClickHouse Keeper
- Airflow 3.1.7
- Python 3.10
- Executor: LocalExecutor
- Docker Compose
- Сеть: infra-net

---

## Шаг 1. Установка Airflow в Docker контейнерах

### 1.1 Подготовка директории и поддерикторий

```bash
mkdir -p ~/infra/airflow
cd ~/infra/airflow

mkdir -p dags logs plugins
```
Права

```bash
sudo chown -R 50000:0 dags logs plugins
sudo chmod -R 775 dags logs plugins
```
---

### 1.2 Создание docker-compose.yaml

Создаю файл:
```bash
nano docker-compose.yaml
```
Содержимое:

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
      - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
      - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins

volumes:
  postgres-db-volume:

networks:
  infra-net:
    external: true
```
---
### 1.3 Создание окружения
Дополнительно в каталоге airflow где будет запускаться контейнер создаю файл окружения .env:

```bash
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
_AIRFLOW_WWW_USER_PASSWORD=admin123
_AIRFLOW_WWW_USER_FIRSTNAME=Admin
_AIRFLOW_WWW_USER_LASTNAME=User
_AIRFLOW_WWW_USER_EMAIL=admin@example.com
```

Генерация FERNET_KEY : docker run --rm apache/airflow:3.1.7-python3.10 \
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

Генерация WEBSERVER__SECRET_KEY, SECRET_KEY и JWT_SECRET: ```bash openssl rand -hex 32 ```

---

### 1.4. Установка зависимостей Python

Для получения возможности установки дополнительных библиотек (которых возможно нет в стандартном контейнере Airflow)

Создаю Dockerfile:
```bash
nano Dockerfile
```

```bash
FROM apache/airflow:3.1.7-python3.10

USER airflow

RUN pip install --no-cache-dir \
    clickhouse-connect==0.7.8 \
    requests==2.31.0
```

Изменяю image (если не сделал этого ранее) в compose на build для всех сервисов Airflow:
```yaml
build:
  context: .
  dockerfile: Dockerfile
```
Пересборка:
```bash
docker compose down
docker compose build
docker compose up -d
```
Полные пересборка и перезапуск при необходимости :

```bash
docker compose down -v
docker volume prune -f
docker compose up airflow-init
docker compose up -d --build
```
---

### 1.5 Запуск Airflow

Инициализация БД:
```bash
docker compose up airflow-init --build
```
Создание пользователя `admin`
```bash
docker compose exec airflow-apiserver airflow users create \
  --username admin \
  --firstname Admin \
  --lastname User \
  --role Admin \
  --email admin@example.com \
  --password admin123
```
Запуск сервисов:

```bash
docker compose up -d --build
```

---

### 1.6 Вход в Web UI

http://<IP_сервера>:8080

Логин:

- admin

Пароль:

- admin123


![📸 Скриншот №1]()

Airflow Web UI после входа

---

## Шаг 2. Настройка подключения к ClickHouse

### 2.1 Создание пользователя в ClickHouse

```sql
CREATE USER IF NOT EXISTS etl_user IDENTIFIED BY 'etl_password123' ON CLUSTER replicated_cluster;
GRANT INSERT, SELECT, ALTER, TRUNCATE ON etl.* TO etl_user ON CLUSTER replicated_cluster;
```

---

### 2.2 Создание БД и таблицы

```sql
CREATE DATABASE IF NOT EXISTS etl ON CLUSTER replicated_cluster;

CREATE TABLE IF NOT EXISTS etl.crypto_prices ON CLUSTER replicated_cluster
(
    id String,
    symbol String,
    name String,
    current_price Float64,
    market_cap UInt64,
    total_volume UInt64,
    load_ts DateTime DEFAULT now()
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/crypto_prices',
    '{replica}'
)
ORDER BY id;
```

![📸 Скриншот №2]()

Результат выполнения CREATE TABLE

---
### 2.3 подключение к Clickhouse

В Airflow → Admin → Connections

Conn Id: clickhouse_conn
Conn Type: HTTP
Host: ch1
Port: 8123
Schema: etl
Login: etl_user
Password: etl_password123

![📸 Скриншот №3]()

---

## Шаг 3. Создание DAG

Файл:

`dags/crypto_etl.py`

```python
from airflow import DAG
from airflow.providers.standard.operators.python import PythonOperator
from datetime import datetime
import requests
import clickhouse_connect


def truncate_table():
    client = clickhouse_connect.get_client(
        host="ch1",
        port=8123,
        username="etl_user",
        password="etl_password123",
        database="etl"
    )

    client.command("TRUNCATE TABLE crypto_prices")


def load_crypto():
    url = "https://api.coingecko.com/api/v3/coins/markets"
    params = {
        "vs_currency": "usd",
        "order": "market_cap_desc",
        "per_page": 10,
        "page": 1
    }

    response = requests.get(url, params=params)
    data = response.json()

    client = clickhouse_connect.get_client(
        host="ch1",
        port=8123,
        username="etl_user",
        password="etl_password123",
        database="etl"
    )

    rows = [
        (
            coin["id"],
            coin["symbol"],
            coin["name"],
            coin["current_price"],
            coin["market_cap"],
            coin["total_volume"]
        )
        for coin in data
    ]

    client.insert(
        "crypto_prices",
        rows,
        column_names=[
            "id",
            "symbol",
            "name",
            "current_price",
            "market_cap",
            "total_volume"
        ]
    )


with DAG(
    dag_id="crypto_etl",
    start_date=datetime(2025, 1, 1),
    schedule="@daily",
    catchup=False
) as dag:

    truncate_task = PythonOperator(
        task_id="truncate_table",
        python_callable=truncate_table
    )

    load_task = PythonOperator(
        task_id="load_crypto_data",
        python_callable=load_crypto
    )

    truncate_task >> load_task
```

![📸 Скриншот №4]()

DAG в статусе Success

---

## Шаг 4. Проверка загрузки данных

```sql
SELECT *
FROM etl.crypto_prices
ORDER BY load_ts DESC
LIMIT 10;
```

![📸 Скриншот №5]()

Данные успешно загружены в ClickHouse

---

Проверка репликации

```sql
SELECT
    hostName(),
    count()
FROM clusterAllReplicas('replicated_cluster', etl.crypto_prices)
GROUP BY hostName();
```

![📸 Скриншот №6]()

Данные присутствуют на всех 4 репликах

---

## Итоги

В ходе выполнения задания:
- Развернут Apache Airflow 3.1.7 (Python 3.10)
- Настроено подключение к ClickHouse (кластер 1 shard / 4 replicas)
- Создан DAG для загрузки данных из API
- Настроена регулярная загрузка (@daily)
- Проверена репликация данных
- Настроен полноценный ETL-пайплайн: API → Airflow → ClickHouse Cluster (ReplicatedMergeTree)
- Данные загружены в Clickhouse

Структура проекта
```txt
airflow/
│
├── docker-compose.yaml
├── Dockerfile
├── dags/
│   └── crypto_etl.py
├── logs/
└── plugins/
```
---