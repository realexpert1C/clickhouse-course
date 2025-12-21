## Домашнее задание: Исполняемые пользовательские функции в ClickHouse

---

#### Вариант 1 

##### Цель:
Цель этого домашнего задания - понять и применить агрегатные функции, функции,
работающие с типами данных, и функции, определяемые пользователем (UDF) в ClickHouse.

---

##### Набор данных:

Создаю таблицу
```sql
CREATE TABLE transactions
(
    transaction_id UInt32,
    user_id UInt32,
    price Float32,
    quantity UInt32,
    transaction_date DateTime
)
ENGINE = MergeTree
ORDER BY transaction_id;
```
Заполняю таблицу случайными реалистичными данными для получения тестового датасета в 100 строк

```sql
INSERT INTO transactions
SELECT
    number + 1                              AS transaction_id,
    randUniform(1, 50)                      AS user_id,
    round(randUniform(5, 500), 2)           AS price,
    randUniform(1, 10)                      AS quantity,
    now() - INTERVAL randUniform(0, 30) DAY AS transaction_date
FROM numbers(100);
```

📌 Что здесь происходит:
* `transaction_id` → 1…100
* `user_id` → 50 пользователей
* `price` → от 5 до 500
* `quantity` → от 1 до 10
* `transaction_date` → последние 30 дней

Проверяю реалистичность вставленных данных

```sql
SELECT
    min(price)      AS min_price,
    max(price)      AS max_price,
    avg(price)      AS avg_price,
    min(quantity)   AS min_qty,
    max(quantity)   AS max_qty
FROM transactions;
```
Результат проверки реалистичности ![hw06_check_tabl](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_check_tabl.png)

---
#### Задания
---
##### 1. Агрегатные функции

1.1 Рассчитайте общий доход от всех операций

```sql
SELECT
    sum(price * quantity) AS total_revenue
FROM transactions;
```

Результат выполнения запроса ![hw06_task_1_1](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_1_1.png)

1.2 Найдите средний доход с одной сделки

```sql
SELECT
    avg(price * quantity) AS avg_transaction_value
FROM transactions;
```

Результат выполнения запроса ![hw06_task_1_2](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_1_2.png)

1.3 Определите общее количество проданной продукции

```sql
SELECT sum(quantity) AS total_quantity
FROM transactions; 
```

Результат выполнения запроса ![hw06_task_1_3](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_1_3.png)


1.4 Подсчитайте количество уникальных пользователей, совершивших покупку

```sql
SELECT countDistinct(user_id) FROM transactions;
```

и альтернативные варианты

```sql
SELECT uniq(user_id) FROM transactions;
```

```sql
SELECT uniqExact(user_id) FROM transactions;
```

```sql
SELECT uniqCombined(user_id) FROM transactions;
```

Какой из них выбрать зависит от задач

|Функция|Точность|Скорость|Когда использовать|
|-------|--------|--------|------------------|
|countDistinct|100%|средняя|обучение, отчёты|
|uniqExact|100%|медленнее|контрольные расчёты|
|uniq|~99%|очень быстрая|большие данные|
|uniqCombined|~99.9%|быстрая|продакшен|


Результат выполнения запроса ![hw06_task_1_4](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_1_4.png)

##### Пояснения к заданию

При выполнении задания использованы стандартные функции для подсчета суммы, среднего и 
количества уникальных значений

---

##### 2. Функции для работы с типами данных

2.1 Преобразуйте `transaction_date` в строку формата `YYYY-MM-DD`

```sql
SELECT
    transaction_date,
    formatDateTime(transaction_date, '%Y-%m-%d') AS transaction_date_str
FROM transactions
LIMIT 5;
```

Результат выполнения запроса ![hw06_task_2_1](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_2_1.png)

2.2 Извлеките год и месяц из `transaction_date`


```sql
SELECT
    transaction_date,
    toYear(transaction_date)  AS year,
    toMonth(transaction_date) AS month
FROM transactions
LIMIT 5;
```

Результат выполнения запроса ![hw06_task_2_2](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_2_2.png)


2.3 Округлите `price`до ближайшего целого числа

```sql
SELECT
    price,
    round(price) AS rounded_price
FROM transactions
LIMIT 5;
```

