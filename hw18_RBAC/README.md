# Домашнее задание - Контроль доступа

### 📄 Текст задания

> 
>
> **Цель:**
> - освоить базовые операции управления пользователями и ролями;
>
> **Описание / шаги:**
> 1. Создайте пользователя jhon с паролем «qwerty»;
> 2. Создайте роль devs
> 3. Выдайте роли devs права на SELECT на любую таблицу
> 4. Выдайте роль devs пользователю jhon
> 5. Предоставьте результаты SELECT из system-таблиц, соответствующих созданным сущностям
> 
> **Критерии оценки:**
>
>Задание считается выполненным, если созданы пользователь и роль, выданы соответствующие права, и предоставлены результаты запросов из системных таблиц.

---

## 📄 SQL-скрипты

### Шаг 1. Создание пользователя `jhon` с паролем 'qwerty'
```sql
CREATE USER jhon IDENTIFIED WITH plaintext_password BY 'qwerty';
```

### Шаг 2. Создание роли `devs`
```sql
CREATE ROLE devs;
```

### Шаг 3. Выдача роли `devs` прав на SELECT на таблицу `system.numbers` (пример)
```sql
GRANT SELECT ON system.numbers TO devs;
```

### Шаг 4. Назначение роли devs пользователю jhon
```sql
GRANT devs TO jhon;
```

---

### Шаг 5. 📊 Проверка через системные таблицы


✅ Запросы, ожидаемые и фактические результаты запросов

#### Проверка наличия пользователя `jhon`
```sql
SELECT name FROM system.users WHERE name = 'jhon';
```

- `system.users`
```text
┌─name─┐
│ jhon │
└──────┘
```
Результат выполнения запроса
![hw18_check_jhon](https://github.com/realexpert1C/clickhouse-course/blob/4cee410f18a8d4b4edb850686d9e64fbe0a39c16/images/hw18_check_jhon.png)

#### Проверка роли `devs`
```sql
SELECT name FROM system.roles WHERE name = 'devs';
```

- `system.roles`
```text
┌─name─┐
│ devs │
└──────┘
```
Результат выполнения запроса
![hw18_check_devs](https://github.com/realexpert1C/clickhouse-course/blob/4cee410f18a8d4b4edb850686d9e64fbe0a39c16/images/hw18_check_devs.png)

#### Проверка выданных прав роли `devs`
```sql
SELECT * FROM system.grants WHERE role_name = 'devs';
```

- `system.grants`
```text
┌──────────┬──────────┬────────────┬────────┬─────────┬─────────────┐
│ user_name│ role_name│ access_type│database│ table   │ column_mask │
├──────────┼──────────┼────────────┼────────┼─────────┼─────────────┤
│ NULL     │ devs     │ SELECT     │ system │ numbers │ NULL        │
└──────────┴──────────┴────────────┴────────┴─────────┴─────────────┘
```
Результат выполнения запроса
![hw18_check_system_grants](https://github.com/realexpert1C/clickhouse-course/blob/4cee410f18a8d4b4edb850686d9e64fbe0a39c16/images/hw18_check_system_grants.png)

#### Проверка назначенной роли пользователю `jhon`
```sql
SELECT * FROM system.role_grants WHERE user_name = 'jhon';
```

- `system.role_grants`
```text
┌─user_name─┬─granted_role_name─┬─granted_role_is_default─┬─with_admin_option─┐
│ jhon      │ devs              │ 1                       │ 0                 │
└───────────┴───────────────────┴─────────────────────────┴───────────────────┘
```
Результат выполнения запроса
![hw18_check_role_grants](https://github.com/realexpert1C/clickhouse-course/blob/4cee410f18a8d4b4edb850686d9e64fbe0a39c16/images/hw18_check_role_grants.png)

---
### 📝 Комментарий с методической точки зрения

Задание направлено на практическое освоение базовых механизмов RBAC (Role-Based Access Control) в ClickHouse — это важный и часто недооценённый аспект безопасного администрирования БД.

Ключевые аспекты задания:
1.	Создание пользователя и роли — проверяется знание базового синтаксиса CREATE USER, CREATE ROLE.
2.	Выдача прав на объект — умение оперировать инструкцией GRANT, понимание уровня доступа (на таблицу, БД, колонку).
3.	Связь пользователя и роли — демонстрируется понимание, что пользователь может быть связан с ролью, а роль содержит права.
4.	Проверка в system.* таблицах — важный элемент: закрепляется навык верификации состояний безопасности через системные таблицы (что также критично при аудитах и CI/CD тестировании).

---

### ✅ Выводы по выполнению задания
* Все шаги выполнены в соответствии с заданием.
* Использована системная таблица `system.numbers` — демонстрационная таблица без необходимости создавать тестовые данные.
* SQL-запросы построены декларативно и читаемо, результат проверок охватывает все требуемые аспекты: наличие пользователя, роли, прав и привязок.
* Предоставлены результаты запросов из системных таблиц.