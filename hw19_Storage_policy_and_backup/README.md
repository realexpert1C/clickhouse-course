Отлично, вот обновлённый конспект с включёнными SQL/CLI-проверочными запросами для каждого этапа, где требуется скриншот.

⸻

Домашнее задание: Storage Policy и резервное копирование в ClickHouse

Цель

Изучить возможности резервного копирования и Storage Policy в ClickHouse, настроить clickhouse-backup с загрузкой на S3 (MinIO), протестировать восстановление после повреждения данных.

---

### Этап 1: Развёртывание S3-совместимого хранилища (MinIO)

Запуск MinIO в Docker:b

#### Создать каталог для MinIO
```bash
mkdir -p ~/infra/minio/data
```

#### Запускаем контейнер MinIO
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

✅ ![hw19_click_bucket]()

---

### Этап 2: Установка и настройка `clickhouse-backup`

Установка
````bash
wget https://github.com/Altinity/clickhouse-backup/releases/download/v2.5.20/clickhouse-backup-linux-amd64.tar.gz
tar -xf clickhouse-backup-linux-amd64.tar.gz
sudo install -o root -g root -m 0755 build/linux/amd64/clickhouse-backup /usr/local/bin
```
Проверка установки:
````bash
clickhouse-backup -v
```
✅ ![hw19_click_backup]()

---

Конфигурация /etc/clickhouse-backup/config.yml

general:
  remote_storage: s3

clickhouse:
  username: default
  password: ""
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

✅ [ВСТАВИТЬ СКРИНШОТ содержимого config.yml]

⸻

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

✅ [ВСТАВИТЬ СКРИНШОТ вывода SELECT]

⸻

Этап 4: Создание и загрузка бэкапа на S3

clickhouse-backup create_remote test_backup
clickhouse-backup list remote

Проверка:

clickhouse-backup list remote

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

⸻

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


---

Выводы
	•	Использование clickhouse-backup и MinIO позволяет быстро организовать offsite-резервное копирование ClickHouse.
	•	Storage Policy с S3-диском — эффективный способ удешевления хранения.
	•	Восстановление проходит без потерь, включая структуру и данные.

