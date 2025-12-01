# Отчет по лабораторной работе №5
# Надежность: Журнал предзаписи (WAL)

**Дата:** 2025-10-08 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Звездин Алексей Сергеевич 


## Цель работы: 
Изучить работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Получить практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев.

## Практическая часть

### Часть 1. Процессы и режимы остановки

**Выполненные задачи:**

1. Нашел процессы checkpointer, background writer и walwriter с помощью команды ps aux | grep postgres.

2. Остановил PostgreSQL в режиме fast командой sudo pg_ctlcluster 16 main stop. Запустил сервер. В журнале /var/log/postgresql/postgresql-16-main.log нашел записи о контрольной точке при завершении работы.

3. Остановил PostgreSQL в режиме immediate командой sudo pg_ctlcluster 16 main stop -m immediate. Запустил сервер. В журнале не нашел записи о восстановлении после сбоя, так как postgres успел выполнить контрольную точку. В режиме fast выполняется контрольная точка перед завершением, а в режиме immediate происходит немедленное завершение с последующим восстановлением при запуске.

**Выполненные команды и результат**

**Задача 1**
```bash
ps aux | grep -E "(checkpointer|background writer|walwriter|postmaster)"
```

**Результаты**
```text
postgres     753  0.0  0.9 225588  9812 ?        Ss   22:45   0:00 postgres: 16/main: checkpointer 
postgres     754  0.0  0.7 225612  7448 ?        Ss   22:45   0:00 postgres: 16/main: background writer 
postgres     867  0.0  1.0 225456  9964 ?        Ss   22:45   0:00 postgres: 16/main: walwriter 
student     1683  0.0  0.2   9308  2284 pts/0    S+   23:26   0:00 grep --color=auto -E (checkpointer|background writer|walwriter|postmaster)
```
**Задача 2**
```bash
sudo pg_ctlcluster 16 main stop
sudo pg_ctlcluster 16 main start
sudo tail -50 /var/log/postgresql/postgresql-16-main.log
```

**Результаты**
```text
2025-11-30 22:50:58.401 MSK [753] LOG:  checkpoint starting: time
2025-11-30 22:50:59.327 MSK [753] LOG:  checkpoint complete: wrote 10 buffers (0.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.905 s, sync=0.009 s, total=0.927 s; sync files=7, longest=0.005 s, average=0.002 s; distance=54 kB, estimate=54 kB; lsn=0/71C4590, redo lsn=0/71C4558
2025-11-30 22:55:58.427 MSK [753] LOG:  checkpoint starting: time
2025-11-30 22:56:01.377 MSK [753] LOG:  checkpoint complete: wrote 30 buffers (0.2%); 0 WAL file(s) added, 0 removed, 0 recycled; write=2.915 s, sync=0.025 s, total=2.950 s; sync files=25, longest=0.003 s, average=0.001 s; distance=135 kB, estimate=135 kB; lsn=0/71E61D8, redo lsn=0/71E61A0
2025-11-30 23:28:07.371 MSK [741] LOG:  received fast shutdown request
2025-11-30 23:28:07.374 MSK [741] LOG:  aborting any active transactions
2025-11-30 23:28:07.391 MSK [741] LOG:  background worker "logical replication launcher" (PID 869) exited with exit code 1
2025-11-30 23:28:07.391 MSK [753] LOG:  shutting down
2025-11-30 23:28:07.393 MSK [753] LOG:  checkpoint starting: shutdown immediate
2025-11-30 23:28:07.401 MSK [753] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.011 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=121 kB; lsn=0/71E6288, redo lsn=0/71E6288
2025-11-30 23:28:07.406 MSK [741] LOG:  database system is shut down
2025-11-30 23:28:15.619 MSK [1717] LOG:  starting PostgreSQL 16.11 (Ubuntu 16.11-1.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
2025-11-30 23:28:15.619 MSK [1717] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2025-11-30 23:28:15.621 MSK [1717] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2025-11-30 23:28:15.637 MSK [1720] LOG:  database system was shut down at 2025-11-30 23:28:07 MSK
2025-11-30 23:28:15.649 MSK [1717] LOG:  database system is ready to accept connections
```
**Задача 3**
```bash
sudo pg_ctlcluster 16 main stop -m immediate
sudo pg_ctlcluster 16 main start
sudo tail -100 /var/log/postgresql/postgresql-16-main.log
```


