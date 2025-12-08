# Отчет по лабораторной работе №6
# Надежность: Блокировки и мониторинг

**Дата:** 2025-10-09 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Звездин Алексей Сергеевич

## Цель работы
Изучить систему блокировок в PostgreSQL и методы мониторинга активности сервера. Получить практические навыки анализа статистики, диагностики блокировок и взаимоблокировок, использования инструментов мониторинга.

## Практическая часть

### Часть 1. Мониторинг активности
**Выполненные задачи:**   

1. В новой базе данных создал таблицу monitor_test (id INT). Вставил несколько строк,
затем удалил все.
Изучил статистику обращений к таблице в pg_stat_all_tables (n_tup_ins, n_tup_del,
n_live_tup, n_dead_tup). Сопоставил с действиями.
Выполнил VACUUM. Снова проверил статистику.
VACUUM уменьшает мёртвые строки и увеличивает vacuum_count.

2. Создал ситуацию взаимоблокировки двух транзакций.
Изучил, какая информация записывается в журнал сообщений сервера при обнаружении
взаимоблокировки.

3. Установил и настроил расширение pg_stat_statements.
Выполнил несколько произвольных запросов.
Изучил информацию в представлении pg_stat_statements (топ запросов, время
выполнения и т.д.)

**Выполненные команды и результат**

**Задача 1**
```sql
student=# CREATE DATABASE lb_06;
student=# \c lb_06
lb_06=# CREATE TABLE monitor_test (id INT);
lb_06=# INSERT INTO monitor_test VALUES (1), (2), (3), (4), (5);
lb_06=# DELETE FROM monitor_test;
lb_06=# SELECT 
    n_tup_ins, 
    n_tup_del,
    n_live_tup,
    n_dead_tup
FROM pg_stat_all_tables 
WHERE relname = 'monitor_test';
lb_06=# VACUUM monitor_test;
lb_06=# SELECT 
    n_tup_ins, 
    n_tup_del,
    n_live_tup,
    n_dead_tup
FROM pg_stat_all_tables 
WHERE relname = 'monitor_test';
```

**Результат**
```text
CREATE DATABASE
You are now connected to database "lb_06" as user "student".
CREATE TABLE
INSERT 0 5
DELETE 5
 n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
-----------+-----------+------------+------------
         5 |         5 |          0 |          5
(1 row)
VACUUM
 n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
-----------+-----------+------------+------------
         5 |         5 |          0 |          0
(1 row)
```

**Задача 2**
session 1
```sql
lb_06=# begin;
lb_06=*# UPDATE monitor_test SET id = 110 WHERE id = 1;
```
session 2
```sql
lb_06=# BEGIN;
lb_06=*# UPDATE monitor_test SET id = 20 WHERE id = 2;
```

session 1
```sql
lb_06=*# UPDATE monitor_test SET id = 30 WHERE id = 2;
```

session 2
```sql
lb_06=*# UPDATE monitor_test SET id = 40 WHERE id = 1;
```

```bash
sudo tail -n 50 /var/log/postgresql/postgresql-16-main.log
```

**Результат**
session 1
```text
BEGIN
UPDATE 1
```

session 2
```text
BEGIN
UPDATE 2
```

session 1
```text
UPDATE 2
```

session 2
```text
ERROR:  deadlock detected
DETAIL:  Process 1351 waits for ShareLock on transaction 831; blocked by process 1332.
Process 1332 waits for ShareLock on transaction 832; blocked by process 1351.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,3) in relation "monitor_test"
```

```text
2025-12-08 05:20:31.780 MSK [1351] student@lb_06 ERROR:  deadlock detected
2025-12-08 05:20:31.780 MSK [1351] student@lb_06 DETAIL:  Process 1351 waits for ShareLock on transaction 831; blocked by process 1332.
	Process 1332 waits for ShareLock on transaction 832; blocked by process 1351.
	Process 1351: UPDATE monitor_test SET id = 40 WHERE id = 1;
	Process 1332: UPDATE monitor_test SET id = 30 WHERE id = 2;
2025-12-08 05:20:31.780 MSK [1351] student@lb_06 HINT:  See server log for query details.
2025-12-08 05:20:31.780 MSK [1351] student@lb_06 CONTEXT:  while updating tuple (0,3) in relation "monitor_test"
2025-12-08 05:20:31.780 MSK [1351] student@lb_06 STATEMENT:  UPDATE monitor_test SET id = 40 WHERE id = 1;
2025-12-08 05:25:05.813 MSK [810] LOG:  checkpoint starting: time
2025-12-08 05:25:05.930 MSK [810] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.103 s, sync=0.005 s, total=0.118 s; sync files=2, longest=0.003 s, average=0.003 s; distance=0 kB, estimate=0 kB; lsn=0/784C4F8, redo lsn=0/784C4C0
```

