# Отчет по лабораторной работе №4
# Техобслуживание: Очистка (VACUUM)

**Дата:** 2025-10-08 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Звездин Алексей Сергеевич 

## Цель работы
Всестороннее изучение механизмов очистки (VACUUM) в PostgreSQL. Получение
практических навыков управления ручной и автоматической очисткой, анализа работы HOT-
обновлений, исследования влияния очистки на размер таблиц и индексов, а также работы с
заморозкой версий строк.

## Практическая часть

### Часть 1. Ручная очистка и ее влияние
**Выполненные задачи:**   
1. Глобально отключил процесс автоочистки. Убедился, что он не работает.

2. В новой базе данных `lb_04` создал таблицу `vacuum_test (id INT)` и индекс по полю `id`. Вставил в таблицу 100 000 случайных чисел.

3. 3 раза обновил половину строк в таблице (`UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;`). После каждого обновления контролировал размер таблицы и индекса с помощью `pg_total_relation_size`. Зафиксировал рост размеров.

4.  Выполнил `VACUUM FULL vacuum_test;`. Размеры таблицы и индекса уменьшились почти в три раза.

5.  Повторил цикл обновлений из пункта 3, но после каждого обновления вызывал обычную очистку (`VACUUM vacuum_test;`). После первого обновления размер индекса и таблицы увеличились. В следующих обновлениях индекс не увеличился, а размер таблицы немного увеличился. По сравнению с обновлением в 3 пункте размер уменьшился.

6.  Включил автоочистку обратно.


**Выполненные команды и результат**

**Задача 1**
```sql
student=# ALTER SYSTEM SET autovacuum = off;
student=# SELECT pg_reload_conf();
student=# SHOW autovacuum;
```

**Результат**
```text
ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)
 autovacuum 
------------
 off
(1 row)
```

**Задача 2**
```sql
student=# CREATE DATABASE lb0_4;
student=# \c lb0_4
lb0_4=# CREATE TABLE vacuum_test (id INT);
lb0_4=# CREATE INDEX idx_vacuum_test_id ON vacuum_test(id);
lb0_4=# INSERT INTO vacuum_test 
SELECT floor(random() * 1000000)::int 
FROM generate_series(1, 100000);
```

**Результат**
```text
CREATE DATABASE
CREATE TABLE
CREATE INDEX
INSERT 0 100000
```

**Задача 3**
```sql
CREATE OR REPLACE FUNCTION check_sizes()
RETURNS TABLE(relation_name text, size_bytes bigint) AS $$
BEGIN
    RETURN QUERY 
    SELECT 'table'::text, pg_total_relation_size('vacuum_test')
    UNION ALL
    SELECT 'index'::text, pg_total_relation_size('idx_vacuum_test_id');
END;
$$ LANGUAGE plpgsql;
SELECT * FROM check_sizes();
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
SELECT * FROM check_sizes();
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
SELECT * FROM check_sizes();
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
SELECT * FROM check_sizes();
```

**Результат**
```text
CREATE FUNCTION
 relation_name | size_bytes 
---------------+------------
 table         |    6307840
 index         |    2654208
(2 rows)

UPDATE 50075
 relation_name | size_bytes 
---------------+------------
 table         |    9928704
 index         |    4456448
(2 rows)

UPDATE 50243
 relation_name | size_bytes 
---------------+------------
 table         |   11624448
 index         |    5234688
(2 rows)

UPDATE 50119
 relation_name | size_bytes 
---------------+------------
 table         |   15441920
 index         |    7823360
(2 rows)

```

**Задача 4**
```sql
lb0_4=# SELECT * FROM check_sizes();
lb0_4=# VACUUM FULL vacuum_test;
lb0_4=# SELECT * FROM check_sizes();
```

**Результат**
```text
 relation_name | size_bytes 
---------------+------------
 table         |   15441920
 index         |    7823360
(2 rows)

VACUUM

 relation_name | size_bytes 
---------------+------------
 table         |    5865472
 index         |    2236416
(2 rows)

```

**Задача 5**
```sql
lb0_4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
VACUUM vacuum_test;
SELECT * FROM check_sizes();

lb0_4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
VACUUM vacuum_test;
SELECT * FROM check_sizes();

lb0_4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
VACUUM vacuum_test;
SELECT * FROM check_sizes();
```

