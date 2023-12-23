# Домашнее задание к уроку 3

### Сделать каталог /var/lib/postgres
Создаю каталог в домашней директории, а не в системных директориях, чтобы не словить проблемы с правами доступа. 
```shell
% cd ~/dev/learn/otus-pgsql
% mkdir data
```

### Развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
Сначала создаю общую для контейнеров сервера и клиента сеть:
```shell
% docker network create otus-pg-net
0b811ed39124da4e599ceefd68b97447994ba761f2becb00d30c64b177e219c4
```

Создаю контейнер с сервером:
```shell
% docker run --name otus-pgsql --network otus-pg-net -h otus-pg-server --env=POSTGRES_PASSWORD=1234 -e PGDATA=/var/lib/postgresql/data/pgdata -p 7432:5432 -v ~/dev/learn/otus-pgsql/data:/var/lib/postgresql/data -d postgres:15
ae79e0e86aa92fb089753ea0e1e540598e27f9c0f97766476f5c0a58b721dcaf
```

### Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
Создаю контейнер с клиентом и подключаюсь к контейнеру с сервером:
```shell
% docker run --rm -it --network otus-pg-net postgres:15 psql -h otus-pg-server -p 5432 -U postgres
Password for user postgres: 
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=# 
```

Создаю таблицу с данными:
```postgresql
postgres=# CREATE TABLE accounts ("id" BIGSERIAL PRIMARY KEY NOT NULL, "name" TEXT NOT NULL);
CREATE TABLE
                                              ^
postgres=# INSERT INTO "accounts" (name) VALUES ('abc'), ('def');
INSERT 0 2

postgres=# SELECT * FROM "accounts";
 id | name 
----+------
  1 | abc
  2 | def
(2 rows)

postgres=#
```

### Подключится к контейнеру с сервером с ноутбука/компьютера извне
Подключаюсь к контейнеру сервера по проброшенному порту:
```shell
% psql -h localhost -p 7432 -U postgres
Password for user postgres: 
psql (16.1, server 15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=# SELECT * FROM "accounts";
 id | name 
----+------
  1 | abc
  2 | def
(2 rows)

postgres=# 
```

### Удалить контейнер с сервером
```shell
% docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                    NAMES
ae79e0e86aa9   postgres:15   "docker-entrypoint.s…"   15 minutes ago   Up 15 minutes   0.0.0.0:7432->5432/tcp   otus-pgsql
% docker stop ae79e0e86aa9
ae79e0e86aa9
% docker rm ae79e0e86aa9  
ae79e0e86aa9
% docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED       STATUS                  PORTS     NAMES
```

### Создать его заново
```shell
% docker run --name otus-pgsql --network otus-pg-net -h otus-pg-server --env=POSTGRES_PASSWORD=1234 -e PGDATA=/var/lib/postgresql/data/pgdata -p 7432:5432 -v ~/dev/learn/otus-pgsql/data:/var/lib/postgresql/data -d postgres:15
9695b534c57db71bf54097a1ed7af7c9a2f99602239b8c1586e7bb0d54785549
```

### Подключится снова из контейнера с клиентом к контейнеру с сервером
```shell
% docker run --rm -it --network otus-pg-net postgres:15 psql -h otus-pg-server -p 5432 -U postgres
Password for user postgres: 
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=#
```

### Проверить, что данные остались на месте
```postgresql
postgres=# SELECT * FROM "accounts";
 id | name 
----+------
  1 | abc
  2 | def
(2 rows)

postgres=#
```

### Проблемы
Данные не сохранялись между перезапусками. На [странице образа на dockerhub](https://hub.docker.com/_/postgres) нашёл:
> **Important Note**: when mounting a volume to /var/lib/postgresql, the /var/lib/postgresql/data path is a local
> volume from the container runtime, thus data is not persisted on the mounted volume.

Поэтому добавилась дополнительная env-переменная.