**Задача 3**

```sql
lb_06=*# CREATE EXTENSION pg_stat_statements;

lb_06=*# SELECT * FROM monitor_test;

lb_06=*# INSERT INTO monitor_test VALUES (100), (200), (300);

lb_06=*# UPDATE monitor_test SET id = id + 1 WHERE id > 150;

lb_06=*# DELETE FROM monitor_test WHERE id = 100;

lb_06=*# SELECT COUNT(*) FROM monitor_test;

lb_06=*# SELECT query, calls 
FROM pg_stat_statements 
ORDER BY calls DESC 
LIMIT 5;

lb_06=*# SELECT query, total_exec_time 
FROM pg_stat_statements 
ORDER BY total_exec_time DESC 
LIMIT 5;
```

**Результат**
```text
CREATE EXTENSION

 id 
----
  1
  2
  1
  2
(4 rows)

INSERT 0 3

UPDATE 2

DELETE 1

 count 
-------
     6
(1 row)

                       query                        | calls 
----------------------------------------------------+-------
 SELECT * FROM monitor_test                         |     1
 SELECT name, setting FROM pg_settings             +|     1
 WHERE name = $1                                    | 
 UPDATE monitor_test SET id = id + $1 WHERE id > $2 |     1
 INSERT INTO monitor_test VALUES ($1), ($2), ($3)   |     1
 BEGIN                                              |     1
(5 rows)

                      query                       |  total_exec_time   
--------------------------------------------------+--------------------
 CREATE EXTENSION pg_stat_statements              |          92.698507
 INSERT INTO monitor_test VALUES ($1), ($2), ($3) |           2.607912
 SELECT name, setting FROM pg_settings           +| 1.7177850000000001
 WHERE name = $1                                  | 
 SELECT * FROM monitor_test                       |           0.513332
 SELECT query, calls                             +|           0.171721
 FROM pg_stat_statements                         +| 
 ORDER BY calls DESC                             +| 
 LIMIT $1                                         | 
(5 rows)
```

### Часть 2. Блокировки объектов

**Выполненные задачи:**   

1. На уровне изоляции Read Committed прочитал одну строку таблицы по первичному
ключу. Изучил удерживаемые блокировки в pg_locks. 

2. Воспроизвел ситуацию автоматического повышения уровня предикатных блокировок
при чтении строк по индексу. Показал, что это может привести к ложной ошибке сериализаци

3. Настроил запись в журнал сообщений о ожиданиях блокировок > 100 мс (log_lock_waits
= on, deadlock_timeout = 100ms).
Создал ситуацию длительного ожидания блокировки. Убедился, что сообщение
появилось в логе.

**Выполненные команды и результат**

**Задача 1**

session 1
```sql
lb_06=*# CREATE TABLE test_locks (
    id INT PRIMARY KEY,
    value TEXT
);
CREATE TABLE
lb_06=*# INSERT INTO test_locks VALUES (1, 'test1'), (2, 'test2'), (3, 'test3');
lb_06=*# BEGIN;
lb_06=*# SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
lb_06=*# SELECT * FROM test_locks WHERE id = 1;
lb_06=*# SELECT pg_backend_pid();
```

session 2
```sql
SELECT 
    locktype,
    relation::regclass,
    mode,
    granted,
    virtualxid,
    transactionid
FROM pg_locks 
WHERE pid = 1892
ORDER BY locktype;
```

**Результат**

session 1
```text
INSERT 0 3
WARNING:  there is already a transaction in progress
BEGIN
SET
 id | value 
----+-------
  1 | test1
(1 row)
 pg_backend_pid 
----------------
           1892
(1 row)
```