**Результаты**
```text
2025-11-30 23:32:58.200 MSK [1718] LOG:  shutting down
2025-11-30 23:32:58.203 MSK [1718] LOG:  checkpoint starting: shutdown immediate
2025-11-30 23:32:58.219 MSK [1718] LOG:  checkpoint complete: wrote 4 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.004 s, sync=0.004 s, total=0.019 s; sync files=3, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=0 kB; lsn=0/71E6338, redo lsn=0/71E6338
2025-11-30 23:32:58.222 MSK [1717] LOG:  database system is shut down
2025-11-30 23:33:06.788 MSK [1806] LOG:  starting PostgreSQL 16.11 (Ubuntu 16.11-1.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
2025-11-30 23:33:06.788 MSK [1806] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2025-11-30 23:33:06.791 MSK [1806] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
2025-11-30 23:33:06.803 MSK [1809] LOG:  database system was shut down at 2025-11-30 23:32:58 MSK
2025-11-30 23:33:06.811 MSK [1806] LOG:  database system is ready to accept connections
```

### Часть 2. Буферный кеш и контрольные точки

**Выполненные задачи:**

1. Анализ размера:
Создал таблицу wal_test (id INT, data TEXT). Вставил в нее 10000 строк.
Определил, что таблица занимает 84 страницы на диске.
Определил, что таблица занимает 88 буферов в кеше.

2. Грязные буферы и контрольная точка:
Узнал общее количество "грязных" буферов в кеше - было 55 буферов.
Выполнил команду CHECKPOINT.
Снова проверил количество "грязных" буферов - стало 0 буферов. Количество уменьшилось потому что контрольная точка сбросила грязные буферы на диск.

3. Предварительное чтение (pg_prewarm):
Подключил расширение pg_prewarm.
Загрузил таблицу в кеш с помощью pg_prewarm.
Перезапустил сервер.
Проверил, осталась ли таблица в кеше - таблица не осталась в кеше. Метод не эффективен после перезапуска сервера так как буферный кеш очищается.

**Выполненные команды и результат**

**Задача 1**
```sql
student=# CREATE TABLE wal_test (id INT, data TEXT);
student=# INSERT INTO wal_test 
SELECT generate_series(1,10000), 
       md5(random()::text);
student=# SELECT 
    pg_relation_size('wal_test') as size_bytes,
    pg_relation_size('wal_test') / current_setting('block_size')::int as pages_count;
student=# CREATE EXTENSION pg_buffercache;
CREATE EXTENSION pg_stat_statements;
student=# SELECT 
    count(*) as buffers_in_cache,
    count(*) * current_setting('block_size')::int as cache_size_bytes
FROM pg_buffercache 
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
```
**Результат**
```text
CREATE TABLE
INSERT 0 10000
 size_bytes | pages_count 
------------+-------------
     688128 |          84
(1 row)
CREATE EXTENSION
CREATE EXTENSION

 buffers_in_cache | cache_size_bytes 
------------------+------------------
               88 |           720896
(1 row)
```

**Задача 2**
```sql
student=# BEGIN;
UPDATE wal_test SET data = md5(random()::text) WHERE id <= 3000;
student=*# SELECT 
    count(*) as dirty_buffers_before
FROM pg_buffercache 
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass) 
AND isdirty;
student=*# CHECKPOINT;
student=*# SELECT 
    count(*) as dirty_buffers_after
FROM pg_buffercache 
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass) 
AND isdirty;
student=*# COMMIT;
```

**Результат**
```text
BEGIN
UPDATE 3000
 dirty_buffers_before 
----------------------
                   55
(1 row)
CHECKPOINT
 dirty_buffers_after 
---------------------
                   0
(1 row)
COMMIT
```

**Задача 3**
```sql
student=# CREATE EXTENSION pg_prewarm;
student=# SELECT pg_prewarm('wal_test');
student=# SELECT 
    count(*) as buffers_after_prewarm
FROM pg_buffercache 
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
```
```bash
sudo systemctl restart postgresql
```
```sql
student=# SELECT 
    count(*) as buffers_after_restart
FROM pg_buffercache 
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
```
**Результат**
```text
CREATE EXTENSION
 pg_prewarm 
------------
        109
(1 row)
 buffers_after_prewarm 
-----------------------
                   113
(1 row)
 buffers_after_restart 
-----------------------
                     0
(1 row)
```


