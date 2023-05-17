<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Углубленный анализ производительности. Профилирование. Мониторинг. Оптимизация" </h2></div>


***

> ### Развернуть Постгрес на ВМ
  * Развернул `PostgreSQL 15` под ОС `Ubuntu 22.04`,  на своей ВМ в `Proxmox`
    ```console
    ubuntu@srv-postgres:~$ cat /etc/os-release 
    PRETTY_NAME="Ubuntu 22.04.2 LTS"
    NAME="Ubuntu"
    VERSION_ID="22.04"
    VERSION="22.04.2 LTS (Jammy Jellyfish)"
    VERSION_CODENAME=jammy
    ID=ubuntu
    ID_LIKE=debian
    HOME_URL="https://www.ubuntu.com/"
    SUPPORT_URL="https://help.ubuntu.com/"
    BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
    PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
    UBUNTU_CODENAME=jammy
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo cat /proc/version
    Linux version 5.15.0-71-generic (buildd@lcy02-amd64-044) (gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #78-Ubuntu SMP Tue Apr 18 09:00:29 UTC 2023
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo pg_lsclusters 
    Ver Cluster Port Status Owner    Data directory              Log file
    15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
    ubuntu@srv-postgres:~$ 
    ```

***

> ### Протестировать pg_bench
* Проводим тестирование с параметрами pgbench:
  * 10 клиентов `--client=10`
  * Новое подключение для каждой транзакции `--connect`
  * 5 параллельных потоков `--jobs=5`
  * Выводим прогрес каждые 30 секунд `--progress=30`
  * Время на тест 600 секунд или 10 минут `--time=600`
  * База: demo
    ```console
    ubuntu@srv-postgres:~$ sudo -u postgres pgbench --client=10 --connect --jobs=5 --progress=30 --time=600 demo
    pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 30.0 s, 66.0 tps, lat 106.606 ms stddev 47.556, 0 failed
    progress: 60.0 s, 66.4 tps, lat 106.058 ms stddev 46.579, 0 failed
    progress: 90.0 s, 66.6 tps, lat 106.415 ms stddev 46.263, 0 failed
    progress: 120.0 s, 66.3 tps, lat 104.145 ms stddev 41.497, 0 failed
    progress: 150.1 s, 66.6 tps, lat 105.470 ms stddev 44.399, 0 failed
    progress: 180.0 s, 66.0 tps, lat 104.274 ms stddev 43.232, 0 failed
    progress: 210.0 s, 65.5 tps, lat 108.150 ms stddev 44.213, 0 failed
    progress: 240.0 s, 66.0 tps, lat 107.245 ms stddev 44.016, 0 failed
    progress: 270.0 s, 66.2 tps, lat 107.257 ms stddev 47.112, 0 failed
    progress: 300.0 s, 65.8 tps, lat 107.845 ms stddev 44.980, 0 failed
    progress: 330.0 s, 65.9 tps, lat 106.175 ms stddev 43.907, 0 failed
    progress: 360.0 s, 65.6 tps, lat 105.817 ms stddev 45.380, 0 failed
    progress: 390.0 s, 66.3 tps, lat 105.819 ms stddev 45.310, 0 failed
    progress: 420.0 s, 66.1 tps, lat 105.847 ms stddev 43.916, 0 failed
    progress: 450.0 s, 66.0 tps, lat 105.475 ms stddev 41.247, 0 failed
    progress: 480.0 s, 65.7 tps, lat 107.046 ms stddev 43.622, 0 failed
    progress: 510.0 s, 66.0 tps, lat 105.563 ms stddev 43.575, 0 failed
    progress: 540.0 s, 65.6 tps, lat 106.386 ms stddev 46.570, 0 failed
    progress: 570.1 s, 65.7 tps, lat 107.015 ms stddev 45.689, 0 failed
    progress: 600.0 s, 66.0 tps, lat 106.003 ms stddev 44.603, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 10
    number of threads: 5
    maximum number of tries: 1
    duration: 600 s
    number of transactions actually processed: 39618
    number of failed transactions: 0 (0.000%)
    latency average = 106.224 ms
    latency stddev = 44.725 ms
    average connection time = 45.224 ms
    tps = 66.022330 (including reconnection times)
    ubuntu@srv-postgres:~$ 
    ```