Результат выполнения запроса ![hw06_task_2_3](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_2_3.png)

2.4 Преобразуйте `transaction_id` в строку

```sql
SELECT
    transaction_id,
    toString(transaction_id) AS transaction_id_str
FROM transactions
LIMIT 5;
```

Результат выполнения запроса ![hw06_task_2_4](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_2_4.png)


И можно добавить, что все указанные в задании 2 преобразования можно сделать одним запросом

```sql
SELECT
    transaction_id,
    toString(transaction_id)                          AS transaction_id_str,
    transaction_date,
    formatDateTime(transaction_date, '%Y-%m-%d')      AS transaction_date_str,
    toYear(transaction_date)                           AS year,
    toMonth(transaction_date)                          AS month,
    price,
    round(price)                                       AS rounded_price
FROM transactions
LIMIT 10;
```

Результат выполнения запроса ![hw06_task_2](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_2.png)

##### Пояснения к заданию

В данном шаге были использованы встроенные функции ClickHouse для преобразования типов данных:
`formatDateTime`, `toYear`, `toMonth`, `round`, `toString`.
Эти функции являются нативными и оптимизированными, что делает их предпочтительными по сравнению со строковыми преобразованиями.

---

##### 3. User-Defined Functions (UDFs)

3.1 Создайте простую UDF для расчета общей стоимости транзакции

```sql
CREATE FUNCTION total_price
AS (quantity, price)
-> round(quantity * price, 2);
```

Результат выполнения запроса ![hw06_task_3_1](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_3_1.png)

Результат показывает, что функция создалась успешно и может быть вызвана
в запросе по имени total_price(quantity, price) с указанием аргументов в скобках

3.2 Используйте созданную UDF для расчета общей цены для каждой транзакции

```sql
SELECT
    transaction_id,
    quantity,
    price,
    total_price(quantity, price) AS total_price
FROM transactions
LIMIT 10;
```

Результат выполнения запроса ![hw06_task_3_2](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_3_2.png) скорость выполнения запроса 0.002сек

3.3 Создайте UDF для классификации транзакций на «высокоценные» и «малоценные»
на основе порогового значения (например, 100)

Создаю функцию с помощью запроса также как в задании 3.1

Критерий:
- `total_price` >= 100 → `High Value`
- иначе → `Low Value`

```sql
CREATE FUNCTION transaction_category
AS (total)
-> if(total >= 100, 'High Value', 'Low Value');
```

Результат выполнения запроса ![hw06_task_3_3](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task_3_3.png) скорость выполнения запроса 0.002сек

Ok в выводе означает, что функция успешно создана

3.4 Примените UDF для категоризации каждой транзакции

```sql
SELECT
    transaction_id,
    quantity,
    price,
    total_price(quantity, price)                       AS total_price,
    transaction_category(total_price(quantity, price)) AS category
FROM transactions
LIMIT 10;
```

Результат выполнения запроса ![hw06_task3_4](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_task3_4.png)

##### Пояснения к заданию

- Видим что разрешено вложенное использование UDF
- Поведение UDF идентично обычным SQL-выражениям

--- 

#### Выводы

В данной работе были реализованы агрегатные функции, функции,
работающие с типами данных, и функции, определяемые пользователем (UDF) в ClickHouse
Определенные системно функции в Clickhouse обширны, позволяют выполнить подавляющее большинство
задач, подробно описаны в документации. Также возможны их комбинации. Но если и этого недостаточно, то существует возможность определения пользователем кастомных функций с помощью запросов (User defined functions - UDF). UDF в ClickHouse, создаваемые пользователем с помощью SQL запросов выполняются построчно и не требуют внешних скриптов или конфигурационных файлов, что делает их простыми и надёжными для аналитических задач.

---


#### Вариант 2

##### Цель:
Цель этого домашнего задания - понять и применить исполняемые пользовательские
функции (EUDF) в ClickHouse. EUDF позволяют расширить функциональность ClickHouse путем
написания пользовательских функций на внешних языках программирования, таких как Python.

Набор данных - использую ранее созданную в Варианте 1 таблицу `transactions`

---
##### 🔧 1. Настройка среды для EUDF


1.1 Установка необходимого программного обеспечения