# Модуль 1: Процессы и режимы остановки PostgreSQL

## 1. Архитектура процессов PostgreSQL

- **Многопроцессная архитектура** - PostgreSQL использует несколько процессов для разных задач
- **Основные процессы**: postmaster (родительский), checkpointer, background writer, walwriter
- **Каждый процесс отвечает** за определенный аспект работы СУБД
- **Координация процессов** через разделяемую память и системные блокировки

## 2. Поиск и идентификация процессов

- **Postmaster** - главный процесс, принимает подключения
- **Checkpointer** - управляет контрольными точками, записывает dirty-буферы на диск
- **Background Writer** - фоновый процесс записи, разгружает основную нагрузку
- **WAL Writer** - записывает данные WAL на диск для обеспечения надежности
- **Методы поиска**: ps, pg_top, системные мониторы

## 3. Режим остановки FAST

- **Graceful shutdown** - плавное завершение работы
- **Новые подключения отклоняются**, существующие завершаются корректно
- **Контрольная точка выполняется** перед остановкой
- **Все dirty-буферы записываются** на диск
- **WAL файлы архивируются** при необходимости
- **Без потери данных**, но требует времени

## 4. Режим остановки IMMEDIATE

- **Аварийное завершение** - эквивалент отключению питания
- **Процессы завершаются немедленно** без подготовки
- **Контрольная точка не выполняется** перед остановкой
- **Dirty-буферы не записываются** - возможна потеря данных
- **Требуется восстановление** при следующем запуске

## 5. Процесс восстановления после сбоя

- **Анализ WAL** - поиск последней завершенной контрольной точки
- **REDO операций** - применение коммитованных изменений из WAL
- **UNDO операций** - откат незакоммитованных изменений
- **Восстановление консистентности** базы данных
- **Автоматический процесс** - выполняется при запуске после сбоя

## 6. Сравнение режимов остановки

- **FAST**: гарантированная целостность, медленное завершение
- **IMMEDIATE**: быстрое завершение, требуется восстановление
- **Выбор режима** зависит от критичности данных и требований к доступности

# Модуль 2: Буферный кеш и контрольные точки

## 1. Архитектура буферного кеша

- **Разделяемый буферный кеш** - общая память для всех процессов
- **Страничная организация** - данные хранятся блоками по 8KB (по умолчанию)
- **LRU алгоритм** - вытеснение редко используемых страниц
- **Dirty-буферы** - измененные страницы, требующие записи на диск

## 2. Анализ размера данных

- **Дисковый размер** - физическое место на диске (pg_relation_size)
- **Размер в кеше** - количество страниц в оперативной памяти
- **Коэффициент попадания** - отношение запросов из кеша к общему количеству
- **Методы анализа**: системные представления, расширения

## 3. Контрольные точки (Checkpoints)

- **Периодический процесс** синхронизации памяти и диска
- **Запись dirty-буферов** - перенос измененных данных на диск
- **Обновление WAL** - отметка завершенной контрольной точки
- **Триггеры**: время, объем WAL, ручной вызов
- **Влияние на производительность** - интенсивная операция ввода-вывода

## 4. Грязные буферы и их управление

- **Dirty-буферы** - страницы, измененные в памяти но не записанные на диск
- **Background Writer** - постепенно записывает dirty-буферы между контрольными точками
- **Checkpointer** - массовая запись всех dirty-буферов при контрольной точке
- **Мониторинг** через pg_stat_bgwriter и pg_buffercache

## 5. Механизм предварительного чтения (pg_prewarm)

- **Загрузка данных в кеш** до фактического использования
- **Уменьшение времени отклика** для критических данных
- **Добровольная операция** - не влияет на целостность данных
- **Временный эффект** - данные сохраняются только до перезапуска

## 6. Эффективность стратегий кеширования

- **Активные данные** - остаются в кеше долгое время
- **Пассивные данные** - быстро вытесняются из кеша
- **Предиктивная загрузка** - упреждающее кеширование на основе паттернов доступа
- **Мониторинг эффективности** через статистику использования буферов

## 7. Взаимодействие WAL и буферного кеша

- **Write-Ahead Logging** - гарантия надежности записи
- **Изменения сначала в WAL**, затем в буферный кеш
- **Контрольные точки** синхронизируют WAL и данные
- **Восстановление** использует WAL для репликации изменений

## Вывод
В процессе лабораторной работы изучил работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Также получилл практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев.