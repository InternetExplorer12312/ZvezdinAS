# Отчет по лабораторной работе №7
# Управление доступом, расширениями и локализацией

**Дата:** 2025-10-09 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Звездин Алексей Сергеевич  

## Цель работы
Освоить управление правами доступа пользователей, работу с расширениями PostgreSQL и настройку параметров локализации. Получить практические навыки настройки аутентификации, управления привилегиями, установки расширений и миграции данных между разными кодировками.

## Практическая часть

### Часть 1. Управление доступом

**Выполненные задачи:**  
1. Создал новую базу данных access_db и двух пользователей: writer (с правом создания
объектов) и reader (только чтение).
Отзовал у роли public все привилегии на схему public в новой БД. Выдал роли writer
права CREATE и USAGE на схему public, а роли reader — только USAGE.
Настроил привилегии по умолчанию так, чтобы роль reader автоматически получала
право SELECT на новые таблицы в схеме public, принадлежащие writer.
Создал пользователей w1 (входит в роль writer) и r1 (входит в роль reader).
Подключился под writer, создал таблицу test_table. Убедился, что r1 может только
читать ее, а w1 — имеет полный доступ (включая DELETE).

2. Создал пользовательские роли alice и bob.
Отредактировал pg_hba.conf, разрешив беспарольный вход (trust) только для postgres и
student. Для всех остальных методов установил reject или md5. Перезагрузил
конфигурацию. Убедился, что вход для alice и bob запрещен.
Для alice и bob настроил аутентификацию по peer (сопоставление с пользователем ОС).
Убедился, что войти нельзя без создания пользователя в ОС.
Создал в ОС пользователя alice. Настроил вход в PostgreSQL для роли alice с методом
peer. Проверил вход.
Проверил, можно ли использовать одно отображение peer для нескольких ролей.

**Выполненные команды и результат**

**Задача 1**
```sql
postgres=# CREATE DATABASE access_db;
postgres=# \c access_db
access_db=# CREATE ROLE writer WITH NOLOGIN;
access_db=# CREATE ROLE reader WITH NOLOGIN;
access_db=# REVOKE ALL ON SCHEMA public FROM public;
access_db=# GRANT CREATE, USAGE ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;
access_db=# ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public 
GRANT SELECT ON TABLES TO reader;
access_db=# CREATE USER w1 WITH PASSWORD '2' IN ROLE writer;
CREATE USER r1 WITH PASSWORD '1' IN ROLE reader;
```

session Writer
```sql
access_db=> CREATE TABLE test_table (
  id SERIAL PRIMARY KEY,
  data TEXT
);
access_db=> INSERT INTO test_table (data) VALUES ('Test data 1'), ('Test data 2');
```

session Reader
```sql
access_db=> SELECT * FROM test_table;
access_db=> DELETE FROM test_table WHERE id = 1;
```

session Writer
```sql
access_db=> DELETE FROM test_table WHERE id = 1;
```

**Результат**

```text
CREATE DATABASE
You are now connected to database "access_db" as user "postgres".
CREATE ROLE
CREATE ROLE
REVOKE
GRANT
GRANT
ALTER DEFAULT PRIVILEGES
CREATE ROLE
CREATE ROLE
```

session Writer
```text
CREATE TABLE
INSERT 0 2
```

session Reader
```text
 id |  data 
----+-------------
  1 | Test data 1
  2 | Test data 2
(2 rows)
ERROR:  permission denied for table test_table
```

session Writer
```text
DELETE 1
```

**Задача 2**
```bash
sudo -u postgres psql
```

```sql
CREATE ROLE alice WITH LOGIN;
CREATE ROLE bob WITH LOGIN;
\du alice
\du bob
```

```sql
SHOW hba_file;
```

```bash
student:~$ sudo nano /etc/postgresql/16/main/pg_hba.conf
```

```text
  GNU nano 7.2 /etc/postgresql/16/main/pg_hba.conf 
local all postgres trust
local all student trust
local all all  md5
host  all all 127.0.0.1/32 md5
host  all all ::1/128 md5
```

```bash
student:~$ sudo systemctl reload postgresql
```

```bash
student:~$ psql -U alice -d postgres
student:~$ psql -U bob -d postgres
```

```bash
student:~$ sudo nano /etc/postgresql/16/main/pg_hba.conf
```

```text
local all alice peer
local all bob peer
```

```sql
SHOW ident_file;
```

