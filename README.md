# Домашняя работа: Механизм блокировок
## Выполнение

1. *Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.*

Для настройки сервера выполнил команды (поставил 20s, чтобы убедиться, что изменил именно нужный параметр):
```console
postgres=# alter system set deadlock_timeout = '20s';
ALTER SYSTEM

postgres=# alter system set log_lock_waits = 'on';
ALTER SYSTEM

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

Чтобы видеть долгие блокировки нужно поменять значение *log_lock_waits* на on (по умолчанию off), а также установить время ожидания в настройке *deadlock_timeout*. После этого нужно перегрузить конфиг с помошью *pg_reload_conf*.

Чтобы увидеть лог, в одной консоли выполнил следующие команды:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 1 where c1 = 1;
UPDATE 1
```

В другой консоли выполнил:
```console
postgres=# update test set c1 = 1 where c1 = 1;
```

Update завис.

Через 20 sec в логах появилась запись.
```console
sudo nano /var/log/postgresql/postgresql-15-main.log

2024-03-04 19:15:50.308 UTC [13044] postgres@postgres LOG:  process 13044 still waiting for ShareLock on transaction 137718 after 20000.135 ms
2024-03-04 19:15:50.308 UTC [13044] postgres@postgres DETAIL:  Process holding the lock: 12952. Wait queue: 13044.
2024-03-04 19:15:50.308 UTC [13044] postgres@postgres CONTEXT:  while updating tuple (442,110) in relation "test"
2024-03-04 19:15:50.308 UTC [13044] postgres@postgres STATEMENT:  update test set c1 = 1 where c1 = 1;
```

2. *Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.*

Сначала в 3-х консолях посмотрим pid наших сессий.

1-ая
```console
postgres=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          13271
(1 row)
```

2-ая
```console
postgres=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          13347
(1 row)
```

3-я
```console
postgres=# update test set c1 = 1 where c1 = 1;
UPDATE 1
postgres=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          13410
(1 row)
```

Во всех 3-х консолях выполнил команду:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 1 where c1 = 1;
```
Создал view
```console
postgres=# CREATE VIEW locks_v AS
SELECT pid,
       locktype,
       CASE locktype
         WHEN 'relation' THEN relation::regclass::text
         WHEN 'transactionid' THEN transactionid::text
         WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text
       END AS lockid,
       mode,
       granted
FROM pg_locks
WHERE locktype in ('relation','transactionid','tuple')
AND (locktype != 'relation' OR relation = 'test'::regclass);
CREATE VIEW
```
Запросил данные из этой view

```console
postgres=# select * from locks_v
;
  pid  |   locktype    |  lockid  |       mode       | granted
-------+---------------+----------+------------------+---------
 13410 | relation      | test     | RowExclusiveLock | t
 13347 | relation      | test     | RowExclusiveLock | t
 13271 | relation      | test     | RowExclusiveLock | t
 13347 | tuple         | test:118 | ExclusiveLock    | t
 13410 | tuple         | test:118 | ExclusiveLock    | f
 13410 | transactionid | 137729   | ExclusiveLock    | t
 13271 | transactionid | 137727   | ExclusiveLock    | t
 13347 | transactionid | 137728   | ExclusiveLock    | t
 13347 | transactionid | 137727   | ShareLock        | f
(9 rows)

postgres=#
```

Все 3 сессии имеют блокировку отношений RowExclusiveLock на таблицу test.
Сессии 13347 и 13410 были начаты после сессии 13271, поэтому они обе имеют блокировку строки (tuple) ExclusiveLock. Транзакция в сессии 13410 была начата позже транзакции в сессии 13347, поэтому в сессии 13347 мы имеем granted равный t, а в сессии 13410 granted равный f. Это означает, что транзакция в сессии 13410 выполнится после того, как закончится транзакция в сессии 13347.

3. *Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?*

Для начала получим pid сессий.

1-ая
```console
postgres=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          13860
(1 row)
```

2-ая
```console
postgres=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          13865
(1 row)
```

3-я
```console
postgres=# update test set c1 = 1 where c1 = 1;
UPDATE 1
postgres=# SELECT pg_backend_pid();
 pg_backend_pid
----------------
          13870
(1 row)
```

1. В первой сессии выполним:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 1 where c1 = 1;
UPDATE 1
```
2. Во второй сессии выполним:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 2 where c1 = 2;
UPDATE 1
```
3. В третьей сессии выполним:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 3 where c1 = 3;
UPDATE 1
```
4. В первой сессии выполним:
```console
postgres=*# update test set c1 = 1 where c1 = 2;
```
5. Во второй сессии выполним:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 2 where c1 = 3;
```
6. В третьей сессии выполним:
```console
postgres=# begin;
BEGIN
postgres=*# update test set c1 = 3 where c1 = 1;
```

Все 3 транзакции оказываются взаимозаблокированы: первая транзакция пытается обновить строку обновляемую во второй транзакции, вторая транзакция пытается обновить строку обновляемую в третьей транзакции, треться транзакция пытается обновить строку обновляемую в первой транзакции.

Через 20 секунд получаем сообщение о deadlock-e.

```console
ERROR:  deadlock detected
DETAIL:  Process 13860 waits for ShareLock on transaction 137731; blocked by process 13865.
Process 13865 waits for ShareLock on transaction 137732; blocked by process 13870.
Process 13870 waits for ShareLock on transaction 137730; blocked by process 13860.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "test"
```

Посмотрим содержимое логов

```console
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres LOG:  process 13860 detected deadlock while waiting for ShareLock on transaction 137731 after 20000.083 ms2024-03-04 20:10:30.301 UTC [13860] postgres@postgres DETAIL:  Process holding the lock: 13865. Wait queue: .
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test"
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres STATEMENT:  update test set c1 = 1 where c1 = 2;
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres ERROR:  deadlock detected
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres DETAIL:  Process 13860 waits for ShareLock on transaction 137731; blocked by process 13865.
        Process 13865 waits for ShareLock on transaction 137732; blocked by process 13870.
        Process 13870 waits for ShareLock on transaction 137730; blocked by process 13860.
        Process 13860: update test set c1 = 1 where c1 = 2;
        Process 13865: update test set c1 = 2 where c1 = 3;
        Process 13870: update test set c1 = 3 where c1 = 1;
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres HINT:  See server log for query details.
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test"
2024-03-04 20:10:30.301 UTC [13860] postgres@postgres STATEMENT:  update test set c1 = 1 where c1 = 2;
```

Здесь можно увидеть сам deadlock, какие pid привели к его появлению и какие запросы были выполнены. По этой информации можно восстановить цепочку событий, чтобы понять почему произошёл deadlock.

4. *Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?*

Из предложения задания следует, что могут, но не очень понятно как это реализовать, всякий раз, когда я пытался выполнить UPDATE в нескольких транзакциях, транзакция запущеная позже вставала на ShareLock ожидания завершения первой транзакции.