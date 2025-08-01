# ClickHouse OTUS Course

Домашние задания и проектная работа по курсу **"ClickHouse для инженеров и архитекторов БД"** (OTUS).

## Структура

Каждое задание хранится в отдельной папке `hwXX_имя`. Внутри — пояснения, SQL-скрипты, схемы, ссылки.


## 📚 Содержание курса: ClickHouse для инженеров и архитекторов БД

1. **Введение**
   - Общая архитектура ClickHouse
   - Принципы работы СУБД
   - Сценарии использования
   - ⚡️ [Конспект лекции 1 + Домашнее задание](./hw00_intro/README.md) 

2. **Развертывание и базовая конфигурация**
   - Способы установки
   - Конфигурационные файлы
   - Интерфейсы: HTTP, Native, ClickHouse Client, DBeaver
   - ⚡️ [Конспект лекции 2 + Домашнее задание](./hw01_adaptation/README.md) 
3. **MergeTree и типы данных**
   - Типы таблиц
   - MergeTree: ключи, партиции, сортировка
   - Типы данных в ClickHouse
   - ⚡️ [Конспект лекции 3](./hw02_installation/README.md)

4. **Язык запросов ClickHouse**
   - SELECT, WHERE, GROUP BY, JOIN
   - Массивы, вложенные структуры, функции
   - ⚡️ [Домашнее задание](https://github.com/realexpert1C/clickhouse-course/tree/main/hw05_sql_features)

5. **Функции в Clickhouse и использование UDF функций** 
   - Конспект лекции
   - Домашнее задание 

6. **Архитектура хранения и сжатие**
   - Устройство хранения
   - Сегментация, слияния, сжатие
   - TTL, Lightweight Delete

7. **Настройки производительности**
   - Конфигурация ресурсов
   - Буферы, потоки, кэш
   - Бенчмаркинг

8. **Кластеризация ClickHouse**
   - Децентрализованный кластер
   - Distributed tables
   - Zookeeper / ClickHouse Keeper

9. **Архитектура отказоустойчивости**
   - Репликация
   - HA решения
   - Бэкапы, restore

10. **Интеграции и экосистема**
   - Kafka, S3, SQL/BI инструменты
   - Материализованные представления

11. **Нагрузочное тестирование и отладка**
    - Инструменты анализа запросов
    - Метрики, профилирование, графики

12. **Безопасность и управление доступом**
    - Аутентификация
    - Роли и политики
    - Аудит

13. **Продвинутые темы**
    - User-defined functions
    - Механизмы расширения
    - Будущее ClickHouse

---

🔗 Каждый раздел может ссылаться на соответствующий `*.md` файл с конспектом.



## Темы курса

- Знакомство с аналитическими СУБД и ClickHouse
- Установка и настройка
- MergeTree, SQL, индексы, джоины
- Шардирование, репликация, мутации
- BI-интеграции, Kafka, Airflow
- RBAC, мониторинг, профилирование
- Финальный проект

## Ссылки

- [Курс на OTUS](https://otus.ru/lessons/clickhouse/)
- [ClickHouse Docs](https://clickhouse.com/docs)
- [Глоссарий терминов](docs/GLOSSARY.md)