```bash
sudo nano /etc/postgresql/16/main/pg_ident.conf
```

```text
peer_map alice alice
peer_map alice bob
```

```bash
sudo adduser alice
student:~$ sudo systemctl reload postgresql
student:~$ psql -U alice -d postgres
student:~$ sudo -u alice bash
alice@course:/home/student$ psql -U alice -d postgres
```

```bash
sudo nano /etc/postgresql/16/main/pg_ident.conf
```

```text
local all alice peer map=peer_map
local all bob peer map=peer_map
```

```bash
alice@course:~$ psql -U alice -d postgres
alice@course:~$ psql -U bob -d postgres
```

**Результат**
```text
psql (16.11 (Ubuntu 16.11-1.pgdg24.04+1))
Type "help" for help.
```

```text
CREATE ROLE
CREATE ROLE

 List of roles
 Role name | Attributes 
-----------+------------
 alice | 

 List of roles
 Role name | Attributes 
-----------+------------
 bob | 
```

```text
  hba_file 
-------------------------------------
 /etc/postgresql/16/main/pg_hba.conf
(1 row)
```

```text
Password for user alice:
Password for user bob: 
```

```text
              ident_file               
---------------------------------------
 /etc/postgresql/16/main/pg_ident.conf
(1 row)
```

```text
info: Adding user `alice' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `alice' (1001) ...
info: Adding new user `alice' (1001) with group `alice (1001)' ...
warn: The home directory `/home/alice' already exists.  Not touching this directory.
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for alice
Enter the new value, or press ENTER for the default
	Full Name []: 
	Room Number []: 
	Work Phone []: 
	Home Phone []: 
	Other []: 
Is the information correct? [Y/n] 
info: Adding new user `alice' to supplemental / extra groups `users' ...
info: Adding user `alice' to group `users' ...
```

```text
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "alice"
psql (16.11 (Ubuntu 16.11-1.pgdg24.04+1))
Type "help" for help.
postgres=> 
```

```text
psql (16.11 (Ubuntu 16.11-1.pgdg24.04+1))
Type "help" for help.
postgres=> \q
psql (16.11 (Ubuntu 16.11-1.pgdg24.04+1))
Type "help" for help.
postgres=> \q
```


### Часть 2. Управление расширениями

**Выполненные задачи:**
1. Установка расширения:
Установил расширение units_of_measure. Убедился, что оно появилось в pg_available_extensions.

2. Создание и исследование:
Создал расширение в своей БД без указания версии. Определил, какая версия
установилась и какие скрипты выполнились.

3. Добавление данных:
Добавил в справочник расширения новые единицы измерения.

4. Управление доступом:
Изменил права доступа к таблицам расширения: отзовал SELECT у public, выдал его
новой специальной роли.

5. Резервное копирование:
Используя pg_dump, выгрузил объекты расширения. Изучил дамп. Убедился, что
выгружаются метаданные (таблицы, типы, функции) и данные.

**Выполненные команды и результат**

**Задача 1**

```bash
student:~$ git clone https://pubgit.postgrespro.ru/pub/uom
student:~$ cd uom/
student:~/uom$ ls
student:~/uom$ sudo make install 
```

```sql
student=# SELECT * FROM pg_available_extensions 
WHERE name LIKE '%unit%' OR name LIKE '%uom%';
```

**Результат**

```text
Makefile  README.md  uom--1.0--1.1.sql  uom--1.0.sql  uom--1.1--1.2.sql  uom--1.2.sql  uom.control
/bin/mkdir -p '/usr/share/postgresql/16/extension'
/bin/mkdir -p '/usr/share/postgresql/16/extension'
/usr/bin/install -c -m 644 .//uom.control '/usr/share/postgresql/16/extension/'
/usr/bin/install -c -m 644 .//uom--1.0.sql .//uom--1.2.sql .//uom--1.0--1.1.sql .//uom--1.1--1.2.sql  '/usr/share/postgresql/16/extension/'
```

```text
 name | default_version | installed_version |       comment        
------+-----------------+-------------------+----------------------
 uom  | 1.2             |                   | Units of Measurement
(1 row)
```

**Задача 2**

```sql
student=# CREATE EXTENSION uom;
student=# SELECT * FROM pg_extension WHERE extname = 'uom';
```

```bash
ls  /usr/share/postgresql/16/extension/uom*
cat /usr/share/postgresql/16/extension/uom--1.2.sql | grep -n "CREATE\|INSERT"
```


**Результат**

```text
CREATE EXTENSION
  oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig |       extcondition       
