# 📊 Системные таблицы ClickHouse для мониторинга и анализа

В ClickHouse есть множество системных таблиц в базе `system`, которые предоставляют информацию о текущем состоянии сервера, запросах, ресурсах и внутренней статистике.

## 🔍 Основные таблицы

### `system.parts`
- Информация о всех партициях таблиц.
- Показывает количество активных и неактивных кусков (parts), размер, статус.
- Полезно для анализа фрагментации таблиц.

### `system.merges`
- Показывает текущие фоновые операции слияния кусков (merges).
- Важно для оценки активности движка MergeTree.

### `system.replication_queue`
- Актуально для таблиц ReplicatedMergeTree.
- Содержит задания по синхронизации реплик, их статус, время выполнения, ошибки.

### `system.replication_alter_columns_queue`
- Очередь операций `ALTER TABLE` в кластере.
- Удобно при отслеживании миграций и изменений схем.

### `system.query_log`
- Лог завершённых запросов (по умолчанию TTL 7 дней).
- Включает:
  - Время начала/завершения
  - Пользователь
  - Тип запроса
  - Кол-во прочитанных/записанных строк и байт
  - Ошибки выполнения

### `system.text_log` и `system.trace_log`
- `text_log`: сообщения логгера (уровни INFO, WARNING, ERROR).
- `trace_log`: стек вызовов (stack traces), при падениях и долгих запросах.
- Используется для глубокого профилирования.

### `system.metric_log`
- Периодическая запись показателей ClickHouse:
  - Загрузка CPU
  - Использование памяти
  - Количество соединений
- Аналогично метрикам Prometheus.

### `system.processes`
- Активные в данный момент запросы.
- Поля: query_id, user, адрес клиента, сколько строк/байт уже обработано.

### `system.asynchronous_metrics`
- Метрики, обновляющиеся асинхронно в фоне.
- Пример: usage of memory, page cache, disk I/O.

### `system.events`
- Счётчики системных событий: сколько раз вызван insert, select, какие ошибки происходили.
- Можно агрегировать за время работы сервера.

## 🛠 Примеры полезных запросов

Получить текущие активные запросы:
```sql
SELECT query_id, query, elapsed, read_rows, memory_usage
FROM system.processes
ORDER BY elapsed DESC;