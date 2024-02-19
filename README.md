# Домашняя работа 1: SQL и реляционные СУБД. Введение в PostgreSQL

## Описание/Пошаговая инструкция выполнения домашнего задания:

* Создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере
* Далее создать инстанс виртуальной машины с дефолтными параметрами
* Добавить свой ssh ключ в metadata ВМ
* Зайти удаленным ssh (первая сессия), не забывайте про ssh-add
* Поставить PostgreSQL
* Зайти вторым ssh (вторая сессия)
* Запустить везде psql из под пользователя postgres
* Выключить auto commit
* В первой сессии новую таблицу и наполнить ее данными create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
* Посмотреть текущий уровень изоляции: show transaction isolation level
* Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
* В первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
* Сделать select from persons во второй сессии. Видите ли вы новую запись и если да то почему?
* Завершить первую транзакцию - commit;
* Сделать select from persons во второй сессии. Видите ли вы новую запись и если да то почему?
* Завершите транзакцию во второй сессии
* Начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
* В первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
* Сделать select* from persons во второй сессии. Видите ли вы новую запись и если да то почему?
* Завершить первую транзакцию - commit;
* Сделать select from persons во второй сессии. Видите ли вы новую запись и если да то почему?
* Завершить вторую транзакцию
* Сделать select * from persons во второй сессии. Видите ли вы новую запись и если да то почему?

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
* Был осуществено подключение в базе данных через **DBeaver**:
    * Хост: localhost
    * Порт: 5433
    * База данных: postgres
    * Пользователь: postgres
    * Пароль: postgres
* Через *Windows PoweShell* были выполнены команды:
    * docker exec -it otus-pg_db-1 bash
    * psql -h localhost -U postgres
    * \set AUTOCOMMIT off
    * \echo :AUTOCOMMIT - Результат: off
* Выполнены следующие команды:

```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
show transaction isolation level;
```

* Результат: создана и заполнена таблицы persons. Команда *show transaction isolation level* вернула **read committed**

*начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
сделать select from persons во второй сессии
видите ли вы новую запись и если да то почему?*

* Не вижу, уровень *read commited* запрещает читать не зафиксированные данные

*завершить первую транзакцию - commit;
сделать select from persons во второй сессии
видите ли вы новую запись и если да то почему?*

* Вижу, так как изменения в первой транзакции были зафиксированы

*начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
сделать select* from persons во второй сессии*
видите ли вы новую запись и если да то почему?*

* Не вижу, уровень *repeatable read* запрещает читать не зафиксированные данные

*завершить первую транзакцию - commit;
сделать select from persons во второй сессии
видите ли вы новую запись и если да то почему?*

* Не вижу, уровень *repeatable read* не позволяет видеть изменения внесённые другими транзацакциями после начала данной транзакции

*завершить вторую транзакцию
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?*

* Вижу, так как была начата новая транзакция после того, как изменения в первой транзакции были зафиксированы