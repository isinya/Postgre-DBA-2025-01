# Postgre-DBA-2025-01 Занятие #08
Журналы

Домашнее задание

>Настройте выполнение контрольной точки раз в 30 секунд.
   ```sql
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 строка)

postgres=# alter system set checkpoint_timeout = 30;
ALTER SYSTEM
postgres=# Select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 строка)

postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 строка)

   ```