session 2
```text
 object        |                              | AccessShareLock          | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | ShareUpdateExclusiveLock | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | ShareUpdateExclusiveLock | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | ShareUpdateExclusiveLock | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 object        |                              | ShareUpdateExclusiveLock | t       |            |              
 object        |                              | AccessExclusiveLock      | t       |            |              
 relation      | 41073                        | AccessExclusiveLock      | t       |            |              
 relation      | 41073                        | ShareUpdateExclusiveLock | t       |            |              
 relation      | pg_toast.pg_toast_2618_index | RowExclusiveLock         | t       |            |              
 relation      | 41079                        | AccessShareLock          | t       |            |              
 relation      | 41079                        | AccessExclusiveLock      | t       |            |              
 relation      | 41056                        | ShareUpdateExclusiveLock | t       |            |              
 relation      | 41056                        | AccessExclusiveLock      | t       |            |              
 relation      | 41084                        | AccessShareLock          | t       |            |              
 relation      | pg_toast.pg_toast_2618       | RowExclusiveLock         | t       |            |              
 relation      | monitor_test                 | AccessShareLock          | t       |            |              
 relation      | monitor_test                 | RowExclusiveLock         | t       |            |              
 relation      | pg_description               | RowExclusiveLock         | t       |            |              
 relation      | pg_settings                  | AccessShareLock          | t       |            |              
 relation      | 41088                        | AccessExclusiveLock      | t       |            |              
 relation      | 41084                        | RowExclusiveLock         | t       |            |              
 relation      | 41084                        | ShareLock                | t       |            |              
 relation      | 41084                        | AccessExclusiveLock      | t       |            |              
 relation      | 41062                        | ShareUpdateExclusiveLock | t       |            |              
 relation      | 41062                        | AccessExclusiveLock      | t       |            |              
 relation      | 41087                        | ShareLock                | t       |            |              
 relation      | 41087                        | AccessExclusiveLock      | t       |            |              
 relation      | 41068                        | AccessExclusiveLock      | t       |            |              
 relation      | 41089                        | AccessExclusiveLock      | t       |            |              
 relation      | 41089                        | AccessShareLock          | t       |            |              
 transactionid |                              | ExclusiveLock            | t       |            |           834
 virtualxid    |                              | ExclusiveLock            | t       | 3/8        |              
(46 rows)
```


**Задача 2**
```sql
lb_06=# CREATE TABLE serial_test (
    id SERIAL PRIMARY KEY,
    category TEXT,
    value INTEGER
);
lb_06=*# INSERT INTO serial_test (category, value) 
VALUES 
    ('A', 100),
    ('A', 200),
    ('B', 300);
lb_06=*# commit;
```

session 1
```sql
lb_06=# BEGIN ISOLATION LEVEL SERIALIZABLE;
lb_06=*# SELECT SUM(value) FROM serial_test WHERE category = 'A';
```

session 2
```sql
lb_06=# BEGIN ISOLATION LEVEL SERIALIZABLE;
lb_06=*# UPDATE serial_test SET value = 500 WHERE category = 'A' AND value = 100;
lb_06=*# COMMIT;
```

session 1
```sql
lb_06=*# UPDATE serial_test SET value = value + 50 WHERE category = 'A';
```

**Результат**
```text
INSERT 0 3
CREATE TABLE
COMMIT
```

session 1
```text
BEGIN
 sum 
-----
 300
(1 row)
```

session 2
```text
BEGIN
UPDATE 1
COMMIT
```

session 1
```text
ERROR:  could not serialize access due to concurrent update
```

**Задача 3**

session 1
```sql
lb_06=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '100ms';
SELECT pg_reload_conf();
lb_06=*# CREATE TABLE test_locks (
    id SERIAL PRIMARY KEY,
    data TEXT,
    counter INTEGER DEFAULT 0
);
lb_06=*# INSERT INTO test_locks (data) VALUES 
    ('row1'),
    ('row2'),
    ('row3');
lb_06=*# commit;
lb_06=# BEGIN;
lb_06=*# UPDATE test_locks SET counter = counter + 1 WHERE id = 1;
```

session 2
```sql
lb_06=# BEGIN;
lb_06=*# UPDATE test_locks SET counter = counter + 1 WHERE id = 1
```

```bash
sudo tail -f /var/log/postgresql/postgresql-16-main.log
```

**Результат**

session 1
```text
ALTER SYSTEM
ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)
CREATE TABLE
INSERT 0 3
COMMIT
BEGIN
UPDATE 1
```

session 2
```text
BEGIN
```

```text
2025-12-08 08:43:05.098 MSK [1364] student@lb_06 LOG:  process 1364 still waiting for ShareLock on transaction 865 after 100.115 ms
2025-12-08 08:43:05.098 MSK [1364] student@lb_06 DETAIL:  Process holding the lock: 1345. Wait queue: 1364.
2025-12-08 08:43:05.098 MSK [1364] student@lb_06 CONTEXT:  while updating tuple (0,1) in relation "test_locks"
2025-12-08 08:43:05.098 MSK [1364] student@lb_06 STATEMENT:  UPDATE test_locks SET counter = counter + 1 WHERE id = 1;
2025-12-08 08:44:43.244 MSK [754] LOG:  checkpoint starting: time
2025-12-08 08:44:43.968 MSK [754] LOG:  checkpoint complete: wrote 8 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.705 s, sync=0.010 s, total=0.724 s; sync files=7, longest=0.005 s, average=0.002 s; distance=3 kB, estimate=191 kB; lsn=0/7B6FD10, redo lsn=0/7B6FCD0
```