Выполняю в контейнере с установленным Clickhouse

```bash
# Установка Python и зависимостей
sudo apt-get update
sudo apt-get install python3 python3-pip -y
pip3 install clickhouse-driver
```

---


⚙️ 1.2 Конфигурация ClickHouse для EUDF

Создаю файл:

`sudo nano /etc/clickhouse-server/config.d/udf.xml`

или на хосте 

`sudo nano ./config.d/udf.xml`

Вставляю содержимое `udf.xml`:


<clickhouse>
    <user_defined_executable_functions_config>
        <allow_functions>true</allow_functions>
        <execution_path>/var/lib/clickhouse/user_scripts</execution_path>
    </user_defined_executable_functions_config>
</clickhouse>

📁 1.3 Создание каталога сценариев для EUDF

Создаю каталог, где будут лежать Python-скрипты:

```bash
sudo mkdir -p /var/lib/clickhouse/user_scripts
sudo chown -R clickhouse:clickhouse /var/lib/clickhouse/user_scripts
```
Или на хосте в директории clickhouse с монтированием их на серевер через docker-compose.yml
```bash
sudo mkdir -p ./common/user_scripts
sudo chown -R 101:101 ./common/user_scripts
```
```yml
volumes:
- /home/admin/infra/clickhouse/user_scripts:/etc/clickhouse/user_scripts:ro
```


После этого перезапускаю ClickHouse:

```bash
sudo systemctl restart clickhouse-server
```
или если как у меня используется Docker
```bash
docker restart clickhouse 
```
---

##### 🔧 1. Создание и применение EUDF



2.1 Создаю простой скрипт Python UDF. В папке /etc/clickhouse/user_scripts
создаю файл `total_price.py` с содержимым:

```python
import sys
import json

def total price(quantity, price):
    return quantity * price
if __name__ == "__main__":
    data = json.load(sys.stdin)
    quantity = data['quantity']
    price = data['price']
    print(total_price(quantity, price))
```

2.2 Использую следующую команду SQL для регистрации EUDF в ClickHouse:

```sql
CREATE FUNCTION total_price AS '/etc/clickhouse/user_scripts/total_price.py'
RETURNS Float32
EXECUTE ON HOST;
```

Результат выполнения запроса ![hw06_var2_fail1](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_var2_fail1.png)

__Пояснение__: Что произошло и что означает полученная ошибка?

Буквально:

`failed at position 78 ('RETURNS')
Expected one of: token, ..., end of query`

Перевод на человеческий язык:

«После AS '/path' я ожидал конец запроса или логическое выражение.
Ключевое слово RETURNS в этом месте для меня не существует».

Это означает, что в Clickhouse не поддержки оператора `RETURNS`

Изученная мной документация и статьи по данной теме привели меня к выводу, что в ClickHouse 24.x (а также мной это было проверено и на 23.3 и на 25.6) исполняемые пользовательские функции (Executable UDF) не регистрируются через CREATE FUNCTION и не поддерживают синтаксис RETURNS <type> EXECUTE ON HOST.

Такой синтаксис отсутствует в грамматике SQL ClickHouse и приводит к синтаксической ошибке ещё на этапе парсинга запроса.

Executable UDF в ClickHouse реализованы исключительно как табличные функции (executable()), возвращающие поток строк, а не как скалярные функции, встраиваемые в expression pipeline.

Таким образом, для выполнения поставленной в задании задачи меняю рекомендуемый код на свой следующим образом.

Учту дополнительно тот факт, что Clickhouse - это колоночная СУБД и обрабатывает данные по колонкам, а не 
по строкам, поэтому во внешнюю исполняемую функцию будут передаваться целиком столбцы, а не отдельные значения из ячеек таблицы.

Поэтому во-первых, я изменю код скрипта `total_price.py`

```python
#!/usr/bin/env python3
import sys
import json

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue

    row = json.loads(line)

    quantity = float(row["quantity"])
    price = float(row["price"])
    rn = row.get("rn")  # технический ключ для join

    out = {
        "result": quantity * price
    }

    if rn is not None:
        out["rn"] = rn

    print(json.dumps(out))
    sys.stdout.flush()
```

