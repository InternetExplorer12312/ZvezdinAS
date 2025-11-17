# Отчет по лабораторной работе №3
# Модель многопользовательского доступа: MVCC

**Дата:** 2025-10-05  
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Звездин Алексей Сергеевич

## Цель работы
Изучить принципы многоверсионного управления конкурентным доступом (MVCC) в PostgreSQL. Получить практические навыки наблюдения за работой MVCC, анализа версий строк, снимков данных и уровней изоляции транзакций. Освоить использование расширений и системных представлений для исследования внутренней структуры данных.

## Практическая часть

### Часть 1. Уровни изоляции и аномалии

**Выполненные задачи:**   
1. Создал таблицу `iso_test (id INT, data TEXT)` и вставил одну строку. В сеансе 1 начал транзакцию с уровнем `READ COMMITTED` и выполнил `SELECT iso_test;. * FROM iso_test;`. В сеансе 2 удалил строку и зафиксировал изменения (`DELETE ...; COMMIT;`). В сеансе 1 выполнил тот же `SELECT` повторно. Вывело 0 строк. Завершил транзакцию в сеансе 1.

2. Повторил предыдущий эксперимент, но в сеансе 1 начал транзакцию с `BEGIN ISOLATION LEVEL REPEATABLE READ;`. `READ COMMITTED` берёт новый снимок на каждый запрос. `REPEATABLE READ` фиксирует снимок данных и держит его неизменным на всю транзакцию.

3. В сеансе 1 начал транзакцию и создал новую таблицу `new_table`, вставил в нее строку. Не фиксировал. В сеансе 2 выполнил `SELECT * FROM new_table;`. Выдало ошибку о несуществовании таблицы. Зафиксировал транзакцию в сеансе 1. Повторил запрос в сеансе 2, таблица появилась. Повторил процесс, но вместо фиксации откатил транзакцию в сеансе 1. После `ROLLBACK;` таблица также нету в сеансе 2.

4. В сеансе 1 начал транзакцию и выполнил `SELECT * FROM iso_test;`. Попытался в сеансе 2 выполнить `DROP TABLE iso_test;`. Удалить таблицу не получится, пока не завершим транзакцию в первом сеансе.

**Выполненные команды и результат**

**Задача 1**
session 1
```bash
sudo -u student psql
```
```sql
CREATE DATABASE lb03;
\c lb03
CREATE TABLE iso_test(id INT, data TEXT);
INSERT INTO iso_test VALUES (1,'any_text_data');
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT * FROM iso_test;
```

session 2 
```bash
sudo -u student psql
```
```sql
\c lb03 
DELETE FROM iso_test WHERE id=1;
COMMIT;
```

session 1
```sql
SELECT * FROM iso_test;
```

session 1

**Результат**
session 1
```text
CREATE DATABASE
You are now connected to database "lb03" as user "student"
CREATE TABLE
INSERT 0 1
BEGIN
 id |     data      
----+---------------
  1 | any_text_data
(1 row)
```

session 2
```text
You are now connected to database "lb03" as user "student"
DELETE FROM iso_test WHERE id=1;
WARNING:  there is no transaction in progress
COMMIT
```

session 1 
```text
 id | data 
----+------
(0 rows)
```

**Задача 2**
session 1
```sql
INSERT INTO iso_test VALUES (1,'any_text_data2');
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM iso_test;
```

session 2
```sql
DELETE FROM iso_test WHERE id=1;
COMMIT;
```

session 1 
```sql
SELECT * FROM iso_test;
```

**Результаты**
session 1
```text
INSERT 0 1
BEGIN
 id |      data      
----+----------------
  1 | any_text_data2
(1 row)
```

session 2
```text
DELETE 1
WARNING:  there is no transaction in progress
COMMIT
```

session 1
```text
 id |      data      
----+----------------
  1 | any_text_data2
(1 row)
```

**Задача 3**
session 1
```sql
BEGIN;
CREATE TABLE new_table(id int, data text);
INSERT INTO new_table VALUES (1, 'any_text_data3');
```

session 2
```sql
SELECT * FROM new_table;
```

session 1
```sql
COMMIT;
```

session 2
```sql
SELECT * FROM new_table;
```

session 1
```sql
BEGIN;
CREATE TABLE new_table2(id int, data text);
INSERT INTO new_table2 VALUES (1, 'any_text_data4');
```

session 2
```sql
SELECT * FROM new_table2;
```

session 1
```sql
ROLLBACK;
```

session 2
```sql
SELECT * FROM new_table2;
```

**Результаты**

session 1
```text
BEGIN
CREATE TABLE
INSERT 0 1
```

session 2
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
```

session 1
```text
COMMIT
```

session 2
```text
 id |      data      
----+----------------
  1 | any_text_data3
(1 row)
```

session 1
```text
BEGIN
CREATE TABLE
INSERT 0 1
```

session 2
```text
ERROR:  relation "new_table2" does not exist
LINE 1: SELECT * FROM new_table2;
```

session 1
```text
ROLLBACK
```

session 2
```text
ERROR:  relation "new_table2" does not exist
LINE 1: SELECT * FROM new_table2;
```

**Задача 4**

session 1
```sql
BEGIN;
lb03=*# SELECT * FROM iso_test;
```

session 2
```sql
DROP TABLE iso_test;

```

**Результаты**

session 1
```text
BEGIN
 id | data 