# Модуль 1: Мониторинг активности PostgreSQL

## 1. Статистика таблиц и VACUUM

- **pg_stat_all_tables** - системное представление для мониторинга таблиц
- **n_tup_ins** - счетчик вставленных строк (увеличивается при INSERT)
- **n_tup_del** - счетчик удаленных строк (увеличивается при DELETE)
- **n_live_tup** - количество "живых" (актуальных) строк в таблице
- **n_dead_tup** - количество "мертвых" (удаленных или устаревших) строк
- **DELETE без VACUUM** - строки помечаются как удаленные, но физически остаются
- **VACUUM** - очищает мертвые строки, освобождает место для повторного использования
- **После VACUUM** - n_dead_tup обнуляется, n_live_tup корректируется

## 2. Взаимоблокировки (Deadlocks)

- **Взаимоблокировка** - ситуация, когда две транзакции ждут ресурсы друг друга
- **Классический сценарий** - транзакции блокируют ресурсы в разном порядке
- **Детектор deadlock** - встроенный механизм PostgreSQL для обнаружения тупиков
- **Журналирование** - при обнаружении deadlock информация пишется в лог сервера
- **Сообщение в логе** - включает PID процессов, запросы, ресурсы блокировки
- **Разрешение** - PostgreSQL аварийно завершает одну из транзакций (жертву)

## 3. Расширение pg_stat_statements

- **pg_stat_statements** - расширение для сбора статистики выполнения SQL-запросов
- **Установка** - CREATE EXTENSION pg_stat_statements + настройка shared_preload_libraries
- **Представление pg_stat_statements** - содержит агрегированную статистику по запросам
- **queryid** - уникальный хеш запроса для идентификации одинаковых запросов
- **calls** - количество выполнений данного запроса
- **total_time** - общее время выполнения (в миллисекундах)
- **rows** - общее количество обработанных строк
- **mean_time** - среднее время выполнения одного вызова
- **Использование** - анализ "тяжелых" запросов, оптимизация производительности

# Модуль 2: Блокировки объектов в PostgreSQL

## 1. Блокировки при чтении

- **Блокировки на уровне строк** - захватываются при изменении данных (UPDATE, DELETE)
- **Блокировки на уровне таблиц** - захватываются при операциях DDL (ALTER TABLE и др.)
- **SELECT в Read Committed** - обычно не берет блокировок на строки (зависит от условий)
- **pg_locks** - системное представление для просмотра текущих блокировок
- **Виды блокировок** - AccessShareLock, RowShareLock, RowExclusiveLock, ShareLock
- **Locktype** - тип блокируемого объекта (relation, tuple, transactionid, etc.)
- **Mode** - режим блокировки (Share, Exclusive, AccessShare)

## 2. Повышение уровня блокировок

- **Предикатные блокировки** - блокировки на уровне условий (predicate locks)
- **Используются в Serializable** - для предотвращения аномалий сериализации
- **Автоматическое повышение** - PostgreSQL может повышать уровень блокировок
- **Ложная ошибка сериализации** - может возникнуть при консервативной блокировке
- **Сценарий с индексом** - чтение по индексу может привести к блокировке диапазона
- **Влияние на производительность** - повышенные блокировки увеличивают contention

## 3. Логирование долгих ожиданий блокировок

- **log_lock_waits** - параметр включения логирования долгих ожиданий блокировок
- **deadlock_timeout** - таймаут для обнаружения взаимоблокировок (по умолчанию 1с)
- **Порог логирования** - ожидания дольше deadlock_timeout попадают в лог
- **Сообщение в логе** - включает информацию о заблокированных процессах и ресурсах
- **Диагностика проблем** - помогает выявлять "узкие места" из-за блокировок
- **Рекомендуемые настройки** - log_lock_waits = on, deadlock_timeout = 100ms-1s


## Вывод
В процессе лабораторной работы была изучена системв блокировок в PostgreSQL и методы мониторинга активности сервера. Также были получены практические навыки анализа статистики, диагностики блокировок и взаимоблокировок, использования инструментов мониторинга.
