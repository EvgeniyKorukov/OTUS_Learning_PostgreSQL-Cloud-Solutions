<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Углубленное изучение бэкапов и репликации" </h2></div>

***

> ### Делам бэкап Постгреса используя WAL-G и восстанавливаемся на другом кластере
  * Создаем master СУБД Postgres
    ```console
    user@srv-pg-ubuntu:~$ sudo pg_createcluster 15 master --start 
    sudo pg_createcluster 15 master --start 

    Creating new PostgreSQL cluster 15/master ...
    /usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/master --auth-local peer --auth-host scram-sha-256 --no-instructions
    The files belonging to this database system will be owned by user "postgres".
    This user must also own the server process.

    The database cluster will be initialized with locale "C.UTF-8".
    The default database encoding has accordingly been set to "UTF8".
    The default text search configuration will be set to "english".

    Data page checksums are disabled.

    fixing permissions on existing directory /var/lib/postgresql/15/master ... ok
    creating subdirectories ... ok
    selecting dynamic shared memory implementation ... posix
    selecting default max_connections ... 100
    selecting default shared_buffers ... 128MB
    selecting default time zone ... Etc/UTC
    creating configuration files ... ok
    running bootstrap script ... ok
    performing post-bootstrap initialization ... ok
    syncing data to disk ... ok
    Ver Cluster Port Status Owner    Data directory                Log file
    15  master  5432 online postgres /var/lib/postgresql/15/master /var/log/postgresql/postgresql-15-master.log
    user@srv-pg-ubuntu:~$ 

    user@srv-pg-ubuntu:~$ pg_lsclusters 

    Ver Cluster Port Status Owner    Data directory                Log file
    15  master  5432 online postgres /var/lib/postgresql/15/master /var/log/postgresql/postgresql-15-master.log
    user@srv-pg-ubuntu:~$ 
    ```

  * Устанавливаем WAL-G
    ```console
    user@srv-pg-ubuntu:~$ wget https://github.com/wal-g/wal-g/releases/download/v2.0.1/wal-g-pg-ubuntu-20.04-amd64.tar.gz && tar -zxvf wal-g-pg-ubuntu-20.04-amd64.tar.gz && sudo mv w                                                                                             al-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
    --2023-05-24 23:13:41--  https://github.com/wal-g/wal-g/releases/download/v2.0.1/wal-g-pg-ubuntu-20.04-amd64.tar.gz
    Resolving github.com (github.com)... 140.82.121.4
    Connecting to github.com (github.com)|140.82.121.4|:443... connected.
    HTTP request sent, awaiting response... 302 Found
    Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/95143992/ecf0de0d-0c36-4aff-8540-15a8a00ba662?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Creden                                                                                             tial=AKIAIWNJYAX4CSVEH53A%2F20230524%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230524T231343Z&X-Amz-Expires=300&X-Amz-Signature=e3fc3c3046dac0f6b85345c4cc5cb2cc16704e40548b914                                                                                             9fbee9f0a868b115c&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=95143992&response-content-disposition=attachment%3B%20filename%3Dwal-g-pg-ubuntu-20.04-amd64.tar.gz&respons                                                                                             e-content-type=application%2Foctet-stream [following]
    --2023-05-24 23:13:43--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/95143992/ecf0de0d-0c36-4aff-8540-15a8a00ba662?X-Amz-Algorithm=AWS4-HMAC-SHA2                                                                                             56&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230524%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230524T231343Z&X-Amz-Expires=300&X-Amz-Signature=e3fc3c3046dac0f6b85345c4cc5cb2cc                                                                                             16704e40548b9149fbee9f0a868b115c&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=95143992&response-content-disposition=attachment%3B%20filename%3Dwal-g-pg-ubuntu-20.04-amd64                                                                                             .tar.gz&response-content-type=application%2Foctet-stream
    Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.110.133, 185.199.111.133, 185.199.108.133, ...
    Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.110.133|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 12822938 (12M) [application/octet-stream]
    Saving to: ‘wal-g-pg-ubuntu-20.04-amd64.tar.gz’

    wal-g-pg-ubuntu-20.04-amd64.tar.gz                                  100%[==================================================================================================================================================================>]  12.23M   459KB/s    in 21s

    2023-05-24 23:14:04 (598 KB/s) - ‘wal-g-pg-ubuntu-20.04-amd64.tar.gz’ saved [12822938/12822938]

    wal-g-pg-ubuntu-20.04-amd64
    user@srv-pg-ubuntu:~$
    user@srv-pg-ubuntu:~$ sudo ls -l /usr/local/bin/wal-g
    -rwxr-xr-x 1 user user 43903416 Aug 25  2022 /usr/local/bin/wal-g
    user@srv-pg-ubuntu:~$
    user@srv-pg-ubuntu:~$
    user@srv-pg-ubuntu:~$ sudo /usr/local/bin/wal-g --version
    wal-g version v2.0.1    b7d53dd 2022.08.25_09:34:20     PostgreSQL
    user@srv-pg-ubuntu:~$
    ```

  * Создаем каталог для РК и прописваем права
    ```console
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo rm -rf /pg_backups && sudo mkdir /pg_backups && sudo chmod 777 /pg_backups
    ubuntu@srv-postgres:~$ 
    ```
    
  * Создаем конфигурацию для WAL-G
    ```console
    user@srv-pg-ubuntu:~$ sudo su - postgres
    postgres@srv-pg-ubuntu:~$ cd ~
    postgres@srv-pg-ubuntu:~$ ls
    15  backups  wal-g-pg-ubuntu-20.04-amd64  wal-g-pg-ubuntu-20.04-amd64.tar.gz
    postgres@srv-pg-ubuntu:~$ vim ~/.walg.json
    postgres@srv-pg-ubuntu:~$ cat ~/.walg.json
    {
        "WALG_FILE_PREFIX": "/pg_backups",
        "WALG_COMPRESSION_METHOD": "brotli",
        "WALG_DELTA_MAX_STEPS": "5",
        "PGDATA": "/var/lib/postgresql/15/master",
        "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
    }
    postgres@srv-pg-ubuntu:~$
    ```    
   
  * Правим параметры кластера баз данных PostgreSQL и перезапускаем его, чтобы параметры применились.
    ```console
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ mkdir /var/lib/postgresql/15/master/log
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ echo "wal_level=replica" >> /var/lib/postgresql/15/master/postgresql.auto.conf
    postgres@srv-pg-ubuntu:~$ echo "archive_mode=on" >> /var/lib/postgresql/15/master/postgresql.auto.conf
    postgres@srv-pg-ubuntu:~$ echo "archive_command='wal-g wal-push \"%p\" >> /var/lib/postgresql/15/master/log/archive_command.log 2>&1' " >> /var/lib/postgresql/15/master/postgresql.auto.conf
    postgres@srv-pg-ubuntu:~$ echo "archive_timeout=60" >> /var/lib/postgresql/15/master/postgresql.auto.conf
    postgres@srv-pg-ubuntu:~$ echo "restore_command='wal-g wal-fetch \"%f\" \"%p\" >> /var/lib/postgresql/15/master/log/restore_command.log 2>&1' " >> /var/lib/postgresql/15/master/postgresql.auto.conf
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ cat ~/15/master/postgresql.auto.conf
    # Do not edit this file manually!
    # It will be overwritten by the ALTER SYSTEM command.
    wal_level=replica
    archive_mode=on
    archive_command='wal-g wal-push "%p" >> /var/lib/postgresql/15/master/log/archive_command.log 2>&1'
    archive_timeout=60
    restore_command='wal-g wal-fetch "%f" "%p" >> /var/lib/postgresql/15/master/log/restore_command.log 2>&1'
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ pg_ctlcluster 15 master stop
    Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
      sudo systemctl stop postgresql@15-master
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ pg_lsclusters
    Ver Cluster Port Status Owner    Data directory                Log file
    15  master  5432 down   postgres /var/lib/postgresql/15/master /var/log/postgresql/postgresql-15-master.log
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ pg_ctlcluster 15 master start
    Warning: the cluster will not be running as a systemd service. Consider using systemctl:
      sudo systemctl start postgresql@15-master
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ pg_lsclusters
    Ver Cluster Port Status Owner    Data directory                Log file
    15  master  5432 online postgres /var/lib/postgresql/15/master /var/log/postgresql/postgresql-15-master.log
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$
    ```

  * Создаем БД и наполняем ёё
    ```console
    postgres@srv-postgres:~$ psql -c "CREATE DATABASE otus;"
    CREATE DATABASE
    postgres@srv-postgres:~$ psql otus -c "CREATE TABLE test(i int);"
    psql otus -c "INSERT INTO test VALUES (10), (20), (30);"
    psql otus -c "SELECT * FROM test;"
    CREATE TABLE
    INSERT 0 3
     i  
    ----
     10
     20
     30
    (3 rows)
    ```
    
  * Выполняем полный бэкап
    ```console   
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:~$ cd /pg_backups  
    postgres@srv-pg-ubuntu:~$
    postgres@srv-pg-ubuntu:/pg_backups$ wal-g backup-push /var/lib/postgresql/15/master 
    wal-g backup-push /var/lib/postgresql/15/master 

    INFO: 2023/05/22 19:18:29.356735 Couldn't find previous backup. Doing full backup.
    INFO: 2023/05/22 19:18:29.367786 Calling pg_start_backup()
    INFO: 2023/05/22 19:18:30.660163 Starting a new tar bundle
    INFO: 2023/05/22 19:18:30.660705 Walking ...
    INFO: 2023/05/22 19:18:30.661893 Starting part 1 ...
    INFO: 2023/05/22 19:18:30.952040 Packing ...
    INFO: 2023/05/22 19:18:30.953272 Finished writing part 1.
    INFO: 2023/05/22 19:18:30.953940 Starting part 2 ...
    INFO: 2023/05/22 19:18:30.954060 /global/pg_control
    INFO: 2023/05/22 19:18:30.955011 Finished writing part 2.
    INFO: 2023/05/22 19:18:30.955124 Calling pg_stop_backup()
    INFO: 2023/05/22 19:18:30.985838 Starting part 3 ...
    INFO: 2023/05/22 19:18:30.986010 backup_label
    INFO: 2023/05/22 19:18:30.986197 tablespace_map
    INFO: 2023/05/22 19:18:30.986460 Finished writing part 3.
    INFO: 2023/05/22 19:18:30.991262 Wrote backup with name base_000000010000000000000002
    postgres@srv-pg-ubuntu:/pg_backups$ 

    postgres@srv-pg-ubuntu:/pg_backups$ 

    postgres@srv-pg-ubuntu:/pg_backups$ cat /var/log/postgresql/postgresql-15-master.log
    cat /var/log/postgresql/postgresql-15-master.log

    2023-05-22 19:05:32.645 UTC [3326] LOG:  starting PostgreSQL 15.3 (Ubuntu 15.3-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.3.0-1ubuntu1~22.04) 11.3.0, 64-bit
    2023-05-22 19:05:32.646 UTC [3326] LOG:  listening on IPv4 address "127.0.0.1", port 5432
    2023-05-22 19:05:32.653 UTC [3326] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
    2023-05-22 19:05:32.670 UTC [3329] LOG:  database system was shut down at 2023-05-22 19:05:27 UTC
    2023-05-22 19:05:32.689 UTC [3326] LOG:  database system is ready to accept connections
    2023-05-22 19:10:32.771 UTC [3327] LOG:  checkpoint starting: time
    2023-05-22 19:10:36.933 UTC [3327] LOG:  checkpoint complete: wrote 43 buffers (0.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=4.042 s, sync=0.060 s, total=4.163 s; sync files=11, longest=0.019 s, average=0.006 s; distance=252 kB, estimate=252 kB
    2023-05-22 19:17:26.194 UTC [3326] LOG:  received fast shutdown request
    2023-05-22 19:17:26.209 UTC [3326] LOG:  aborting any active transactions
    2023-05-22 19:17:26.214 UTC [3326] LOG:  background worker "logical replication launcher" (PID 3332) exited with exit code 1
    2023-05-22 19:17:26.218 UTC [3327] LOG:  shutting down
    2023-05-22 19:17:26.226 UTC [3327] LOG:  checkpoint starting: shutdown immediate
    2023-05-22 19:17:26.263 UTC [3327] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.046 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=227 kB
    2023-05-22 19:17:26.269 UTC [3326] LOG:  database system is shut down
    2023-05-22 19:17:33.590 UTC [3446] LOG:  starting PostgreSQL 15.3 (Ubuntu 15.3-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.3.0-1ubuntu1~22.04) 11.3.0, 64-bit
    2023-05-22 19:17:33.590 UTC [3446] LOG:  listening on IPv4 address "127.0.0.1", port 5432
    2023-05-22 19:17:33.599 UTC [3446] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
    2023-05-22 19:17:33.617 UTC [3449] LOG:  database system was shut down at 2023-05-22 19:17:26 UTC
    2023-05-22 19:17:33.629 UTC [3446] LOG:  database system is ready to accept connections
    2023-05-22 19:18:29.384 UTC [3447] LOG:  checkpoint starting: immediate force wait
    2023-05-22 19:18:30.656 UTC [3447] LOG:  checkpoint complete: wrote 928 buffers (5.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=0.107 s, sync=1.026 s, total=1.273 s; sync files=251, longest=0.011 s, average=0.005 s; distance=11018 kB, estimate=11018 kB
    postgres@srv-pg-ubuntu:/pg_backups$ 

    postgres@srv-pg-ubuntu:/pg_backups$ cat /var/lib/postgresql/15/master/log/archive_command.log
    cat /var/lib/postgresql/15/master/log/archive_command.log

    INFO: 2023/05/22 19:18:29.584123 FILE PATH: 000000010000000000000001.br
    INFO: 2023/05/22 19:18:31.066643 FILE PATH: 000000010000000000000002.br
    INFO: 2023/05/22 19:18:31.117664 FILE PATH: 000000010000000000000002.00000028.backup.br
    postgres@srv-pg-ubuntu:/pg_backups$ 

    postgres@srv-pg-ubuntu:/pg_backups$ wal-g backup-listwal-g backup-list

    name                          modified             wal_segment_backup_start
    base_000000010000000000000002 2023-05-22T19:18:30Z 000000010000000000000002
    postgres@srv-pg-ubuntu:/pg_backups$ 
    ```
    
  * Изменяем данные в таблице
    ```console    
    postgres@srv-pg-ubuntu:/pg_backups$ psql otus -c "UPDATE test SET i = 3 WHERE i = 30"
    psql otus -c "UPDATE test SET i = 3 WHERE i = 30"

    UPDATE 1
    postgres@srv-pg-ubuntu:/pg_backups$ 
    ```
    
  * Выполняем инкрементальный бэкап
    ```console    
    postgres@srv-pg-ubuntu:/pg_backups$ wal-g backup-push /var/lib/postgresql/15/master
    wal-g backup-push /var/lib/postgresql/15/master

    INFO: 2023/05/22 19:19:28.455790 LATEST backup is: 'base_000000010000000000000002'
    INFO: 2023/05/22 19:19:28.456638 Delta backup from base_000000010000000000000002 with LSN 0/2000028.
    INFO: 2023/05/22 19:19:28.477300 Calling pg_start_backup()
    INFO: 2023/05/22 19:19:28.684793 Delta backup enabled
    INFO: 2023/05/22 19:19:28.685294 Starting a new tar bundle
    INFO: 2023/05/22 19:19:28.685879 Walking ...
    INFO: 2023/05/22 19:19:28.688508 Starting part 1 ...
    INFO: 2023/05/22 19:19:28.720077 Packing ...
    INFO: 2023/05/22 19:19:28.721430 Finished writing part 1.
    INFO: 2023/05/22 19:19:28.722138 Starting part 2 ...
    INFO: 2023/05/22 19:19:28.722756 /global/pg_control
    INFO: 2023/05/22 19:19:28.723666 Finished writing part 2.
    INFO: 2023/05/22 19:19:28.723999 Calling pg_stop_backup()
    INFO: 2023/05/22 19:19:28.754345 Starting part 3 ...
    INFO: 2023/05/22 19:19:28.754575 backup_label
    INFO: 2023/05/22 19:19:28.754821 tablespace_map
    INFO: 2023/05/22 19:19:28.755262 Finished writing part 3.
    INFO: 2023/05/22 19:19:28.766645 Wrote backup with name base_000000010000000000000004_D_000000010000000000000002
    postgres@srv-pg-ubuntu:/pg_backups$ 
    postgres@srv-pg-ubuntu:/pg_backups$ wal-g backup-listwal-g backup-list

    name                                                     modified             wal_segment_backup_start
    base_000000010000000000000002                            2023-05-22T19:18:30Z 000000010000000000000002
    base_000000010000000000000004_D_000000010000000000000002 2023-05-22T19:19:28Z 000000010000000000000004
    postgres@srv-pg-ubuntu:/pg_backups$ 
    postgres@srv-pg-ubuntu:/pg_backups$ ls
    basebackups_005  wal_005
    postgres@srv-pg-ubuntu:/pg_backups$ 
    ```   
    
    
    
