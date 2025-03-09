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
Всего записано 454 MByte, контрольных точек 21. Т.е. 21.6 МB на точку.
> Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию.    
Почему так произошло?
   ```sh
[root@altLinux-02 5]# cat /var/lib/pgsql/data/log/postgresql-2025-03-09_145516.log | grep 'slot checkpoint'
2025-03-09 18:57:05.438 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 18:57:35.052 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 18:58:05.062 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 18:58:35.078 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 18:59:05.110 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 18:59:35.037 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:00:05.078 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:00:35.019 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:01:05.046 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:01:35.078 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:02:05.101 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:02:35.038 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:03:05.066 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:03:35.099 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:04:05.023 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:04:35.057 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:05:05.093 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:05:35.013 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:06:05.033 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:06:35.070 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
2025-03-09 19:08:05.083 MSK [5212] ОТЛАДКА:  performing replication slot checkpoint
   ```
Все контрольные точки записаны. ИНтервал 30 секунд. Ошибок в логе нет.    
Все хорошо. Нет другой нагрузки в базе. СУБД развернута на SSD.
> Сравните tps в синхронном/асинхронном режиме утилитой pgbench.

Синхроная запись
   ```sql
postgres=# show synchronous_commit ;
 synchronous_commit
--------------------
 on
   ```
   ```sh
[root@altLinux-02 5]# pgbench -T 60 postgres -U postgres
pgbench (15.10)
starting vacuum...end.
...
latency average = 1.314 ms
initial connection time = 4.871 ms
tps = 761.155095 (without initial connection time)

   ```
Aсинхроная запись
   ```sql
postgres=# show synchronous_commit ;
 synchronous_commit
--------------------
 off
   ```
   ```sh
[root@altLinux-02 5]# pgbench -T 60 postgres -U postgres
[root@altLinux-02 5]# pgbench -T 60 postgres -U postgres
pgbench (15.10)
...
latency average = 1.100 ms
initial connection time = 4.729 ms
tps = 909.069907 (without initial connection time)
   ```
В асинхронном режиме выполняется **910 tps**, синхронном **760 tps**. Прирост порядка 20%

