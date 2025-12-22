#### Домашнее задание "Джоины и агрегации в ClickHouse"

---
##### 1. ✅ Создание базы данных и таблиц

```sql
CREATE DATABASE IF NOT EXISTS imdb;
```

```sql
CREATE TABLE IF NOT EXISTS imdb.actors  
(  
    id         UInt32,  
    first_name String,  
    last_name  String,  
    gender     FixedString(1)  
) ENGINE = MergeTree
ORDER BY (id, first_name, last_name, gender);
```

```sql
CREATE TABLE IF NOT EXISTS imdb.genres  
(  
    movie_id UInt32,  
    genre    String  
) ENGINE = MergeTree
ORDER BY (movie_id, genre);
```

```sql
CREATE TABLE IF NOT EXISTS imdb.movies  
(  
    id   UInt32,  
    name String,  
    year UInt32,  
    rank Float32 DEFAULT 0  
) ENGINE = MergeTree
ORDER BY (id, name, year);
```

```sql
CREATE TABLE IF NOT EXISTS imdb.roles  
(  
    actor_id   UInt32,  
    movie_id   UInt32,  
    role       String,  
    created_at DateTime DEFAULT now()  
) ENGINE = MergeTree
ORDER BY (actor_id, movie_id);
```

---

#### ✅ 2. Загрузка тестовых данных из S3

```sql
INSERT INTO imdb.actors  
SELECT *  
FROM s3(
  'https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_actors.tsv.gz',  
  'TSVWithNames'
);
```

```sql
INSERT INTO imdb.genres  
SELECT *  
FROM s3(
  'https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies_genres.tsv.gz',  
  'TSVWithNames'
);
```

```sql
INSERT INTO imdb.movies  
SELECT *  
FROM s3(
  'https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_movies.tsv.gz',  
  'TSVWithNames'
);
```

```sql
INSERT INTO imdb.roles(actor_id, movie_id, role)  
SELECT actor_id, movie_id, role  
FROM s3(
  'https://datasets-documentation.s3.eu-west-3.amazonaws.com/imdb/imdb_ijs_roles.tsv.gz',  
  'TSVWithNames'
);
```

---

Проверяю, что таблицы созданы и данные загружены

🔎 Проверяю список таблиц:

```sql
SHOW TABLES FROM imdb;
```
Результат выполнения запроса ![hw10_check1](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_check1.png)

🔎 Проверяю количество строк:

```sql
SELECT 
    'actors' AS table, count() AS rows FROM imdb.actors
UNION ALL
SELECT 'genres', count() FROM imdb.genres
UNION ALL
SELECT 'movies', count() FROM imdb.movies
UNION ALL
SELECT 'roles', count() FROM imdb.roles;
```
Результат выполнения запроса ![hw10_check2](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_check2.png)

---

#### ✅ 3. Построение запросов

1. Найти жанры для каждого фильма
Для этого к таблице `movies` присоединяю таблицу `genres`, а именно строки, совпадающие по ключу `id` = `movie_id`. Использую __INNER JOIN__. 

```sql
SELECT 
    m.id,
    m.name,
    g.genre
FROM imdb.movies m
JOIN imdb.genres g ON m.id = g.movie_id
ORDER BY m.id
LIMIT 10;
```
В запросе указал __JOIN__, потому что по умолчанию в ClickHouse — это __INNER JOIN__.

Результат выполнения запроса ![hw10_select1](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_select1.png)

---

2. Запросить все фильмы, у которых нет жанра
Для этого к таблице `movies` делаю __LEFT JOIN__ также таблицы `genres` и фильтрую по условию `g.genre IS NULL` или пустая строка. 

```sql
SELECT 
    m.*, 
    g.genre
FROM imdb.movies AS m
LEFT JOIN imdb.genres AS g ON m.id = g.movie_id
WHERE g.genre IS NULL OR g.genre = ''
LIMIT 10;
```

Результат выполнения запроса ![hw10_select2](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_select2.png)

---

3. Объединить каждую строку из таблицы “Фильмы” с каждой строкой из таблицы “Жанры”
Это типичный __CROSS JOIN__ — декартово произведение.

```sql
SELECT *
FROM imdb.movies m
CROSS JOIN imdb.genres g
LIMIT 100;
```

Результат выполнения запроса ![hw10_select3](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_select3.png)

---

4. Найти жанры для каждого фильма, НЕ используя __INNER JOIN__
Решаю с помощью __LEFT JOIN__ и фильтра `g.genre IS NOT NULL` или пустая строка, результат эквивалентен __INNER JOIN__.

```sql
SELECT 
    m.name, 
    g.genre
FROM imdb.movies m
LEFT JOIN imdb.genres g ON m.id = g.movie_id
WHERE g.genre IS NOT NULL AND g.genre != ''
LIMIT 10;
```

Результат выполнения запроса ![hw10_select4](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_select4.png)

---

5. Найти всех актеров и актрис, снявшихся в фильме в N году
Присоединяю `movies` → `roles` → `actors`, фильтрую по year = 2002. Используется __INNER JOIN__.

```sql
SELECT 
    a.first_name, 
    a.last_name, 
    m.name AS movie, 
    m.year
FROM imdb.movies m
JOIN imdb.roles r ON m.id = r.movie_id
JOIN imdb.actors a ON r.actor_id = a.id
WHERE m.year = 2002
LIMIT 10;
```

Результат выполнения запроса ![hw10_select5](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_select5.png)

---

6. Запросить все фильмы, у которых нет жанра, через __ANTI JOIN__

```sql
SELECT m.*, g.genre  
FROM imdb.movies m  
ANTI LEFT JOIN imdb.genres g  ON m.id = g.movie_id
LIMIT 10;
```

Результат выполнения запроса ![hw10_select6](https://github.com/realexpert1C/clickhouse-course/blob/1c8c10149292b8c4b2ed44bf4578971fbc6c6ce5/images/hw10_select6.png)

---

#### ✅ Выводы

Какая работа была проделана
* Созданы таблицы и база данных imdb в ClickHouse с использованием MergeTree-движка.
* Загружены реальные тестовые данные из публичных S3-источников.
* Реализованы различные типы JOIN-ов: INNER, LEFT, ANTI, CROSS, а также построены запросы с фильтрацией.
* Проверена структура и наполнение таблиц, проанализированы особенности соединения данных в ClickHouse.

Что узнал и чему научился в процессе выполнения
* Закрепил различия между типами соединений: INNER, LEFT, CROSS, ANTI JOIN.
* Осознал поведение ClickHouse при JOIN-ах, в том числе важность фильтрации NULL и пустых строк.
* Научился использовать ANTI JOIN для поиска отсутствующих связей.
* Освоил работу с реальными данными из S3.

---