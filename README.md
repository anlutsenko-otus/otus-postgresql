# Домашняя работа: Работа с журналами
## Выполнение

1. *Настройте выполнение контрольной точки раз в 30 секунд.*

Изменил настройку checkpoint_timeout на 30 секунд и перезагрузил конфигурацию:
```console
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)

postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)

postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)
```

2. *10 минут c помощью утилиты pgbench подавайте нагрузку.*

Проинициилизировал pgbench
```console
otus@otus:~$ pgbench -i -U postgres postgres
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.36 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.25 s, vacuum 0.05 s, primary keys 0.06 s).
```

Запустил pgbench
```console
otus@otus:~$ pgbench -c8 -P 6 -T 600 -U postgres postgres
Password:
pgbench (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 6.0 s, 635.8 tps, lat 12.428 ms stddev 10.618
progress: 12.0 s, 579.0 tps, lat 13.819 ms stddev 9.376
progress: 18.0 s, 650.3 tps, lat 12.301 ms stddev 6.518
...
progress: 600.0 s, 646.5 tps, lat 12.376 ms stddev 6.958
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 376876
latency average = 12.735 ms
latency stddev = 7.638 ms
initial connection time = 65.850 ms
tps = 628.177219 (without initial connection time)
```

Общее число контрольных точек
```console
postgres=# SELECT
total_checkpoints,
seconds_since_start / total_checkpoints / 60 AS minutes_between_checkpoints
FROM
(SELECT
EXTRACT(EPOCH FROM (now() - pg_postmaster_start_time())) AS seconds_since_start,
(checkpoints_timed+checkpoints_req) AS total_checkpoints
FROM pg_stat_bgwriter
) AS sub;
 total_checkpoints | minutes_between_checkpoints
-------------------+-----------------------------
                57 |      0.23111920321637426833
(1 row)
```

Посмотрим, что лежит в pg_wal

```console
postgres=# create extension pageinspect;
CREATE EXTENSION
postgres=# select * from pg_ls_waldir();
           name           |   size   |      modification
--------------------------+----------+------------------------
 000000010000000000000022 | 16777216 | 2024-03-05 19:23:20+00
 000000010000000000000020 | 16777216 | 2024-03-05 19:22:19+00
 000000010000000000000021 | 16777216 | 2024-03-05 19:22:54+00
 00000001000000000000001F | 16777216 | 2024-03-05 19:27:05+00
(4 rows)
```

3. *Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.*

4 файла по 16MB, то есть на одну контрольную точку (всего 57) приходится чуть больше 1MB.

4. *Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?*

```console
postgres=# SELECT
total_checkpoints,
seconds_since_start / total_checkpoints / 60 AS minutes_between_checkpoints
FROM
(SELECT
EXTRACT(EPOCH FROM (now() - pg_postmaster_start_time())) AS seconds_since_start,
(checkpoints_timed+checkpoints_req) AS total_checkpoints
FROM pg_stat_bgwriter
) AS sub;
 total_checkpoints | minutes_between_checkpoints
-------------------+-----------------------------
                57 |      0.23111920321637426833
(1 row)
```

По данным в minutes_between_checkpoints видим, что checkpoint-ы выполнялись раньше чем раз в 30 секунд. Полагаю, что это произошло из-за повышенной нагрузки, которую генерировал pgbench.

5. *Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.*

Включаю асинхронный режим для wall

```console
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

postgres=# alter system set synchronous_commit = 'off';
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 row)
```

Запустил pgbench
```console
root@otus:~# pgbench -c8 -P 6 -T 600 -U postgres postgres
Password:
pgbench (14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
starting vacuum...end.
progress: 6.0 s, 3249.0 tps, lat 2.434 ms stddev 0.680
progress: 12.0 s, 3218.8 tps, lat 2.485 ms stddev 0.701
progress: 18.0 s, 3142.0 tps, lat 2.546 ms stddev 0.716
...
progress: 600.0 s, 3153.4 tps, lat 2.536 ms stddev 0.797
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 1915531
latency average = 2.505 ms
latency stddev = 0.720 ms
initial connection time = 65.888 ms
tps = 3192.798518 (without initial connection time)
```

Как и ожидалось, tps при в асинхронном режиме на порядок выше чем при синхронном: 628 против 3192.

6. *Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?*

Остановил и удалил старый кластер
```console
root@otus:~# pg_ctlcluster 14 main stop

root@otus:~# pg_dropcluster 14 main
```

Создал новый кластер с включёнными контрольными суммами
```console
root@otus:~# pg_createcluster 14 my_cluster -- --data-checksums
Creating new PostgreSQL cluster 14/my_cluster ...
/usr/lib/postgresql/14/bin/initdb -D /var/lib/postgresql/14/my_cluster --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/14/my_cluster ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster    Port Status Owner    Data directory                    Log file
14  my_cluster 5432 down   postgres /var/lib/postgresql/14/my_cluster /var/log/postgresql/postgresql-14-my_cluster.log
```

Проверим включены ли контрольные суммы:

```console
postgres=# show data_checksums;
 data_checksums
----------------
 on
(1 row)
```

Создадим таблицу. Вставим значения.

```console
postgres=# create table test (col int);
CREATE TABLE

postgres=# insert into test (col) select * from generate_series(1, 10);
INSERT 0 10
postgres=# select * from test;
 col
-----
   1
   2
   3
   4
   5
   6
   7
   8
   9
  10
(10 rows)
```

Определим расположение таблицы на диске

```console
postgres=# SELECT pg_relation_filepath('test');
 pg_relation_filepath
----------------------
 base/13761/16384
(1 row)
```

Остановим кластер

```console
root@otus:~# pg_ctlcluster 14 my_cluster stop
root@otus:~# pg_lsclusters
Ver Cluster    Port Status Owner    Data directory                    Log file
14  my_cluster 5432 down   postgres /var/lib/postgresql/14/my_cluster /var/log/postgresql/postgresql-14-my_cluster.log
```

Изменим файл
```console
root@otus:~# nano /var/lib/postgresql/14/my_cluster/base/13761/16384
```

При попытке запросить данные из таблицы получаем ошибку

```console
postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 57896 but expected 406
ERROR:  invalid page in block 0 of relation base/13761/16384
```

Меняем значение настройки ignore_checksum_failure на on. Но всё равно видим ошибку. Кластер старт-стопал.

```console
postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 57896 but expected 406
ERROR:  invalid page in block 0 of relation base/13761/16384
```

Сделал новую таблицу test1, заполнил аналогичными данными. Узнал её расположение

```console
postgres=# SELECT pg_relation_filepath('test1');
 pg_relation_filepath
----------------------
 base/13761/16387
 ```

 Изменил файл.

```console
root@otus:~# nano /var/lib/postgresql/14/my_cluster/base/13761/16387
```

Теперь в при попытке запросить данные вижу, что таблица пуста.

```console
postgres=# select * from test1;
 col
-----
(0 rows)
```

Выглядит так, что настройка ignore_checksum_failure применяется к таблицам во время их создания.