***

> ### Выставить оптимальные настройки
 * Настройкам ОС пренебрегаем т.к. тут их показатели будут не заметны. В частности, я имею ввиду:
   * huge pages
   * transparent_hugepages
   * swapiness
   * лимиты и прочее
 * Берем калькулятор параметров [PGTune](https://pgtune.leopard.in.ua/) и считаем параметры для следующей конфигурации:
   * DB version: `15`
   * OS Type: `Linux`
   * DB Type: `Mixed type of application`
   * Total Memory (RAM): `2 GB`
   * Number of CPUs: `1`
   * Number of Connections: `20` ❗️т.к. pgtune не считает с connection меньше 20
   * Data Storage: `SSD Storage`
     ```sql
     # DB Version: 15
     # OS Type: linux
     # DB Type: mixed
     # Total Memory (RAM): 2 GB
     # CPUs num: 1
     # Connections num: 20
     # Data Storage: ssd

     max_connections = 20
     shared_buffers = 512MB
     effective_cache_size = 1536MB
     maintenance_work_mem = 128MB
     checkpoint_completion_target = 0.9
     wal_buffers = 16MB
     default_statistics_target = 100
     random_page_cost = 1.1
     effective_io_concurrency = 200
     work_mem = 6553kB
     min_wal_size = 1GB
     max_wal_size = 4GB   
     ```
 * Применяем параметры на СУБД PostgreSQL и перезапускаем кластер, чтобы применились параметры:
    ```sql
    ubuntu@srv-postgres:~$ sudo -u postgres psql
    could not change directory to "/home/ubuntu": Permission denied
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    Type "help" for help.

    postgres=# 
    ALTER SYSTEM SET max_connections = '20';
    ALTER SYSTEM SET shared_buffers = '512MB';
    ALTER SYSTEM SET effective_cache_size = '1536MB';
    ALTER SYSTEM SET maintenance_work_mem = '128MB';
    ALTER SYSTEM SET checkpoint_completion_target = '0.9';
    ALTER SYSTEM SET wal_buffers = '16MB';
    ALTER SYSTEM SET default_statistics_target = '100';
    ALTER SYSTEM SET random_page_cost = '1.1';
    ALTER SYSTEM SET effective_io_concurrency = '200';
    ALTER SYSTEM SET work_mem = '6553kB';
    ALTER SYSTEM SET min_wal_size = '1GB';
    ALTER SYSTEM SET max_wal_size = '4GB';
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    postgres=# 
    postgres=# \q
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo  pg_ctlcluster 15 main restart
    ubuntu@srv-postgres:~$ 
    ```


***

> ### Проверить насколько выросла производительность
* Проводим тестирование с параметрами pgbench:
  * 10 клиентов `--client=10`
  * Новое подключение для каждой транзакции `--connect`
  * 5 параллельных потоков `--jobs=5`
  * Выводим прогрес каждые 30 секунд `--progress=30`
  * Время на тест 600 секунд или 10 минут `--time=600`
  * База: demo
    ```console
    ubuntu@srv-postgres:~$ sudo -u postgres pgbench --client=10 --connect --jobs=5 --progress=30 --time=600 demo
    pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 30.0 s, 65.2 tps, lat 107.680 ms stddev 46.124, 0 failed
    progress: 60.0 s, 65.4 tps, lat 108.561 ms stddev 46.275, 0 failed
    progress: 90.0 s, 65.4 tps, lat 107.072 ms stddev 44.912, 0 failed
    progress: 120.0 s, 65.4 tps, lat 107.234 ms stddev 47.425, 0 failed
    progress: 150.0 s, 65.3 tps, lat 108.209 ms stddev 46.529, 0 failed
    progress: 180.0 s, 65.7 tps, lat 106.189 ms stddev 42.666, 0 failed
    progress: 210.0 s, 65.5 tps, lat 106.677 ms stddev 43.738, 0 failed
    progress: 240.0 s, 65.2 tps, lat 107.284 ms stddev 44.676, 0 failed
    progress: 270.0 s, 65.5 tps, lat 108.234 ms stddev 46.285, 0 failed
    progress: 300.0 s, 65.3 tps, lat 107.736 ms stddev 45.785, 0 failed
    progress: 330.0 s, 64.2 tps, lat 108.711 ms stddev 45.018, 0 failed
    progress: 360.1 s, 65.4 tps, lat 108.028 ms stddev 46.110, 0 failed
    progress: 390.0 s, 65.2 tps, lat 106.436 ms stddev 44.878, 0 failed
    progress: 420.0 s, 65.0 tps, lat 109.260 ms stddev 47.808, 0 failed
    progress: 450.0 s, 64.8 tps, lat 108.565 ms stddev 47.684, 0 failed
    progress: 480.0 s, 64.7 tps, lat 107.474 ms stddev 44.278, 0 failed
    progress: 510.0 s, 65.4 tps, lat 106.759 ms stddev 44.940, 0 failed
    progress: 540.0 s, 65.2 tps, lat 105.888 ms stddev 42.045, 0 failed
    progress: 570.0 s, 65.2 tps, lat 106.874 ms stddev 42.809, 0 failed
    progress: 600.0 s, 65.1 tps, lat 108.668 ms stddev 46.620, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 10
    number of threads: 5
    maximum number of tries: 1
    duration: 600 s
    number of transactions actually processed: 39125
    number of failed transactions: 0 (0.000%)
    latency average = 107.568 ms
    latency stddev = 45.366 ms
    average connection time = 45.791 ms
    tps = 65.198881 (including reconnection times)
    ubuntu@srv-postgres:~$ 
    ```
* Выводы:
  * `number of transactions actually processed` было `39618` стало `39125`. Количество транзакций уменьшилось, что не есть хорошо.
  * `latency average` было `106.224 ms` стало `107.568 ms`. Средняя задержка увеличилась, что не есть хорошо.
  * `latency stddev` было `44.725 ms` стало `45.366 ms`. Средняя задержка дисковых устройств увеличилась, что не есть хорошо.
  * `average connection time` было `45.224 ms` стало `45.791 ms`. Увеличилось среднее время подключения, что не есть хорошо.
  * `tps` было `66.022330` стало `65.198881`. Уменьшилось количество транзакций в секунду, что не есть хорошо.
* Результат: 
  * Сделали хуже, чем было, на основании этого синтетического теста. 
  * Количество подключений в pgbench `10`, а в pgtune нельзя поставить меньше `20`. Это не существенно, но могло немного повлиять на результы тестов. Хотя при изменении количества подклчений меняется только `work_mem`
  * Еще, после калькуляции в pgtune уменьшили `max_connections` со `100` до `20`. Это как раз и могло сказаться на результате.

***

> ### Настроить кластер на оптимальную производительность не обращая внимания на стабильность БД
* Для этого изменим настройки и перезапустим кластер PostgreSQL для применения измений:
  * Переведем режим журналирования в минимальный режим т.к. реплик у нас нет, а минимального уровня хватит для восстановаления. 
    * [wal_level](https://postgrespro.ru/docs/postgrespro/15/runtime-config-wal)=`minimal`)
    * Параметр wal_level определяет, как много информации записывается в WAL. Со значением replica (по умолчанию) в журнал записываются данные, необходимые для поддержки архивирования WAL и репликации, включая запросы только на чтение на ведомом сервере. Вариант minimal оставляет только информацию, необходимую для восстановления после сбоя или аварийного отключения.
  * Количество `max_wal_senders` надо сделать равным `0` т.к. мы изменили wal_level, и если этого не сделать, то кластер PostgreSQL не запуститься
    * [max_wal_senders](https://postgrespro.ru/docs/postgrespro/15/runtime-config-replication#GUC-MAX-WAL-SENDERS)=`0`
  * Переведем кластер PostgreSQL в асинхронный режим, тем самым мы не будем дожидаться оповещения о результате операции. 
    * [synchronous_commit](https://postgrespro.ru/docs/postgrespro/15/runtime-config-wal#GUC-SYNCHRONOUS-COMMIT)=`off`
    * Определяет, после завершения какого уровня обработки WAL сервер будет сообщать об успешном выполнении операции. Допустимые значения: remote_apply (применено удалённо), on (вкл., по умолчанию), remote_write (записано удалённо), local (локально) и off (выкл.).
  * Отключим режим синхронизации(подтверждения физической записи на СХД) СУБД с ОС по дисковому вводу/выводу. Есть риск потерять данные в случает отключения питания. 
    * [fsync](https://postgrespro.ru/docs/postgrespro/15/runtime-config-wal#GUC-FSYNC)=`off`
    * Если этот параметр установлен, сервер Postgres Pro старается добиться, чтобы изменения были записаны на диск физически, выполняя системные вызовы fsync() или другими подобными методами (см. wal_sync_method). Это даёт гарантию, что кластер баз данных сможет вернуться в согласованное состояние после сбоя оборудования или операционной системы.
    * ❗️ Хотя отключение fsync часто даёт выигрыш в скорости, это может привести к неисправимой порче данных в случае отключения питания или сбоя системы. Поэтому отключать fsync рекомендуется, только если вы легко сможете восстановить всю базу из внешнего источника.
  * Отключим запись всего содержимого каждой страницы в wal при checkpoint
    * [full_page_writes](https://postgrespro.ru/docs/postgrespro/15/runtime-config-wal#GUC-FULL-PAGE-WRITES)=`off`
    * Когда этот параметр включён, сервер Postgres Pro записывает в WAL всё содержимое каждой страницы при первом изменении этой страницы после контрольной точки. Это необходимо, потому что запись страницы, прерванная при сбое операционной системы, может выполниться частично, и на диске окажется страница, содержащая смесь старых данных с новыми. При этом информации об изменениях на уровне строк, которая обычно сохраняется в WAL, будет недостаточно для получения согласованного содержимого такой страницы при восстановлении после сбоя. Сохранение образа всей страницы гарантирует, что страницу можно восстановить корректно, ценой увеличения объёма данных, которые будут записываться в WAL. (Так как воспроизведение WAL всегда начинается от контрольной точки, достаточно сделать это при первом изменении каждой страницы после контрольной точки. Таким образом, уменьшить затраты на запись полных страниц можно, увеличив интервалы контрольных точек.)
    * ❗️ Отключение этого параметра ускоряет обычные операции, но может привести к неисправимому повреждению или незаметной порче данных после сбоя системы. Так как при этом возникают практически те же риски, что и при отключении fsync, хотя и в меньшей степени, отключать его следует только при тех же обстоятельствах, которые перечислялись в рекомендациях для вышеописанного параметра.
  * Отключим `autovacuum`. 
    * [autovacuum](https://postgrespro.ru/docs/postgrespro/15/runtime-config-autovacuum)=`off`
    * ❗️ Этого ни в коем случае нельзя делать на промышленной системе и выйгруш мы получим только на начальном этапе, пока таблицы и индексы не разрастутся от модификации и изменений. 
    * ❗️НО у нас в задаче ничего не сказано про долгосрочную производительность и не обращать внимание на стабильность БД

    ```sql
    ubuntu@srv-postgres:~$ sudo -u postgres psql
    could not change directory to "/home/ubuntu": Permission denied
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    Type "help" for help.

    postgres=# alter system set wal_level='minimal';
    alter system set max_wal_senders='0';
    alter system set synchronous_commit='off';
    alter system set fsync='off';
    alter system set full_page_writes='off';
    alter system set autovacuum='off';
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    ALTER SYSTEM
    postgres=# 
    postgres=# \q
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo  pg_ctlcluster 15 main restart
    ubuntu@srv-postgres:~$ 
    ```
* Проводим тестирование с параметрами pgbench:
  * 10 клиентов `--client=10`
  * Новое подключение для каждой транзакции `--connect`
  * 5 параллельных потоков `--jobs=5`
  * Выводим прогрес каждые 30 секунд `--progress=30`
  * Время на тест 600 секунд или 10 минут `--time=600`
  * База: demo
    ```console
    ubuntu@srv-postgres:~$ sudo -u postgres pgbench --client=10 --connect --jobs=5 --progress=30 --time=600 demo
    pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    starting vacuum...end.
    progress: 30.0 s, 67.1 tps, lat 95.504 ms stddev 33.282, 0 failed
    progress: 60.0 s, 67.0 tps, lat 95.077 ms stddev 34.999, 0 failed
    progress: 90.0 s, 67.4 tps, lat 96.808 ms stddev 36.913, 0 failed
    progress: 120.0 s, 67.3 tps, lat 95.080 ms stddev 32.575, 0 failed
    progress: 150.0 s, 66.6 tps, lat 95.622 ms stddev 33.714, 0 failed
    progress: 180.0 s, 67.4 tps, lat 95.704 ms stddev 35.366, 0 failed
    progress: 210.1 s, 67.6 tps, lat 92.890 ms stddev 32.252, 0 failed
    progress: 240.0 s, 67.3 tps, lat 95.530 ms stddev 33.755, 0 failed

    progress: 270.0 s, 67.3 tps, lat 95.716 ms stddev 35.639, 0 failed
    progress: 300.0 s, 67.4 tps, lat 96.186 ms stddev 35.318, 0 failed
    progress: 330.0 s, 67.5 tps, lat 94.324 ms stddev 32.761, 0 failed
    progress: 360.0 s, 67.3 tps, lat 96.327 ms stddev 33.907, 0 failed
    progress: 390.0 s, 67.5 tps, lat 95.377 ms stddev 34.131, 0 failed
    progress: 420.0 s, 67.3 tps, lat 95.728 ms stddev 33.764, 0 failed
    progress: 450.0 s, 67.3 tps, lat 95.219 ms stddev 34.432, 0 failed
    progress: 480.0 s, 67.3 tps, lat 97.167 ms stddev 36.083, 0 failed
    progress: 510.0 s, 67.6 tps, lat 93.963 ms stddev 31.146, 0 failed
    progress: 540.0 s, 67.3 tps, lat 95.263 ms stddev 34.615, 0 failed
    progress: 570.0 s, 67.3 tps, lat 96.225 ms stddev 33.643, 0 failed
    progress: 600.0 s, 67.3 tps, lat 94.768 ms stddev 33.274, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 10
    number of threads: 5
    maximum number of tries: 1
    duration: 600 s
    number of transactions actually processed: 40390
    number of failed transactions: 0 (0.000%)
    latency average = 95.419 ms
    latency stddev = 34.119 ms
    average connection time = 53.138 ms
    tps = 67.307295 (including reconnection times)
    ubuntu@srv-postgres:~$ 
    ```  
* Выводы:
  * `number of transactions actually processed` было `39125` стало `40390`. Количество транзакций увеличилось и это хорошо.
  * `latency average` было `107.568 ms` стало `95.419 ms`. Средняя задержка уменьшилась и это хорошо.
  * `latency stddev` было `45.366 ms` стало `34.119 ms`. Средняя задержка дисковых устройств уменьшилась и это хорошо.
  * `average connection time` было `45.791 ms` стало `53.138 ms`. Увеличилось среднее время подключения, что не есть хорошо.
  * `tps` было `65.198881` стало `67.307295`. Увеличилось количество транзакций в секунду и это хорошо.
* Результат: 
  * Мы получили прирост производительности.
  * Но сильно снизили надежность.
  * Всегда приходтся вибирирать "золотую середину" между производительностью и надежностью.
  * Все зависит от требований к системе, ее важности и надежности.


***






