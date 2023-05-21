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


***

> ### Topic
  * text

***
