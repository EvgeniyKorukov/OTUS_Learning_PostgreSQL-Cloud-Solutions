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
   * Number of Connections: `20`
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

    ```
* Выводы:
  * 

***

> ### Настроить кластер на оптимальную производительность не обращая внимания на стабильность БД
* Настройкам ОС пренебрегаем т.к. тут их показатели будут не заметны. В частности, я имею ввиду:

  

***






