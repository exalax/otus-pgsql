# Домашнее задание к уроку 2

### Создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере.
Выбрал докер потому что есть в наличии и проще всего запустить.
```shell
docker run --name otus-pgsql --env=POSTGRES_PASSWORD=1234 -p 7432:5432 -d postgres:15
```

### Запустить везде psql из под пользователя postgres
```shell
psql -h localhost -p 7432 postgres postgres
```

### Выключить auto commit
```
# Session 1

postgres=# \set AUTOCOMMIT off
postgres=# \echo :AUTOCOMMIT
off
```

```
# Session 2

postgres=# \set AUTOCOMMIT off
postgres=# \echo :AUTOCOMMIT
off
```

### В первой сессии создать новую таблицу и наполнить ее данными
```postgresql
# Session 1

postgres=# CREATE TABLE persons (id SERIAL, first_name TEXT, second_name TEXT);
CREATE TABLE
postgres=*# INSERT INTO persons (first_name, second_name) VALUES ('ivan', 'ivanov');
INSERT 0 1
postgres=*# INSERT INTO persons(first_name, second_name) VALUES ('petr', 'petrov');
INSERT 0 1
postgres=*# COMMIT;
COMMIT
```

### Посмотреть текущий уровень изоляции
```postgresql
# Session 1

postgres=# SHOW TRANSACTION ISOLATION LEVEL;
transaction_isolation 
-----------------------
 read committed
(1 row)
```

```postgresql
# Session 2

postgres=# SHOW TRANSACTION ISOLATION LEVEL;
transaction_isolation 
-----------------------
 read committed
(1 row)
```

### Начать новую транзакцию в обеих сессиях с дефолтным (не меняя) уровнем изоляции
```postgresql
# Session 1

postgres=# BEGIN;
BEGIN
postgres=*#
```

```postgresql
# Session 2

postgres=# BEGIN;
BEGIN
postgres=*#
```

### В первой сессии добавить новую запись
```postgresql
# Session 1

postgres=*# INSERT INTO persons (first_name, second_name) VALUES ('sergey', 'sergeev');
INSERT 0 1
```

### Сделать select from persons во второй сессии
```postgresql
# Session 2

postgres=*# SELECT * FROM persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
Новая запись отсутствует в результате запроса потому что транзакция в первой сессии не завершена,
а уровень изоляции по умолчанию - `read committed`: аномалия `dirty read` отсутствует.

### Завершить первую транзакцию
```postgresql
# Session 1

postgres=*# COMMIT;
COMMIT
postgres=#
```

### Сделать select from persons во второй сессии
```postgresql
# Session 2

postgres=*# SELECT * FROM persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Теперь новая строка присутствует в результате запроса потому что транзакция в первой сессии завершена,
изменения зафиксированы. Уровень изоляции по умолчанию - `read committed`: аномалия `dirty read` отсутствует,
но мы получили аномалию `phantom read`.

### Завершить транзакцию во второй сессии
```postgresql
# Session 2

postgres=*# COMMIT;
COMMIT
postgres=#
```


### Начать новые, но уже repeatable read, транзации
```postgresql
# Session 1

postgres=# SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET
postgres=*#
```

```postgresql
# Session 2

postgres=# SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET
postgres=*#
```

### В первой сессии добавить новую запись
```postgresql
# Session 1

postgres=*# INSERT INTO persons (first_name, second_name) VALUES ('sveta', 'svetova');
INSERT 0 1
```

### Сделать select * from persons во второй сессии
```postgresql
# Session 2

postgres=*# SELECT * FROM persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Уровень изоляции `repeatable read`: аномалия `dirty read` отсутствует, поэтому
результат изменений незавершенной транзакции не виден.

### Завершить первую транзакцию
```postgresql
# Session 1

postgres=*# COMMIT;
COMMIT
postgres=#
```

### Сделать select * from persons во второй сессии
```postgresql
# Session 2

postgres=*# SELECT * FROM persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Уровень изоляции `repeatable read`: аномалия `phantom read` отсутствует, поэтому
результат изменений параллельной транзакции не виден.

### Завершить вторую транзакцию
```postgresql
# Session 2

postgres=*# COMMIT;
COMMIT
postgres=#
```

### Сделать select * from persons во второй сессии
```postgresql
# Session 2

postgres=*# SELECT * FROM persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
postgres=*# 
```
Предыдущая транзакция завершена, новая транзакция начинается с актуальным на момент старта состоянием.
Новая транзакция началась после завершения транзакции в первой сессии, поэтому новая запись присутствует.
