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
При асинхронной записи скрорость работы БС выше, т.к. после заверешения транзации система не ждет пока будет завершена запись, запись происходит в фоновом режиме.
В асинхронном режиме выполняется **910 tps**, синхронном **760 tps**. Прирост порядка 20%. 
СУБД развернута на SSD, на HDD прирост был бы существеннее.


>Создайте новый кластер с включенной контрольной суммой страниц.
   ```sh
[musson@AltLinux-03 ~]$ sudo -u postgres initdb --data_checksums -D /var/lib/pgsql/data/
Файлы, относящиеся к этой СУБД, будут принадлежать пользователю "postgres".
От его имени также будет запускаться процесс сервера.

Кластер баз данных будет инициализирован с локалью "ru_RU.UTF-8".
Кодировка БД по умолчанию, выбранная в соответствии с настройками: "UTF8".
Выбрана конфигурация текстового поиска по умолчанию "russian".

Контроль целостности страниц данных включён.
   ```
>Создайте таблицу. Вставьте несколько значений.
   ```sql
postgres=# create table test_text(t text);
CREATE TABLE
postgres=# insert into test_text select 'строка '||s.id from generate_series(1,500) as s(id);
INSERT 0 500
postgres=# show data_checksums;
 data_checksums
----------------
 on
postgres=# select pg_relation_filepath('test_text');
 pg_relation_filepath
----------------------
 base/5/16387

   ```
>Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло?
   ```sh
[root@AltLinux-03 ~]# systemctl stop postgresql.service
[root@AltLinux-03 ~]# dd if=/dev/zero of=/var/lib/pgsql/data/base/5/16387 oflag=dsync conv=notrunc bs=1 count=8
8+0 записей получено
8+0 записей отправлено
8 байт скопировано, 0,00262769 s, 3,0 kB/s
   ```sql
postgres=# select * from test_text limit 10;
ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 30265, а ожидалась - 55324
ОШИБКА:  неверная страница в блоке 0 отношения base/5/16387
   ```
**Включен контроль целостности страниц. Из-за ошибки контрольной суммы СУБД не вернула данные.**
>как проигнорировать ошибку и продолжить работу?
   ```sql
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 off
postgres=# alter system set ignore_checksum_failure = 'on';
postgres=# Select pg_reload_conf();
 pg_reload_conf
----------------
 t
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
postgres=# select * from test_text limit 5;
 ПРЕДУПРЕЖДЕНИЕ:  ошибка проверки страницы: получена контрольная сумма 30265, а ожидалась - 55324
     t
-----------
 строка 1
 строка 2
 строка 3
 строка 4
 строка 5
   ```
Для продолжения работы нужно включить флаг игнорирования данной ошибки **set ignore_checksum_failure = 'on'**