Здесь я использую технический ключ `rn`. Потому что возвращаться после обработки будет колнка целиком и по данному ключу я присоединю ее к основной таблице

Во-вторых, регистрацию EUDF сделаю через файл *.*ml (xml или yml) как указано в документации Clickhouse.
Редактирую файл udf.xml и добавляю в него путь к папке (которую также создаю дополнительно) `user_functions`, в которой будут размещаться файлы xml, регистрирующие EDUF

Новая редакция `udf.xml`

```xml
<clickhouse>
    <user_defined_executable_functions_config>
        <allow_functions>true</allow_functions>

        <!-- КЛЮЧЕВОЙ МОМЕНТ -->
        <function_config_dir>
            /etc/clickhouse-server/user_functions
        </function_config_dir>

        <execution_path>
            /var/lib/clickhouse/user_scripts
        </execution_path>
    </user_defined_executable_functions_config>
</clickhouse>
```

В папке `user_functions` я создаю файл `total_price_function.xml` следующего содержания:
```xml
<functions>
    <function>
        <type>executable</type>
        <name>total_price</name>

        <return_type>Float32</return_type>
        <return_name>result</return_name>

        <argument>
            <name>quantity</name>
            <type>Float32</type>
        </argument>

        <argument>
            <name>price</name>
            <type>Float32</type>
        </argument>

        <format>JSONEachRow</format>
        <command>total_price.py</command>

        <execute_direct>1</execute_direct>
        <deterministic>true</deterministic>
    </function>
</functions>
```

И сразу добавлю аналогичный файл `transaction_category_function.xml` для функции `transaction_category.py`: 

```xml
<functions>
    <function>
        <type>executable</type>
        <name>transaction_category</name>
        <!-- Возвращаемое значение -->
        <return_type>String</return_type>
        <return_name>category</return_name>
        <!-- Аргументы функции -->
        <argument>
            <name>total_price</name>
            <type>Float32</type>
        </argument>
        <format>JSONEachRow</format>
        <command>transaction_category.py</command>
        <execute_direct>1</execute_direct>
        <deterministic>true</deterministic>
    </function>
</functions>
```

После этого делаю рестарт сервера Clickhouse


__Пояснения__:

- __Что сделал__: Установил внутри контейнера с Clickhouse Python и необходимые библиотеки для интеграции с ClickHouse.

- __Зачем__: Без Python-окружения EUDF работать не будут. Библиотека clickhouse-driver обеспечивает связь между ClickHouse и Python-скриптами.

- __Результат__: Готовое окружение для запуска Python-скриптов из ClickHouse.

- __Чему научился__: Важность подготовки среды для исполняемых функций и управления зависимостями.
  

---

### 📝 3. Использование EUDF

3.1 Рассчитайте общую цену для каждой транзакции (используя `total_price.py`): 
Выполяю запрос, переписанный мной с учетом сделанных в функции изменений 

```sql
SELECT
    t.transaction_id,
    t.quantity,
    t.price,
    e.result AS total_price
FROM
(
    SELECT
        row_number() OVER (ORDER BY transaction_id) AS rn,
        transaction_id,
        quantity,
        price
    FROM transactions
    LIMIT 10
) AS t
LEFT JOIN
(
    SELECT rn, result
    FROM executable(
        'total_price.py',
        'JSONEachRow',
        'rn UInt64, result Float32',
        (
            SELECT
                row_number() OVER (ORDER BY transaction_id) AS rn,
                quantity,
                price
            FROM transactions
            LIMIT 10
        )
    )
) AS e
USING rn
ORDER BY rn;
```

Результат выполнения запроса ![hw06_var2_sel1](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_var2_sel1.png) скорость выполнения запроса - 0.043 сек

3.2 Создаю более сложную Python EUDF: классификация транзакций. Python-скрипт `transaction_category.py`:

```python
#!/usr/bin/env python3
import sys
import json

def transaction_category(total_price, threshold=100):
    if total_price >= threshold:
        return "High Value"
    else:
        return "Low Value"

for line in sys.stdin:
    line = line.strip()
    if not line:
        continue

    row = json.loads(line)

    total_price = float(row["total_price"])
    rn = row.get("rn")

    out = {
        "category": transaction_category(total_price, 100)
    }

    if rn is not None:
        out["rn"] = rn

    print(json.dumps(out))
    sys.stdout.flush()
```

