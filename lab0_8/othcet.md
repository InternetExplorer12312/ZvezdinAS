# Отчет по лабораторной работе №8
# Резервное копирование и управление доступом

**Дата:** 2025-10-09 
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Звездин Алексей Сергеевич  

## Цель работы
Освоить методы резервного копирования и восстановления данных в PostgreSQL,
включая логическое и физическое копирование, а также углубить навыки управления правами доступа
пользователей.

## Практическая часть

### Часть 1: Управление доступом (Повторение и закрепление)

**Выполненные задачи:** 

1. Настройка привилегий: Повторил задания из Практики по управлению доступом (создание БД,
ролей writer/reader, настройка GRANT/REVOKE, проверка доступа w1/r1).

2. Настройка аутентификации (Практика+): Повторил задания по настройке pg_hba.conf для
ролей alice и bob с использованием методов trust, reject и peer.

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

### Часть 2: Логическое резервное копирование

**Выполненные задачи:** 
1. Простой дамп и восстановление:
Создал БД backup_db и таблицу с данными.
Сделал логическую копию с помощью pg_dump.
Удалил БД и восстановите ее из копии. Убедился в целостности данных.

2. Параллельный дамп:
Создал несколько БД с различными объектами.
Сделал копию глобальных объектов через pg_dumpall --globals-only.
Сделал дампы каждой БД с помощью pg_dump в параллельном режиме (-j).

3. Восстановление кластера: Восстановил весь кластер на "другом сервере" (виртуально, в другой
каталог или другую ВМ) из созданных резервных копий.

4. Проблемы при загрузке (Практика+): Создал данные и параметры COPY
(например, несовпадение кодировок, делимитеров), которые приведут к ошибке при загрузке
дампа.

**Задача 1**
```bash
student:~$ sudo -u postgres psql
```

```sql
postgres=# CREATE DATABASE backup_db;
\c backup_db
backup_db=# CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    position VARCHAR(100),
    salary DECIMAL(10,2)
);
backup_db=# INSERT INTO employees (name, position, salary) VALUES
('Иван Иванов', 'Менеджер', 75000.00),
('Мария Петрова', 'Разработчик', 90000.00),
('Алексей Сидоров', 'Аналитик', 65000.00);
backup_db=# \q
```
```bash
student:~$ pg_dump -U postgres -d backup_db -f backup_db.sql
student:~$ sudo -u postgres psql -c "DROP DATABASE backup_db;"
student:~$ sudo -u postgres psql -c "CREATE DATABASE backup_db;"
student:~$ psql -U postgres -d backup_db -f backup_db.sql
student:~$ sudo -u postgres psql -d backup_db -c "SELECT * FROM employees;"
```

**Результат**

```text
psql (16.11 (Ubuntu 16.11-1.pgdg24.04+1))
Type "help" for help.
```

```text
CREATE DATABASE
You are now connected to database "backup_db" as user "postgres".
CREATE TABLE
INSERT 0 3
```

```text
DROP DATABASE
CREATE DATABASE
SET
SET
SET
SET
SET
 set_config 
------------
 
(1 row)

SET
SET
SET
SET
SET
SET
CREATE TABLE
ALTER TABLE
CREATE SEQUENCE
ALTER SEQUENCE
ALTER SEQUENCE
ALTER TABLE
COPY 3
 setval 
--------
      3
(1 row)
ALTER TABLE
 id |      name       |  position   |  salary  
----+-----------------+-------------+----------
  1 | Иван Иванов     | Менеджер    | 75000.00
  2 | Мария Петрова   | Разработчик | 90000.00
  3 | Алексей Сидоров | Аналитик    | 65000.00
(3 rows)
```

**Задача 2**

```sql
student:~$ sudo -u postgres psql


postgres=# CREATE DATABASE db1;
CREATE DATABASE db2;
CREATE DATABASE db3;
postgres=# \c db1
db1=# CREATE TABLE clients (id SERIAL, name TEXT);
INSERT INTO clients (name) VALUES ('Клиент 1'), ('Клиент 2');
db1=# \c db2  
db2=# CREATE TABLE products (id SERIAL, product_name TEXT, price NUMERIC);
INSERT INTO products VALUES (1, 'Товар A', 1000), (2, 'Товар B', 2000);
db2=# \c db3
db3=# CREATE TABLE orders (id SERIAL, client_id INT, amount NUMERIC);
INSERT INTO orders VALUES (1, 1, 5000), (2, 2, 3000);
```

```bash
student:~$ pg_dump -U postgres -d db1 -j 2 -F d -f db1_parallel
student:~$ pg_dump -U postgres -d db2 -j 2 -F d -f db2_parallel
student:~$ pg_dump -U postgres -d db3 -j 2 -F d -f db3_parallel
```

**Результат**

```text
psql (16.11 (Ubuntu 16.11-1.pgdg24.04+1))
Type "help" for help.
CREATE DATABASE
CREATE DATABASE
CREATE DATABASE
You are now connected to database "db1" as user "postgres".
CREATE TABLE
INSERT 0 2
You are now connected to database "db2" as user "postgres".
CREATE TABLE
INSERT 0 2
You are now connected to database "db3" as user "postgres".
CREATE TABLE
INSERT 0 2
```


