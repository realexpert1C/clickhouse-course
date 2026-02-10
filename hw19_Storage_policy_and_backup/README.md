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
* Доступ: `private`

![✅ Скриншот бакета в MINIO](https://github.com/realexpert1C/clickhouse-course/blob/10ffe52d98d8f6be18a92dc2e1172ae8d77fa2c3/images/hw19_click_backet.png.jpeg)


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

![✅ Скриншот содержимого config.yml](https://github.com/realexpert1C/clickhouse-course/blob/7f995b6bdbc2bdc982a155323368206b7a6c6000/images/hw19_yaml_cfg.png)

---

Этап 2.1. Настройка Storage Policy в ClickHouse

Для выполнения требований задания была добавлена пользовательская политика хранения.

Конфигурация storage policy

Файл `/etc/clickhouse-server/config.d/storage_policy.xml`:

```xml
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
```

Применение конфигурации
```bash
docker exec -it ch1 clickhouse-client
```

```sql
SYSTEM RELOAD CONFIG;`
```
Проверка

```sql
SELECT policy_name, volume_name, disks
FROM system.storage_policies
ORDER BY policy_name, volume_name;
```

![✅ Скриншот вывода SELECT](https://github.com/realexpert1C/clickhouse-course/blob/3030ba68169c596fdfec247039d50c9dafd090c7/images/hw19_screen_policy.png)

---

Этап 3. Создание тестовой базы данных и таблиц

Создание базы и таблиц

```sql
CREATE DATABASE IF NOT EXISTS hw19;

CREATE TABLE hw19.t1
(
  id UInt32,
  message String,
  ts DateTime
)
ENGINE = MergeTree
ORDER BY id
SETTINGS storage_policy = 'local_only';

CREATE TABLE hw19.uk_price_paid_daily_copy
AS default.uk_price_paid_daily
ENGINE = MergeTree
ORDER BY tuple()
SETTINGS storage_policy = 'local_only';

```


Заполнение данными

```sql

INSERT INTO hw19.t1 VALUES
(1,'Hello',now()),
(2,'World',now());

INSERT INTO hw19.uk_price_paid_daily_copy
SELECT * FROM default.uk_price_paid_daily
LIMIT 10000;

```
Проверка наличия созданных таблиц
```sql
SELECT database, name, engine, storage_policy
FROM system.tables
WHERE database='hw19';
```
![✅ Скриншот вывода SELECT](https://github.com/realexpert1C/clickhouse-course/blob/28479b4254130d57eeb506323a0b6ee64d25bba0/images/hw19_check_tables.png)


Фиксация исходного состояния данных (baseline)

```sql
SELECT count(), min(ts), max(ts) FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;
```

![✅ Скриншот выводов SELECT ... FROM hw19.t1 и SELECT ... FROM hw19.uk_price_paid_daily_copy](https://github.com/realexpert1C/clickhouse-course/blob/d424555a08cde45962270a5e4e56b982c9db1302/images/hw19_baseline.png)

---

Этап 4. Резервное копирование данных в S3

Проверка удалённых бэкапов

```bash
clickhouse-backup list remote
```
Полный бэкап

```bash
clickhouse-backup create_remote full_backup --rbac
```

Бэкап одной таблицы

```bash
clickhouse-backup create_remote t1_backup -t hw19.t1
```

Бэкап базы данных

```bash
clickhouse-backup create_remote hw19_db_backup -t 'hw19.*'
```

![✅ Скриншот бакета в MINIO](https://github.com/realexpert1C/clickhouse-course/blob/5145c196e29f7014d17d6503832e36a03345197e/images/hw19_backups.png)
---

Этап 5. Повреждение данных

Изменение данных в таблице

```sql
ALTER TABLE hw19.t1
UPDATE message = 'CORRUPTED'
WHERE id = 2;
```

Удаление таблицы

```sql
DROP TABLE hw19.uk_price_paid_daily_copy;
```

Проверяю контрольные запросы

```sql
SELECT * FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;
```

✅ ![Скриншот выводов SELECT ... FROM hw19.t1 и SELECT ... FROM hw19.uk_price_paid_daily_copy](https://github.com/realexpert1C/clickhouse-course/blob/8dc52bf7dc4af908ce26526d53f9819b4876ba4d/images/hw19_changes.png)

---

Этап 6. Восстановление данных

Восстановление одной таблицы

```bash
clickhouse-backup restore_remote t1_backup -t hw19.t1
```

Восстановление базы данных

```bash
clickhouse-backup restore_remote hw19_db_backup -t 'hw19.*'
```

Полное восстановление

```bash
clickhouse-backup restore_remote full_backup --rbac
```

---

Этап 7. Проверка успешного восстановления

Сравнение с baseline

```sql
SELECT count(), min(ts), max(ts) FROM hw19.t1;

SELECT * FROM hw19.t1;

SELECT count(), sum(price)
FROM hw19.uk_price_paid_daily_copy;
```


✅ ![Скриншот выводов SELECT ... ](https://github.com/realexpert1C/clickhouse-course/blob/752ce739ca312f5ea3eb11662e0d2316e7d75a3c/images/hw19_base_compare.png)


Результаты совпадают с зафиксированными значениями до повреждения данных.

---

Выводы:

* Настроено резервное копирование ClickHouse в S3-совместимое хранилище.
* Проверены различные варианты бэкапов (полный, таблица, база данных).
* Смоделированы повреждения данных.
* Успешно выполнено восстановление данных из резервных копий.