Применение `transaction_category.py` - категоризирую каждую транзакцию, используя
`transaction_category`:

```sql
SELECT
    t.transaction_id,
    t.quantity,
    t.price,
    round(t.total_price, 2),
    c.category
FROM
(
    SELECT
        row_number() OVER (ORDER BY transaction_id) AS rn,
        transaction_id,
        quantity,
        price,
        quantity * price AS total_price
    FROM transactions
    LIMIT 10
) AS t
LEFT JOIN
(
    SELECT rn, category
    FROM executable(
        'transaction_category.py',
        'JSONEachRow',
        'rn UInt64, category String',
        (
            SELECT
                row_number() OVER (ORDER BY transaction_id) AS rn,
                quantity * price AS total_price
            FROM transactions
            LIMIT 10
        )
    )
) AS c
USING rn
ORDER BY rn;
```

Результат выполнения запроса ![hw06_var2_sel2](https://github.com/realexpert1C/clickhouse-course/blob/5267f7be7a267075186ac8fcdf6866c36b50feae/images/hw06_var2_sel2.png) скорость выполнения запроса - 0.052 сек

__Пояснения__
Как видим из выполненных заданий, в Clickhouse возможно выполнение UDF как заданных пользователем в SQL запросе, так и исполняемых скриптов, которые создаются в отдельных файлах на других языках программирования. Однако, следует учитывать целесообразность такого применения. На примере учебных заданий видим, что EDUF отработали корректно, но скорость выполнения на порядок превысила аналогичные запросы в Варианте 1. 

#### Выводы

🔹 Что стало понятным в процессе работы
1.	SQL UDF в ClickHouse
	* являются синтаксическим сахаром над выражениями;
	* компилируются и выполняются внутри expression pipeline;
	* работают построчно и максимально эффективно;
	* не требуют внешних зависимостей, файлов и конфигураций.
👉 Идеальны для:
	* простых вычислений;
	* повторно используемой бизнес-логики;
	* аналитических расчётов внутри ClickHouse.
2.	Executable UDF (EUDF)
	* не являются скалярными функциями;
	* реализованы как табличные функции, возвращающие поток строк;
	* работают через IPC (stdin/stdout);
	* получают данные блоками, а не по одной строке;
	* требуют внешней среды (Python, права, конфигурация).
👉 Идеальны для:
	* сложной бизнес-логики;
	* интеграции с ML/AI моделями;
	* использования библиотек, недоступных в SQL;
	* прототипирования логики, которую сложно выразить в SQL.

---

🔹 Почему EUDF нельзя использовать так, как описано в задании

Учебное задание предполагает синтаксис:

`CREATE FUNCTION ... RETURNS ... EXECUTE ON HOST;`

Однако:
- такого синтаксиса не существует в грамматике SQL ClickHouse;
- RETURNS и EXECUTE ON HOST не поддерживаются парсером;
- EUDF не встраиваются в expression pipeline, как SQL UDF;
- они возвращают таблицы, а не значения.

📌 Фактически:

Executable UDF в ClickHouse — это внешние процессы, а не функции в классическом SQL-понимании.

---

🔹 Ключевое архитектурное различие

|Характеристика|SQL UDF|Executable UDF|
|--------------|-------|--------------|
|Тип|scalar expression|table function|
|Выполнение|внутри ClickHouse|внешний процесс|
|Производительность|максимальная|ниже|
|Гибкость|ограниченная|очень высокая|
|Зависимости|нет|Python / ML / libs|
|Лучшее применение|аналитика|сложная логика|

---

🔹 Чему удалось научиться
* правильно выбирать тип UDF под задачу;
* понимать архитектурные ограничения ClickHouse;
* работать с EUDF через executable() и JOIN;
* проектировать безопасные и воспроизводимые конфигурации;
* критически оценивать учебные примеры и документацию.

---

🔹 Практический вывод

Если задачу можно решить SQL — её нужно решать SQL.
EUDF стоит применять только тогда, когда:
- логика действительно выходит за пределы SQL,
- или требуется внешняя вычислительная среда.
