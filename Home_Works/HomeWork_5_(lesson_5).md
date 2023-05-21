<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Углубленное изучение бэкапов и репликации" </h2></div>


***

> ### Делам бэкап Постгреса используя WAL-G или pg_probackup и восстанавливаемся на другом кластере
  * Создаем master СУБД Postgres
    ```console
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ pg_lsclusters 
    Ver Cluster Port Status Owner Data directory Log file
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo pg_createcluster 15 main --start
    Creating new PostgreSQL cluster 15/main ...
    /usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions
    The files belonging to this database system will be owned by user "postgres".
    This user must also own the server process.

    The database cluster will be initialized with locale "en_US.UTF-8".
    The default database encoding has accordingly been set to "UTF8".
    The default text search configuration will be set to "english".

    Data page checksums are disabled.

    fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
    creating subdirectories ... ok
    selecting dynamic shared memory implementation ... posix
    selecting default max_connections ... 100
    selecting default shared_buffers ... 128MB
    selecting default time zone ... Etc/UTC
    creating configuration files ... ok
    running bootstrap script ... ok
    performing post-bootstrap initialization ... ok
    syncing data to disk ... ok
    Ver Cluster Port Status Owner    Data directory              Log file
    15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ pg_lsclusters 
    Ver Cluster Port Status Owner    Data directory              Log file
    15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
    ubuntu@srv-postgres:~$ 
    ```

  * Устанавливаем pg_probackup
    ```console
    ubuntu@srv-postgres:~$ sudo sh -c 'echo "deb [arch=amd64] https://repo.postgrespro.ru/pg_probackup/deb/ $(lsb_release -cs) main-$(lsb_release -cs)" > /etc/apt/sources.list.d/pg_probackup.list' && sudo wget -O - https://repo.postgrespro.ru/pg_probackup/keys/GPG-KEY-PG_PROBACKUP | sudo apt-key add - && sudo apt-get update


    Redirecting output to ‘wget-log’.
    Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).
    OK
    Hit:1 http://ru.archive.ubuntu.com/ubuntu jammy InRelease
    Get:2 http://ru.archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB] 
    Get:3 http://ru.archive.ubuntu.com/ubuntu jammy-backports InRelease [108 kB]
    Get:4 https://repo.postgrespro.ru/pg_probackup/deb jammy InRelease [3,354 B]
    Get:5 http://ru.archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
    Hit:6 http://apt.postgresql.org/pub/repos/apt jammy-pgdg InRelease                   
    Get:7 http://ru.archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [601 kB]
    Get:8 http://ru.archive.ubuntu.com/ubuntu jammy-updates/main Translation-en [170 kB]
    Get:9 http://ru.archive.ubuntu.com/ubuntu jammy-updates/main amd64 c-n-f Metadata [14.5 kB]
    Get:10 http://ru.archive.ubuntu.com/ubuntu jammy-updates/restricted amd64 Packages [251 kB]
    Get:11 http://ru.archive.ubuntu.com/ubuntu jammy-updates/restricted Translation-en [36.9 kB]
    Get:12 http://ru.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [906 kB]
    Get:13 http://ru.archive.ubuntu.com/ubuntu jammy-updates/universe Translation-en [186 kB]
    Get:14 http://ru.archive.ubuntu.com/ubuntu jammy-updates/universe amd64 c-n-f Metadata [18.9 kB]
    Get:15 https://repo.postgrespro.ru/pg_probackup/deb jammy/main-jammy amd64 Packages [1,696 B]
    Get:16 http://ru.archive.ubuntu.com/ubuntu jammy-security/main amd64 Packages [385 kB]
    Get:17 http://ru.archive.ubuntu.com/ubuntu jammy-security/main Translation-en [111 kB]
    Get:18 http://ru.archive.ubuntu.com/ubuntu jammy-security/main amd64 c-n-f Metadata [9,812 B]
    Get:19 http://ru.archive.ubuntu.com/ubuntu jammy-security/restricted amd64 Packages [250 kB]
    Get:20 http://ru.archive.ubuntu.com/ubuntu jammy-security/restricted Translation-en [36.6 kB]
    Get:21 http://ru.archive.ubuntu.com/ubuntu jammy-security/universe amd64 Packages [726 kB]
    Get:22 http://ru.archive.ubuntu.com/ubuntu jammy-security/universe Translation-en [126 kB]
    Get:23 http://ru.archive.ubuntu.com/ubuntu jammy-security/universe amd64 c-n-f Metadata [14.6 kB]
    Fetched 4,185 kB in 4s (958 kB/s)                            
    Reading package lists... Done
    W: https://repo.postgrespro.ru/pg_probackup/deb/dists/jammy/InRelease: Key is stored in legacy trusted.gpg keyring (/etc/apt/trusted.gpg), see the DEPRECATION section in apt-key(8) for details.
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo DEBIAN_FRONTEND=noninteractive apt install pg-probackup-15 pg-probackup-15-dbg postgresql-contrib postgresql-15-pg-checksums -y
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    The following additional packages will be installed:
      pg-checksums-doc
    The following NEW packages will be installed:
      pg-checksums-doc pg-probackup-15 pg-probackup-15-dbg postgresql-15-pg-checksums postgresql-contrib
    0 upgraded, 5 newly installed, 0 to remove and 16 not upgraded.
    Need to get 1,010 kB of archives.
    After this operation, 1,504 kB of additional disk space will be used.
    Get:1 https://repo.postgrespro.ru/pg_probackup/deb jammy/main-jammy amd64 pg-probackup-15 amd64 2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy [210 kB]
    Get:2 https://repo.postgrespro.ru/pg_probackup/deb jammy/main-jammy amd64 pg-probackup-15-dbg amd64 2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy [689 kB]
    Get:3 http://apt.postgresql.org/pub/repos/apt jammy-pgdg/main amd64 pg-checksums-doc all 1.1-4.pgdg22.04+1 [7,128 B]
    Get:4 http://apt.postgresql.org/pub/repos/apt jammy-pgdg/main amd64 postgresql-15-pg-checksums amd64 1.1-4.pgdg22.04+1 [36.4 kB]
    Get:5 http://apt.postgresql.org/pub/repos/apt jammy-pgdg/main amd64 postgresql-contrib all 15+249.pgdg22.04+1 [68.1 kB]
    Fetched 1,010 kB in 1s (1,157 kB/s)           
    Selecting previously unselected package pg-checksums-doc.
    (Reading database ... 115567 files and directories currently installed.)
    Preparing to unpack .../pg-checksums-doc_1.1-4.pgdg22.04+1_all.deb ...
    Unpacking pg-checksums-doc (1.1-4.pgdg22.04+1) ...
    Selecting previously unselected package pg-probackup-15.
    Preparing to unpack .../pg-probackup-15_2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy_amd64.deb ...
    Unpacking pg-probackup-15 (2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy) ...
    Selecting previously unselected package pg-probackup-15-dbg.
    Preparing to unpack .../pg-probackup-15-dbg_2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy_amd64.deb ...
    Unpacking pg-probackup-15-dbg (2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy) ...
    Selecting previously unselected package postgresql-15-pg-checksums.
    Preparing to unpack .../postgresql-15-pg-checksums_1.1-4.pgdg22.04+1_amd64.deb ...
    Unpacking postgresql-15-pg-checksums (1.1-4.pgdg22.04+1) ...
    Selecting previously unselected package postgresql-contrib.
    Preparing to unpack .../postgresql-contrib_15+249.pgdg22.04+1_all.deb ...
    Unpacking postgresql-contrib (15+249.pgdg22.04+1) ...
    Setting up pg-checksums-doc (1.1-4.pgdg22.04+1) ...
    Setting up postgresql-15-pg-checksums (1.1-4.pgdg22.04+1) ...
    Setting up postgresql-contrib (15+249.pgdg22.04+1) ...
    Setting up pg-probackup-15 (2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy) ...
    Setting up pg-probackup-15-dbg (2.5.12-1.d6721662ec76257d9470b1d20d75b7bc6bb1501c.jammy) ...
    Processing triggers for man-db (2.10.2-1) ...
    NEEDRESTART-VER: 3.5
    NEEDRESTART-KCUR: 5.15.0-72-generic
    NEEDRESTART-KEXP: 5.15.0-72-generic
    NEEDRESTART-KSTA: 1
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ 
    ```
  
  * Создаем каталог для РК, прописваем права и переменные окружения
    ```console
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo rm -rf /pg_backups && sudo mkdir /pg_backups && sudo chmod 777 /pg_backups
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo su postgres

    postgres@srv-postgres:/home/ubuntu$ 
    postgres@srv-postgres:/home/ubuntu$ echo "BACKUP_PATH=/pg_backups/">>~/.bashrc
    postgres@srv-postgres:/home/ubuntu$ echo "export BACKUP_PATH">>~/.bashrc
    postgres@srv-postgres:/home/ubuntu$ cd ~
    postgres@srv-postgres:~$ . .bashrc
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ echo $BACKUP_PATH
    /pg_backups/
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$
    ```
    
  * Создаем пользователя для РК в СУБД и назначаем ему права
    ```console 
    postgres@srv-postgres:~$ psql
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    Type "help" for help.

    postgres=# BEGIN;
    CREATE ROLE backup WITH LOGIN;
    GRANT USAGE ON SCHEMA pg_catalog TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
    COMMIT;

    ALTER ROLE backup WITH REPLICATION;

    exit
    BEGIN
    CREATE ROLE
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    COMMIT
    ALTER ROLE
    postgres-# 
    ```

  * Инициализируем pg_probackup, добавляем инстанс и смотрим настройки инстанса
    ```console
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 init
    INFO: Backup catalog '/pg_backups' successfully initialized
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 add-instance --instance 'main' -D /var/lib/postgresql/15/main
    INFO: Instance 'main' successfully initialized
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 show-config --instance main
    # Backup instance information
    pgdata = /var/lib/postgresql/15/main
    system-identifier = 7235684450752214614
    xlog-seg-size = 16777216
    # Connection parameters
    pgdatabase = postgres
    # Replica parameters
    replica-timeout = 5min
    # Archive parameters
    archive-timeout = 5min
    # Logging parameters
    log-level-console = INFO
    log-level-file = OFF
    log-format-console = PLAIN
    log-format-file = PLAIN
    log-filename = pg_probackup.log
    log-rotation-size = 0TB
    log-rotation-age = 0d
    # Retention parameters
    retention-redundancy = 0
    retention-window = 0
    wal-depth = 0
    # Compression parameters
    compress-algorithm = none
    compress-level = 1
    # Remote access parameters
    remote-proto = ssh
    postgres@srv-postgres:~$ 
    ```

  * Создаем 2 БД и наполняем их
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

    postgres@srv-postgres:~$ psql -c "CREATE DATABASE otus2;"
    CREATE DATABASE
    postgres@srv-postgres:~$ psql otus2 -c "CREATE TABLE test2(i int);"
    psql otus2 -c "INSERT INTO test2 VALUES (40), (50), (60), (70);"
    psql otus2 -c "SELECT * FROM test2;"
    CREATE TABLE
    INSERT 0 4
     i  
    ----
     40
     50
     60
     70
    (4 rows)

    postgres@srv-postgres:~$ 
    ```

  * Делаем полный РК и смотрим список уже имеющихся РК
    ```console
    ostgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 backup --instance 'main' -b FULL --stream --temp-slot

    INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RV0ULQ, backup mode: FULL, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
    WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
    WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
    INFO: Database backup start
    INFO: wait for pg_backup_start()
    INFO: Wait for WAL segment /pg_backups/backups/main/RV0ULQ/database/pg_wal/000000010000000000000003 to be streamed
    INFO: PGDATA size: 36MB
    INFO: Current Start LSN: 0/3000028, TLI: 1
    INFO: Start transferring data files
    INFO: Data files are transferred, time elapsed: 1s
    INFO: wait for pg_stop_backup()
    INFO: pg_stop backup() successfully executed
    INFO: stop_lsn: 0/3003CF8
    INFO: Getting the Recovery Time from WAL
    INFO: Syncing backup files to disk
    INFO: Backup files are synced, time elapsed: 1s
    INFO: Validating backup RV0ULQ
    INFO: Backup RV0ULQ data files are valid
    INFO: Backup RV0ULQ resident size: 53MB
    INFO: Backup RV0ULQ completed
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 show

    BACKUP INSTANCE 'main'
    ================================================================================================================================
     Instance  Version  ID      Recovery Time           Mode  WAL Mode  TLI  Time  Data   WAL  Zratio  Start LSN  Stop LSN   Status 
    ================================================================================================================================
     main      15       RV0ULQ  2023-05-21 18:41:52+00  FULL  STREAM    1/0   11s  37MB  16MB    1.00  0/3000028  0/3003CF8  OK     
    postgres@srv-postgres:~$ 
    ```

  * Выполняем инкрементальный РК
    * Добавляем данные в таблицу
    * Устанавливаем пользователю backup пароль
    * Создаем pgpass с фишкой от Евгения Аристова
    * Выдаем права на запуск функции для РК. Это надо делать для каждой БД
    * Выполняем инкрементальный РК
    * Смотрим список РК и их виды (full, delta,page)
    ```console
    ostgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ psql otus -c "insert into test values (4);"
    INSERT 0 1
    postgres@srv-postgres:~$ psql -c "ALTER USER backup PASSWORD 'otus123';"
    ALTER ROLE
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ psql otus -c "insert into test values (4);"
    INSERT 0 1
    postgres@srv-postgres:~$ psql -c "ALTER USER backup PASSWORD 'otus123';"
    ALTER ROLE
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ echo "localhost:5432:otus:backup:otus123">>~/.pgpass
    postgres@srv-postgres:~$ echo "localhost:5432:replication:backup:otus123">>~/.pgpass
    postgres@srv-postgres:~$ chmod 600 ~/.pgpass
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ psql -d otus
    psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
    Type "help" for help.

    otus=# BEGIN;
    GRANT USAGE ON SCHEMA pg_catalog TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.current_setting(text) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.set_config(text, text, boolean) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_is_in_recovery() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_start(text, boolean) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_backup_stop(boolean) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_create_restore_point(text) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_switch_wal() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_last_wal_replay_lsn() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_current() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_current_snapshot() TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.txid_snapshot_xmax(txid_snapshot) TO backup;
    GRANT EXECUTE ON FUNCTION pg_catalog.pg_control_checkpoint() TO backup;
    COMMIT;
    BEGIN
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    GRANT
    COMMIT
    otus=# 
    otus=# \q
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot -h localhost -U backup -d otus -p 5432
    INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RV0VQ9, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
    WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
    INFO: Database backup start
    INFO: wait for pg_backup_start()
    INFO: Parent backup: RV0ULQ
    INFO: Wait for WAL segment /pg_backups/backups/main/RV0VQ9/database/pg_wal/00000001000000000000000B to be streamed
    INFO: PGDATA size: 36MB
    INFO: Current Start LSN: 0/B000028, TLI: 1
    INFO: Parent Start LSN: 0/3000028, TLI: 1
    INFO: Start transferring data files
    INFO: Data files are transferred, time elapsed: 0
    INFO: wait for pg_stop_backup()
    INFO: pg_stop backup() successfully executed
    INFO: stop_lsn: 0/B000168
    INFO: Getting the Recovery Time from WAL
    INFO: Syncing backup files to disk
    INFO: Backup files are synced, time elapsed: 0
    INFO: Validating backup RV0VQ9
    INFO: Backup RV0VQ9 data files are valid
    INFO: Backup RV0VQ9 resident size: 16MB
    INFO: Backup RV0VQ9 completed
    postgres@srv-postgres:~$ 
    postgres@srv-postgres:~$ pg_probackup-15 show

    BACKUP INSTANCE 'main'
    ==================================================================================================================================
     Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time   Data   WAL  Zratio  Start LSN  Stop LSN   Status 
    ==================================================================================================================================
     main      15       RV0VQ9  2023-05-21 19:06:11+00  DELTA  STREAM    1/1    3s  436kB  16MB    1.00  0/B000028  0/B000168  OK     
     main      15       RV0ULQ  2023-05-21 18:41:52+00  FULL   STREAM    1/0   11s   37MB  16MB    1.00  0/3000028  0/3003CF8  OK     
    postgres@srv-postgres:~$ 
    ```
    
  * Для Евгения Аристова)
    * Создавать отдельного пользователя надо в том случае, если РК занимается отдельное подразделение или есть требования службы безопастности.
    * Если мы работаем под пользователем postgres, то не надо выдавать права на каждую БД, а РК можно сделать так
      *  pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot
      ```console
      postgres@srv-postgres:/home/ubuntu$ pg_probackup-15 backup --instance 'main' -b DELTA --stream --temp-slot
      could not change directory to "/home/ubuntu": Permission denied
      INFO: Backup start, pg_probackup version: 2.5.12, instance: main, backup ID: RV0W98, backup mode: DELTA, wal mode: STREAM, remote: false, compress-algorithm: none, compress-level: 1
      WARNING: This PostgreSQL instance was initialized without data block checksums. pg_probackup have no way to detect data block corruption without them. Reinitialize PGDATA with option '--data-checksums'.
      WARNING: Current PostgreSQL role is superuser. It is not recommended to run pg_probackup under superuser.
      INFO: Database backup start
      INFO: wait for pg_backup_start()
      INFO: Parent backup: RV0VQ9
      INFO: Wait for WAL segment /pg_backups/backups/main/RV0W98/database/pg_wal/00000001000000000000000D to be streamed
      INFO: PGDATA size: 36MB
      INFO: Current Start LSN: 0/D000028, TLI: 1
      INFO: Parent Start LSN: 0/B000028, TLI: 1
      INFO: Start transferring data files
      INFO: Data files are transferred, time elapsed: 0
      INFO: wait for pg_stop_backup()
      INFO: pg_stop backup() successfully executed
      INFO: stop_lsn: 0/D000168
      INFO: Getting the Recovery Time from WAL
      INFO: Syncing backup files to disk
      INFO: Backup files are synced, time elapsed: 2s
      INFO: Validating backup RV0W98
      INFO: Backup RV0W98 data files are valid
      INFO: Backup RV0W98 resident size: 38MB
      INFO: Backup RV0W98 completed
      postgres@srv-postgres:/home/ubuntu$ 
      ```

  * Создаем новую СУБД (кластер баз данных PostgreSQL) и восстанавливаем ее из имеющейся РК
    * Создаем новый экземпляр Postgres и удаляем его данные
    * Восстанавливаем из РК
    * Запускаем новый экземпляр Postgres
    * Проверяем, что есть новые данные
  ```console
  ubuntu@srv-postgres:~$ sudo pg_createcluster 15 main2
  [sudo] password for ubuntu: 
  Creating new PostgreSQL cluster 15/main2 ...
  /usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main2 --auth-local peer --auth-host scram-sha-256 --no-instructions
  The files belonging to this database system will be owned by user "postgres".
  This user must also own the server process.

  The database cluster will be initialized with locale "en_US.UTF-8".
  The default database encoding has accordingly been set to "UTF8".
  The default text search configuration will be set to "english".

  Data page checksums are disabled.

  fixing permissions on existing directory /var/lib/postgresql/15/main2 ... ok
  creating subdirectories ... ok
  selecting dynamic shared memory implementation ... posix
  selecting default max_connections ... 100
  selecting default shared_buffers ... 128MB
  selecting default time zone ... Etc/UTC
  creating configuration files ... ok
  running bootstrap script ... ok
  performing post-bootstrap initialization ... ok
  syncing data to disk ... ok
  Ver Cluster Port Status Owner    Data directory               Log file
  15  main2   5433 down   postgres /var/lib/postgresql/15/main2 /var/log/postgresql/postgresql-15-main2.log
  ubuntu@srv-postgres:~$ sudo rm -rf /var/lib/postgresql/15/main2
  ubuntu@srv-postgres:~$ sudo ls /var/lib/postgresql/15/main2
  ls: cannot access '/var/lib/postgresql/15/main2': No such file or directory
  ubuntu@srv-postgres:~$ sudo pg_probackup-15 show -B /pg_backups/

  BACKUP INSTANCE 'main'
  ===================================================================================================================================
   Instance  Version  ID      Recovery Time           Mode   WAL Mode  TLI  Time    Data   WAL  Zratio  Start LSN  Stop LSN   Status 
  ===================================================================================================================================
   main      15       RV0W98  2023-05-21 19:17:33+00  DELTA  STREAM    1/1    6s  6147kB  32MB    1.00  0/D000028  0/D000168  OK     
   main      15       RV0VQ9  2023-05-21 19:06:11+00  DELTA  STREAM    1/1    3s   436kB  16MB    1.00  0/B000028  0/B000168  OK     
   main      15       RV0ULQ  2023-05-21 18:41:52+00  FULL   STREAM    1/0   11s    37MB  16MB    1.00  0/3000028  0/3003CF8  OK     
  ubuntu@srv-postgres:~$ 

  ubuntu@srv-postgres:~$ sudo -u postgres pg_probackup-15 restore -B /pg_backups/ --instance main -i RV0VQ9 -D /var/lib/postgresql/15/main2 
  could not change directory to "/home/ubuntu": Permission denied
  INFO: Validating parents for backup RV0VQ9
  INFO: Validating backup RV0ULQ
  INFO: Backup RV0ULQ data files are valid
  INFO: Validating backup RV0VQ9
  INFO: Backup RV0VQ9 data files are valid
  INFO: Backup RV0VQ9 WAL segments are valid
  INFO: Backup RV0VQ9 is valid.
  INFO: Restoring the database from backup at 2023-05-21 19:06:09+00
  INFO: Start restoring backup files. PGDATA size: 52MB
  INFO: Backup files are restored. Transfered bytes: 52MB, time elapsed: 1s
  INFO: Restore incremental ratio (less is better): 100% (52MB/52MB)
  INFO: Syncing restored files to disk
  INFO: Restored backup files are synced, time elapsed: 7s
  INFO: Restore of backup RV0VQ9 completed.
  ubuntu@srv-postgres:~$ 
  ubuntu@srv-postgres:~$ 
  ubuntu@srv-postgres:~$ sudo pg_ctlcluster 15 main2 start
  ubuntu@srv-postgres:~$ 
  ubuntu@srv-postgres:~$ pg_lsclusters 
  Ver Cluster Port Status Owner    Data directory               Log file
  15  main    5432 online postgres /var/lib/postgresql/15/main  /var/log/postgresql/postgresql-15-main.log
  15  main2   5433 online postgres /var/lib/postgresql/15/main2 /var/log/postgresql/postgresql-15-main2.log
  ubuntu@srv-postgres:~$ 
  ubuntu@srv-postgres:~$ sudo -u postgres psql -p 5433 -d otus -c 'select * from test;'
  could not change directory to "/home/ubuntu": Permission denied
   i  
  ----
   10
   20
   30
    4
  (4 rows)

  ubuntu@srv-postgres:~$ 
  ```
  
  * **Новая запись в таблице есть, после восстановления-цель достигнута**

***