----+------
(0 rows)
```

session 2
```text

```


### Часть 2. Фантомное чтение и снимки
**Выполненные задачи:**   
1.	Создал пустую таблицу `phantom_test (id INT)`. Продемонстрировал на уровне `Read Committed`, что аномалия "фантомное чтение" не предотвращается (вставка новых строк в другом сеансе становится видимой).

2.	В сеансе 1 начал транзакцию с уровнем `Repeatable Read` (пока без запросов). В сеансе 2 удалил все строки из `phantom_test` и зафиксировал. В сеансе 1 выполнил `SELECT * FROM phantom_test;`. Удалённые стркои вывелись. Выполнил в сеансе 1 запрос `SELECT * FROM pg_database;` (не касаясь `phantom_test`). При последующем запросе также вывело удалённые строки.

3.	Убедился, что `DROP TABLE` является транзакционной операцией (можно откатить).


**Задача 1**

session 1
```sql
CREATE TABLE phantom_test(id INT);
lb03=# BEGIN ISOLATION LEVEL READ COMMITTED;
lb03=*# SELECT * FROM phantom_test;
```

session 2
```sql
INSERT INTO phantom_test VALUES (1);
COMMIT;
```

session 1
```sql
SELECT * FROM phantom_test;
```

**Результаты**

session 1
```text
CREATE TABLE
BEGIN
 id 
----
(0 rows)
```

session 2
```text
INSERT 0 1
WARNING:  there is no transaction in progress
COMMIT
```

session 1
```text
 id 
----
  1
(1 row)
```

**Задача 2**

session 1
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
```

session 2
```sql
DELETE FROM phantom_test;
COMMIT;
```

session 1
```sql
SELECT * FROM phantom_test; 
SELECT * FROM pg_database;   
SELECT * FROM phantom_test;
COMMIT;
```

**Результаты**

session 1
```text
BEGIN
```

session 2
```text
DELETE 1
WARNING:  there is no transaction in progress
COMMIT
```

session 1
```text
 id 
----
  1
(1 row)
 id 
----
  1
(1 row)
```

**Задача 3**

```sql
CREATE TABLE test_transaction (id INT, name TEXT);
INSERT INTO test_transaction VALUES (1, 'test_data');
COMMIT;
lb03=# BEGIN;
lb03=*# DROP TABLE test_transaction;
lb03=*# SELECT * FROM test_transaction;
lb03=!# ROLLBACK;
lb03=# SELECT * FROM test_transaction;
```

**Результаты**
```text
CREATE TABLE
INSERT 0 1
COMMIT
BEGIN
DROP TABLE
ERROR:  relation "test_transaction" does not exist
LINE 1: SELECT * FROM test_transaction;
ROLLBACK
 id |   name    
----+-----------
  1 | test_data
(1 row)
```

# Часть 1: Уровни изоляции и аномалии

## 1. Read Committed vs Удаление

- Уровень изоляции READ COMMITTED (по умолчанию в PostgreSQL)
- Видит только закоммиченные данные других транзакций
- НЕТ защиты от неповторяемого чтения
- При повторном SELECT видит актуальное состояние данных

## 2. Repeatable Read vs Удаление

- Уровень изоляции REPEATABLE READ
- Создает снимок данных при первом обращении к таблице
- Все SELECT в транзакции видят один и тот же снимок
- Защищает от неповторяемого чтения - удаленные строки остаются видимыми
- Снимок создается при первом запросе к конкретной таблице

## 3. Создание таблицы в транзакции

- DDL операции (CREATE TABLE) транзакционны в PostgreSQL
- До коммита новая таблица невидима для других сеансов
- После COMMIT таблица становится видимой для всех
- После ROLLBACK таблица исчезает

## 4. Блокировка DDL

- SELECT устанавливает ACCESS SHARE блокировку
- DROP TABLE требует EXCLUSIVE блокировку
- Эти блокировки несовместимы
- DROP будет ждать завершения транзакции с SELECT

# Часть 2: Фантомное чтение и снимки

## 1. Фантомное чтение (Read Committed)

- Read Committed не защищает от фантомного чтения
- Новые строки, добавленные другими транзакциями, видны при повторных SELECT
- "Фантомы" = строки, которых не было при первом чтении
- Аномалия возникает при вставке новых строк

## 2. Невидимость удалений (Repeatable Read)

- Repeatable Read защищает от фантомного чтения
- Снимок данных фиксируется при первом обращении к таблице
- Запросы к другим таблицам не влияют на уже созданный снимок
- Удаленные другими транзакциями строки остаются видимыми
- Новые строки, добавленные после начала транзакции, не видны

## 3. Транзакционность DDL

- DROP TABLE и другие DDL операции можно откатить в PostgreSQL
- ROLLBACK отменяет DROP TABLE и восстанавливает таблицу


# Ключевые понятия

- **Грязное чтение** - чтение незакоммиченных данных других транзакций
- **Неповторяемое чтение** - разные результаты при повторном чтении тех же данных
- **Фантомное чтение** - появление новых строк при повторном выполнении того же запроса
- **Снимок данных** - согласованное состояние данных на момент начала операции

## Вывод
В процессе выполнения лабораторной работы я изучил принципы многоверсионного управления конкурентным доступом (MVCC) в PostgreSQL. Получил практические навыки наблюдения за работой MVCC, снимков данных и уровней изоляции транзакций. 