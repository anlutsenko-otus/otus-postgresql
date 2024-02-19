# Домашняя работа 2: Установка PostgreSQL

## Описание/Пошаговая инструкция выполнения домашнего задания:

* Создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом
* Поставить на нем Docker Engine
сделать каталог /var/lib/postgres
* Развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
* Развернуть контейнер с клиентом postgres
* Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
* Подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
* Удалить контейнер с сервером
* Создать его заново
* Подключится снова из контейнера с клиентом к контейнеру с сервером
* Проверить, что данные остались на месте

## Выполнение

Задание выполнялось в операционной системе **Microsoft Windows 11**

* Был скачан, установлен и запушен **Docker Desktop**
* В дирректории G:\otus был сохранён файл docker-compose.yaml

```yaml
version: '3.1'

volumes:
  pg_project:

services:
  pg_db:
    image: postgres:14
    restart: always
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=stage
    volumes:
      - pg_project:/var/lib/postgresql/data
    ports:
      - ${POSTGRES_PORT:-5433}:5432
```

* Через cmd в этой дирректории была выполнена команда *docker-compose up*
* Был скачан, установлен и запущен **DBeaver**
* Был осуществено подключение в базе данных через **DBeaver**:
    * Хост: localhost
    * Порт: 5433
    * База данных: postgres
    * Пользователь: postgres
    * Пароль: postgres
* Выполнены следующие команды:

```sql
create table names (id bigint, name text);
insert into names (id, name) values (1, 'John'), (2, 'Steve');
select * from names;
```

* Через **Docker Desktop** был удалён контейнер с сервером
* Через cmd в этой дирректории была выполнена команда *docker-compose up*
* В **DBeaver** была выполнена команда:

```sql
create table names (id bigint, name text);
insert into names (id, name) values (1, 'John'), (2, 'Steve');
select * from names;
```

Результат выполнения запроса:
| id | name  |
|----|-------|
| 1  | John  |
| 2  | Steve |