**Результат**
```text
UPDATE 50055
VACUUM
 relation_name | size_bytes 
---------------+------------
 table         |    9936896
 index         |    4464640
(2 rows)

UPDATE 49868
VACUUM
 relation_name | size_bytes 
---------------+------------
 table         |    9936896
 index         |    4464640
(2 rows)

UPDATE 50198
VACUUM
 relation_name | size_bytes 
---------------+------------
 table         |    9945088
 index         |    4464640
(2 rows)
```

**Задача 6**
```sql
lb0_4=# ALTER SYSTEM SET autovacuum = on;
SELECT pg_reload_conf();
lb0_4=# SHOW autovacuum;
```

**Резульат**
```text
ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)

 autovacuum 
------------
 on
(1 row)
```

### Часть 2. HOT-обновления и самоочистка
**Выполненные задачи:**   
1.	Создал таблицу без индексов. Вставил данные. Выполнил несколько обновлений, не удовлетворяющих условиям HOT. Проанализировал табличную страницу до и после обновлений и последующей самоочистки. Следил за появлением и исчезновением мертвых кортежей.

2.	Создал таблицу `hot_test` и индекс по одному из полей. Вставил строку. Выполнил обновление, которое удовлетворяет условиям HOT. С помощью `pageinspect` убедился, что новая версия строки находится на той же странице, а запись в индексе не изменилась.

3.	Воспроизвёл ситуацию, когда на странице недостаточно места для нового HOT-обновления. Выполнид обновление. Убедился с помощью `pageinspect`, что новая версия создалась на другой странице. Проверил, сколько записей теперь ссылается на этот кортеж в индексе.

**Выполненные команды и результат**

**Задача 1**
```sql
lb0_4=# CREATE TABLE no_hot_test (
    id SERIAL PRIMARY KEY,
    data1 TEXT,
    data2 TEXT
);

lb0_4=# INSERT INTO no_hot_test (data1, data2) 
SELECT 'data1_' || i, 'data2_' || i 
FROM generate_series(1, 100) i;

lb0_4=# CREATE EXTENSION IF NOT EXISTS pageinspect;

lb0_4=# SELECT lp, t_xmin, t_xmax, t_ctid  
FROM heap_page_items(get_raw_page('no_hot_test', 0)) 
LIMIT 10;

lb0_4=# UPDATE no_hot_test SET data1 = 'updated_' || id WHERE id % 2 = 0;

lb0_4=# SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('no_hot_test', 0)) 
WHERE t_xmax != 0 LIMIT 10;

lb0_4=# VACUUM no_hot_test;

lb0_4=# SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('no_hot_test', 0)) 
WHERE t_xmax != 0 LIMIT 10;
```
**Результат**
```text
CREATE TABLE
INSERT 0 100
CREATE EXTENSION

 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    758 |      0 | (0,1)
  2 |    758 |      0 | (0,2)
  3 |    758 |      0 | (0,3)
  4 |    758 |      0 | (0,4)
  5 |    758 |      0 | (0,5)
  6 |    758 |      0 | (0,6)
  7 |    758 |      0 | (0,7)
  8 |    758 |      0 | (0,8)
  9 |    758 |      0 | (0,9)
 10 |    758 |      0 | (0,10)
(10 rows)

UPDATE 50

 lp | t_xmin | t_xmax | t_ctid  
----+--------+--------+---------
  2 |    758 |    762 | (0,101)
  4 |    758 |    762 | (0,102)
  6 |    758 |    762 | (0,103)
  8 |    758 |    762 | (0,104)
 10 |    758 |    762 | (0,105)
 12 |    758 |    762 | (0,106)
 14 |    758 |    762 | (0,107)
 16 |    758 |    762 | (0,108)
 18 |    758 |    762 | (0,109)
 20 |    758 |    762 | (0,110)
(10 rows)

VACUUM

 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
(0 rows)
```

**Задача 2**
```sql
lb0_4=# CREATE TABLE hot_test (id SERIAL PRIMARY KEY, main_data TEXT, secondary_data TEXT);

lb0_4=# CREATE INDEX idx_hot_main ON hot_test(main_data);

lb0_4=# INSERT INTO hot_test (main_data, secondary_data) VALUES ('main1', 'secondary1');

lb0_4=# SELECT lp, t_xmin, t_xmax, t_ctid, 
       (t_infomask2 & 2048) != 0 as HEAP_HOT_UPDATED,
       (t_infomask2 & 1024) != 0 as HEAP_ONLY_TUPLE
FROM heap_page_items(get_raw_page('hot_test',0));
```
**Результат**
```text
CREATE TABLE
CREATE INDEX
INSERT 0 1
 lp | t_xmin | t_xmax | t_ctid | heap_hot_updated | heap_only_tuple 
----+--------+--------+--------+------------------+-----------------
  1 |    765 |      0 | (0,1)  | f                | f
(1 row)
```