**Задача 3**

```bash
student:~$ sudo -u postgres psql -c "CREATE DATABASE db1_restore;"
student:~$ sudo -u postgres psql -c "CREATE DATABASE db2_restore;"
student:~$ sudo -u postgres psql -c "CREATE DATABASE db3_restore;"
student:~$ pg_restore -U postgres -d db1_restore -j 2 db1_parallel/
student:~$ pg_restore -U postgres -d db2_restore -j 2 db2_parallel/
student:~$ pg_restore -U postgres -d db3_restore -j 2 db3_parallel/
```

**Результат**

```text
CREATE DATABASE
CREATE DATABASE
CREATE DATABASE
```


**Задача 4**

```sql
student=# CREATE TABLE type_error (x INT);                    
student=# \! echo "text" > /tmp/minimal.csv              
student=# COPY minimal_error FROM '/tmp/minimal.csv' WITH (FORMAT csv);
```

**Результат**

```text
CREATE TABLE
ERROR:  invalid input syntax for type integer: "text"
CONTEXT:  COPY minimal_error, line 1, column x: "text"
```

# Модуль 2: Логическое резервное копирование

## 1. Простой дамп и восстановление

- **Логическое резервное копирование** - сохранение структуры и данных БД в виде SQL-команд или специальных форматов (custom, directory)
- **pg_dump** - утилита PostgreSQL для создания логических дампов отдельных баз данных, поддерживает различные форматы вывода
- **SQL-формат** - текстовый формат, содержащий SQL-команды для воссоздания объектов и данных, читаем человеком
- **Custom формат** - бинарный формат, поддерживающий сжатие, параллельное восстановление и выборочное восстановление объектов
- **Directory формат** - создает директорию с файлами (по одному на таблицу + метаданные), позволяет параллельный дамп
- **Восстановление дампа** - выполнение SQL-команд из дампа с помощью psql (для SQL формата) или pg_restore (для custom/directory форматов)
- **Целостность данных** - проверяется сравнением количества записей, контрольных сумм или хешей данных до и после восстановления

## 2. Параллельный дамп

- **Параллельный дамп (-j)** - позволяет использовать несколько параллельных соединений для дампа/восстановления, ускоряя процесс для больших БД
- **Ограничения параллелизма** - работает только с форматами directory (-Fd) и custom (-Fc), не поддерживается для SQL формата
- **Глобальные объекты** - роли, табличные пространства, настройки базы данных, которые не принадлежат конкретной БД
- **pg_dumpall --globals-only** - создает дамп только глобальных объектов кластера PostgreSQL
- **Оптимальное количество потоков** - обычно 2-4 потока на ядро процессора, зависит от нагрузки на диск и сеть
- **Преимущества параллельного дампа** - значительное ускорение для БД с большим количеством таблиц, особенно на системах с быстрыми дисками

## 3. Восстановление кластера

- **Кластер PostgreSQL** - совокупность всех баз данных, работающих под управлением одного экземпляра PostgreSQL
- **Восстановление на другой сервер** - процесс переноса всей структуры и данных на новое оборудование или в новую среду
- **Директория данных (PGDATA)** - каталог, содержащий все файлы БД, конфигурацию и журналы транзакций
- **Разделение по портам** - для запуска нескольких кластеров на одном сервере необходимо использовать разные порты (5432, 5433 и т.д.)
- **Этапы восстановления кластера**:
  1. Восстановление глобальных объектов (роли, табличные пространства)
  2. Создание пустых баз данных
  3. Восстановление схем и данных в каждую БД
  4. Настройка прав доступа и привилегий
- **Подходы к восстановлению**:
  - Использование другой директории PGDATA
  - Запуск на другом порту
  - Использование Docker-контейнеров
  - Настройка физической репликации

## 4. Проблемы при загрузке

- **Несовпадение кодировок** - ошибки возникают при попытке загрузить данные в кодировке, отличной от кодировки базы данных
- **Разделители (делимитеры)** - несоответствие указанного разделителя фактическому формату CSV файла
- **Несоответствие типов данных** - попытка загрузить строковые данные в числовые поля или наоборот
- **Нарушение ограничений** - дублирование первичных ключей, нарушение внешних ключей, проверочных ограничений
- **Нехватка столбцов** - в CSV файле меньше или больше столбцов, чем в целевой таблице
- **Экранирование специальных символов** - неправильное экранирование кавычек, запятых, переносов строк в CSV
- **Обработка NULL значений** - различия в представлении NULL значений (пустая строка, \N, NULL)
- **Формат даты и времени** - несоответствие формата даты в файле и настроек формата даты в PostgreSQL
- **Ошибки прав доступа** - недостаточные права на чтение файла или запись в таблицу
- **Прерывание загрузки** - частичная загрузка данных при ошибке, требующая отката или очистки
- **Методы диагностики** - использование опции LOG ERRORS, просмотр логов PostgreSQL, проверка файлов перед загрузкой


## Вывод
В ходе лабораторной работы были освоены методы резервного копирования и восстановления данных в PostgreSQL,
включая логическое и физическое копирование, а также углублены навыки управления правами доступа
пользователей.