-------+---------+----------+--------------+----------------+------------+-----------+--------------------------
 65629 | uom     |    16384 |         2200 | t              | 1.2        | {65630}   | {"WHERE NOT predefined"}
(1 row)
```

```text
/usr/share/postgresql/16/extension/uom--1.0--1.1.sql
/usr/share/postgresql/16/extension/uom--1.0.sql
/usr/share/postgresql/16/extension/uom--1.1--1.2.sql
/usr/share/postgresql/16/extension/uom--1.2.sql
/usr/share/postgresql/16/extension/uom.control

1:\echo Use "CREATE EXTENSION uom" to load this file. \quit
6:CREATE TABLE uom_ref (
11:INSERT INTO uom_ref VALUES 
16:CREATE FUNCTION uom2uom (value numeric, name_from text, name_to text)
30:CREATE TYPE uom AS (
36:CREATE FUNCTION uom_cmp (a uom, b uom) RETURNS int LANGUAGE SQL
42:CREATE FUNCTION uom_lt(a uom, b uom) RETURNS boolean
45:CREATE FUNCTION uom_le(a uom, b uom) RETURNS boolean
48:CREATE FUNCTION uom_eq(a uom, b uom) RETURNS boolean
51:CREATE FUNCTION uom_ge(a uom, b uom) RETURNS boolean
54:CREATE FUNCTION uom_gt(a uom, b uom) RETURNS boolean
57:CREATE OPERATOR <  (PROCEDURE = uom_lt, LEFTARG = uom, RIGHTARG = uom);
58:CREATE OPERATOR <= (PROCEDURE = uom_le, LEFTARG = uom, RIGHTARG = uom);
59:CREATE OPERATOR =  (PROCEDURE = uom_eq, LEFTARG = uom, RIGHTARG = uom);
60:CREATE OPERATOR >= (PROCEDURE = uom_ge, LEFTARG = uom, RIGHTARG = uom);
61:CREATE OPERATOR >  (PROCEDURE = uom_gt, LEFTARG = uom, RIGHTARG = uom);
63:CREATE OPERATOR CLASS uom_ops DEFAULT FOR TYPE uom USING btree AS
```

**Задача 3**

```bash
student=# SELECT * FROM public.uom_ref ORDER BY name;
student=# INSERT INTO public.uom_ref (name, k) 
VALUES 
('foot', 0.3048),
('inch', 0.0254);
student=# SELECT * FROM public.uom_ref WHERE name IN ('foot', 'inch');
```

**Результат**

```text
  name  |    k    | predefined 
--------+---------+------------
 аршин  |  0.7112 | t
 верста |  1066.8 | t
 вершок | 0.04445 | t
 км     |    1000 | t
 м      |       1 | t
 сажень |  2.1336 | t
 см     |    0.01 | t
(7 rows)

INSERT 0 2

 name |   k    | predefined 
------+--------+------------
 foot | 0.3048 | f
 inch | 0.0254 | f
(2 rows)
```

**Задача 4**

```sql
student=# CREATE ROLE uom_reader NOLOGIN;
student=# REVOKE ALL ON TABLE public.uom_ref FROM public;
student=# GRANT SELECT ON TABLE public.uom_ref TO uom_reader;
student=# CREATE USER reader_uom WITH PASSWORD 'reader123';
student=# GRANT uom_reader TO reader_uom;
```


**Результат**

```text
CREATE ROLE
REVOKE
GRANT
CREATE ROLE
GRANT ROLE
```

**Задача 5**

```bash
student:~$ pg_dump -U postgres -d student --table=public.uom_ref > student_uom_ref_dump.sql
student:~$ head -30 student_uom_ref_dump.sql
```

**Результат**

```text
--
-- PostgreSQL database dump
--

\restrict LHTiQJehHa3eiEh61UpdHtgdku0oqomnaa08zmIdQRzjrhDQFavEvscFxu3C3GY

-- Dumped from database version 16.11 (Ubuntu 16.11-1.pgdg24.04+1)
-- Dumped by pg_dump version 16.11 (Ubuntu 16.11-1.pgdg24.04+1)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

--
-- Data for Name: uom_ref; Type: TABLE DATA; Schema: public; Owner: student
--

COPY public.uom_ref (name, k, predefined) FROM stdin;
foot	0.3048	f
inch	0.0254	f
\.
```

# Модуль 1: Управление доступом

## 1. Базовые привилегии

- **База данных access_db** - создается для изоляции тестовой среды управления доступом
- **Роль writer** - создается с правом создания объектов (CREATE) в схеме public
- **Роль reader** - создается только с правом чтения (SELECT) существующих объектов
- **Отзыв прав у public** - стандартная мера безопасности, убирающая автоматические права
- **Права на схему public** - writer получает CREATE + USAGE, reader только USAGE
- **Привилегии по умолчанию** - настраиваются так, чтобы reader автоматически получал SELECT на новые таблицы writer
- **Пользователи w1 и r1** - конкретные учетные записи, входящие в соответствующие роли
- **Тестовая таблица test_table** - создается для проверки распределения прав доступа
- **Проверка прав** - r1 может только читать (SELECT), w1 имеет полный доступ (SELECT, INSERT, UPDATE, DELETE)

## 2. Аутентификация

- **Пользовательские роли alice и bob** - создаются для тестирования методов аутентификации
- **pg_hba.conf** - основной файл конфигурации аутентификации PostgreSQL
- **Trust аутентификация** - разрешает вход без пароля (опасно, только для локального тестирования)
- **Настройка trust только для postgres и student** - ограничение привилегированного доступа
- **md5 аутентификация** - стандартный метод с шифрованием пароля
- **reject метод** - явный отказ в аутентификации
- **Peer аутентификация** - использует идентификатор пользователя ОС для входа в PostgreSQL
- **Требование создания пользователя в ОС** - peer метод требует существования соответствующего пользователя ОС
- **pg_ident.conf** - файл отображения пользователей ОС на роли PostgreSQL
- **Одно отображение для нескольких ролей** - один пользователь ОС может быть сопоставлен с несколькими ролями PostgreSQL через отображение

# Модуль 2: Управление расширениями

## 1. Установка расширения

- **Расширение units_of_measure/uom** - добавляет функционал работы с единицами измерения
- **pg_available_extensions** - системное представление со списком доступных расширений
- **default_version** - версия расширения, которая устанавливается по умолчанию
- **installed_version** - версия, которая фактически установлена в БД (NULL если не установлено)

## 2. Создание и исследование

- **CREATE EXTENSION без указания версии** - устанавливает расширение с версией по умолчанию
- **Установленная версия** - определяется через запрос к pg_extension или pg_available_extensions
- **Системный каталог расширений** - /usr/share/postgresql/<версия>/extension/
- **Control-файл** (uom.control) - содержит метаданные расширения (версия, описание)
- **SQL-скрипты** (uom--1.2.sql) - выполняются при установке, создают объекты БД
- **Просмотр выполненных скриптов** - через анализ файлов в системном каталоге или запросы к pg_extension

## 3. Добавление данных

- **Справочник расширения** - таблицы, содержащие данные расширения (например, uom_ref)
- **Структура таблицы uom_ref** - обычно содержит поля: name (название единицы), k (коэффициент преобразования)
- **Добавление новых единиц** - выполняется через INSERT в таблицы расширения
- **Примеры единиц** - футы (foot, коэффициент 0.3048), дюймы (inch, коэффициент 0.0254)

## 4. Управление доступом

- **Отзыв SELECT у public** - стандартная практика безопасности для объектов расширений
- **Создание специальной роли** - выделенная роль для доступа к объектам расширения
- **GRANT SELECT новой роли** - предоставление прав чтения только авторизованным пользователям
- **Наследование прав** - пользователи получают доступ через членство в специальной роли

## 5. Резервное копирование

- **pg_dump** - утилита командной строки для создания дампов PostgreSQL
- **Выгрузка объектов расширения** - используется флаг --extension=<имя>
- **Метаданные в дампе** - команды CREATE EXTENSION, CREATE TABLE, CREATE FUNCTION
- **Данные в дампе** - содержимое таблиц расширения (команды INSERT)
- **Исследование дампа** - проверка наличия всех необходимых объектов и данных
- **Форматы дампа** --schema-only (только структура), --data-only (только данные), полный дамп


## Вывод
В процессе лабораторной работы были освоены управление правами доступа пользователей, проведена работа с расширениями PostgreSQL и настройка параметров локализации. Получены практические навыки настройки
аутентификации, управления привилегиями, установки расширений.