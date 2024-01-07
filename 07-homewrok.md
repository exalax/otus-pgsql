# Домашнее задание к уроку 7

### Создайте новый кластер PostgresSQL
Создаю контейнер с кластером и возможностью подключения извне.
```shell
docker run --name otus-pgsql --env=POSTGRES_PASSWORD=1234 -p 7432:5432 -d postgres:15
```

### Зайдите в созданный кластер под пользователем postgres
```shell
psql -U postgres -h localhost -p 7432 -W
Password: 
psql (16.1, server 15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=# 
```

### Создайте новую базу данных "testdb"
```postgresql
postgres=# CREATE DATABASE "testdb";
CREATE DATABASE 
```

### Зайдите в созданную базу данных под пользователем postgres
```postgresql
postgres=# \c testdb
Password: 
psql (16.1, server 15.5 (Debian 15.5-1.pgdg120+1))
You are now connected to database "testdb" as user "postgres".
```

### Создайте новую схему testnm
```postgresql
testdb=# CREATE SCHEMA "testnm";
CREATE SCHEMA
```

### Создайте новую таблицу t1 с одной колонкой c1 типа integer
```postgresql
testdb=# CREATE TABLE "testnm"."t1" ("c1" integer);
CREATE TABLE
```

### Вставьте строку со значением c1=1
```postgresql
testdb=# INSERT INTO "testnm"."t1" ("c1") VALUES (1);
INSERT 0 1
```

### Создайте новую роль readonly
```postgresql
testdb=# CREATE ROLE "readonly";
CREATE ROLE
```

### Дайте новой роли право на подключение к базе данных "testdb"
```postgresql
testdb=# GRANT CONNECT ON DATABASE "testdb" TO "readonly";
GRANT
```

### Дайте новой роли право на использование схемы "testnm"
```postgresql
testdb=# GRANT USAGE ON SCHEMA "testnm" TO "readonly";
GRANT
```

### Дайте новой роли право на select для всех таблиц схемы "testnm"
```postgresql
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA "testnm" TO "readonly";
GRANT
```

### Создайте пользователя testread с паролем test123
```postgresql
testdb=# CREATE USER "testread" WITH PASSWORD 'test123';
CREATE ROLE
```

### Дайте роль readonly пользователю testread
```postgresql
testdb=# GRANT "readonly" to "testread";
GRANT ROLE
```

### Зайдите под пользователем testread в базу данных testdb
```shell
psql -U testread -h localhost -p 7432 -W testdb
Password: 
psql (16.1, server 15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

testdb=>
```

### Сделайте select * from t1;
```postgresql
testdb=> SELECT * FROM "t1";
ERROR:  relation "t1" does not exist
LINE 1: SELECT * FROM "t1";
```

Таблица `t1` не найдена. Я её создавал в схеме `testnm`. Не по шпаргалке, просто так понял задание.
В `search path` схема `testnm` тоже не входит:
```postgresql
testdb=> SHOW search_path;
   search_path   
-----------------
 "$user", public
(1 row)
```

В текущей схеме таблиц тоже нет:
```postgresql
testdb=> SELECT current_schema();
current_schema 
----------------
 public
(1 row)

testdb=> \dt
Did not find any relations.
```

Дальше пропущу несколько пунктов, потому что сразу создал таблицу в нужной схеме.

### Сделайте "select * from testnm.t1;"
```postgresql
testdb=> select * from testnm.t1;
 c1 
----
  1
(1 row)
```

При явном указании схемы таблица и данные нашлись, права доступа дали возможность сделать `SELECT`.
Непонятна формулировка вопросов, в чате осталось без ответа:
> как сделать так чтобы такое больше не повторялось?
> сделайте select * from testnm.t1;
> получилось?
> есть идеи почему? если нет - смотрите шпаргалку
> сделайте select * from testnm.t1;
> получилось?
> ура!

Возможно, вопрос про то, что нужно или выбирать нужную схему по-умолчанию, или явно указывать имя схемы при создании таблицы.  


### Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```postgresql
testdb=> SELECT current_schema();
 current_schema 
----------------
 public
(1 row)

testdb=> create table t2(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
                    ^
testdb=> 
```

### А как так? Нам же никто прав на создание таблиц и insert в них под ролью readonly? 

Судя по вопросу, ожидалось, что создастся таблица и в неё добавится строка, но этого не произошло.
В 15-й версии [поведение изменилось](https://www.postgresql.org/about/news/postgresql-15-released-2526/):
> PostgreSQL 15 also revokes the CREATE permission from all users except a database owner from the public (or default) schema.


### Есть идеи как убрать эти права?
Для начала, выдадим права всем ролям на создание таблиц в схеме `public`:
```postgresql
testdb=# GRANT CREATE ON SCHEMA public TO PUBLIC;
GRANT
```

Теперь таблица создаётся и данные добавляются:
```postgresql
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
testdb=> SELECT * FROM t2;
 c1 
----
  2
(1 row)
```

Отберём права у всех подряд (роль `PUBLIC`) и попробуем создать новую таблицу:
```postgresql
testdb=# REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE
``` 

Теперь создание таблиц в `public` снова не работает:
```postgresql
testdb=> create table t3(c1 integer); insert into t3 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
ERROR:  relation "t3" does not exist
LINE 1: insert into t3 values (2);
                    ^
```

### Расскажите что получилось и почему

В схеме `public` до 15-й версии специальной роли `public`, которая включает в себя все роли, в том числе созданные позже,
выдавалось разрешение на создание таблиц. Собственно, вся "магия" - это специальная роль `public`.
