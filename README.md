# Postgre-DBA-2025-01 Занятие #08
Журналы

Домашнее задание

>Настройте выполнение контрольной точки раз в 30 секунд.
   ```sql
postgres=# alter system set checkpoint_timeout = 30;
ALTER SYSTEM
postgres=# Select pg_reload_conf();
 pg_reload_conf
----------------
 t

postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
   ```
Текущая позиция добавления WAL до запуска нагрузочного теста.
   ```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/2FCD0820
   ```
>10 минут c помощью утилиты pgbench подавайте нагрузку.
   ```sh
[root@altLinux-02 5]# pgbench -P 10 -T 600 postgres -U postgres
pgbench (15.10)
starting vacuum...end.
progress: 10.0 s, 749.8 tps, lat 1.333 ms stddev 0.196, 0 failed
progress: 20.0 s, 714.5 tps, lat 1.399 ms stddev 0.110, 0 failed
....
progress: 590.0 s, 723.5 tps, lat 1.382 ms stddev 0.110, 0 failed
progress: 600.0 s, 755.9 tps, lat 1.323 ms stddev 0.189, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
duration: 600 s
number of transactions actually processed: 444858
number of failed transactions: 0 (0.000%)
latency average = 1.348 ms
latency stddev = 0.166 ms
initial connection time = 4.798 ms
tps = 741.433828 (without initial connection time)
   ```
Текущая позиция добавления WAL после нагрузочного теста.
   ```sql
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 0/4C2C3CA0
   ```
>Измерьте, какой объем журнальных файлов был сгенерирован за это время.    
Оцените, какой объем приходится в среднем на одну контрольную точку.
   ```sql
postgres=# select pg_size_pretty(pg_size_bytes(('0/4C2C3CA0'::pg_lsn - '0/2FCD0820'::pg_lsn)::text));
 pg_size_pretty
----------------
 454 MB
   ```
   ```sh
[root@altLinux-02 5]# cat /var/lib/pgsql/data/log/postgresql-2025-03-09_145516.log | grep -c 'slot checkpoint'
21
   ```