**Задача 3**
```sql
student=# CREATE TABLE hot_overflow (id INT, data TEXT);
student=# ALTER TABLE hot_overflow SET (fillfactor = 20);
student=# 
INSERT INTO hot_overflow 
SELECT i, rpad('X', 1000, 'X') FROM generate_series(1, 5) i;
student=# CREATE INDEX idx_overflow ON hot_overflow(id);
student=# UPDATE hot_overflow SET data = rpad('A', 1000, 'A') WHERE id = 1;
UPDATE hot_overflow SET data = rpad('B', 1000, 'B') WHERE id = 1;
UPDATE hot_overflow SET data = rpad('C', 1000, 'C') WHERE id = 1;
UPDATE hot_overflow SET data = rpad('D', 1000, 'D') WHERE id = 1;
student=# SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('hot_overflow', 0)) WHERE lp = 1;
student=# SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('hot_overflow', 1)) WHERE lp = 1;
student=# SELECT COUNT(*) as index_entries FROM bt_page_items('idx_overflow', 1);
```
**Результат**
```text
CREATE TABLE
ALTER TABLE 
INSERT 0 5
CREATE INDEX
UPDATE 1
UPDATE 1
UPDATE 1
UPDATE 1
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |        |        | 
(1 row)
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    810 |      0 | (1,1)
(1 row)
 index_entries 
---------------
             5
(1 row)
```

# Часть 1: Ручная очистка и ее влияние

## 1. Отключение автоочистки

- Автоочистка (autovacuum) - автоматический процесс обслуживания PostgreSQL
- Отключение через ALTER SYSTEM + pg_reload_conf() или конфиг файл
- Без автоочистки мертвые кортежи накапливаются, занимая место
- Требуется ручное управление процессом VACUUM

## 2. Накопление мертвых кортежей

- При UPDATE/DELETE старые версии строк становятся мертвыми кортежами
- Без очистки размер таблицы и индексов постоянно растет
- Мертвые кортежи замедляют операции чтения (больше данных для сканирования)
- Индексы содержат ссылки на несуществующие кортежи

## 3. VACUUM FULL vs обычный VACUUM

- Обычный VACUUM - освобождает место для повторного использования в той же таблице
- VACUUM FULL - переписывает всю таблицу, физически освобождая дисковое пространство
- VACUUM FULL требует эксклюзивной блокировки таблицы, обычный VACUUM - нет
- VACUUM FULL возвращает пространство ОС, обычный VACUUM - нет

## 4. Влияние на производительность

- Без очистки: постоянный рост размера, замедление операций
- С обычным VACUUM: стабильный размер, место переиспользуется
- С VACUUM FULL: минимальный размер, но дорогая операция

# Часть 2: HOT-обновления и самоочистка

## 1. HOT (Heap-Only Tuples) механизм

- Специальная оптимизация для уменьшения накладных расходов при обновлениях
- Работает когда обновляются поля, не входящие в индексы
- Новая версия строки создается на той же странице
- Индекс продолжает указывать на оригинальную версию

## 2. Условия для HOT-обновлений

- Обновляются только поля, не входящие в индексы
- На странице достаточно свободного места для новой версии
- Цепочка версий не превышает максимальную длину
- Все версии находятся в пределах одной страницы

## 3. HOT-обновление с переносом

- Когда на странице недостаточно места для новой версии
- Новая версия создается на другой странице
- Индекс все еще указывает на оригинальную версию
- Создается межстраничная HOT-цепочка
- В индексе остается одна запись, несмотря на перенос

## 4. Преимущества HOT

- Уменьшение фрагментации индексов (не требуется обновление индексов)
- Снижение нагрузки на WAL (меньше записей в журнале)
- Улучшение производительности частых обновлений
- Эффективное использование пространства

## 5. Самоочистка и HOT

- Autovacuum автоматически обрабатывает HOT-цепи
- Удаляет старые версии из цепочек
- Освобождает место для новых HOT-обновлений
- Поддерживает оптимальную длину цепочек

## Вывод
В ходе выполнения лабораторной работы я всестороннее изучил механизмы очистки (VACUUM) в PostgreSQL. Получил
практические навыки управления ручной и автоматической очисткой, анализа работы HOT-
обновлений, исследования влияния очистки на размер таблиц и индексов, а также работал с
заморозкой версий строк.