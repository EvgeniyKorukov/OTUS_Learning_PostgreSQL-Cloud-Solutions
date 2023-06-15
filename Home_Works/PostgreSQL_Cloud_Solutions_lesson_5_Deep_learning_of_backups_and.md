#### 1)

#### Делам бэкап Постгреса используя WAL-G или pg_probackup и восстанавливаемся на другом кластере

#### 1.1) install WAL-G

<pre> 
-- sudo rm /usr/local/bin/wal-g
wget https://github.com/philyuchkoff/wal-g-centos7/releases/download/2.0.1/wal-g-pg && mkdir /usr/local/bin/wal-g && sudo mv wal-g-pg /usr/local/bin/wal-g
sudo ls -l /usr/local/bin/wal-g

sudo rm -rf /var/postgre_backup && sudo mkdir /var/postgre_backup && sudo chmod 777 /var/postgre_backup
chmod 645 /usr/local/bin/wal-g/wal-g-pg
chown postgres:postgres /var/postgre_backup

</pre>


#### 1.2)Создаем файл конфигурации для wal-g

<pre> 
sudo su - postgres
vim ~/.walg.json


-- https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md
-- https://github.com/wal-g/wal-g/blob/master/docs/STORAGES.md

{
    "WALG_FILE_PREFIX": "/var/postgre_backup",
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "PGDATA": "/var/lib/pgsql/15/data",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}

-- опция для дебага
--     "WALG_LOG_LEVEL": "DEVEL"

mkdir /var/lib/pgsql/15/data/log_wal_g
chown postgres:postgres /var/lib/pgsql/15/data/log_wal_g
chmod 700 /var/lib/pgsql/15/data/log_wal_g

vim /var/lib/pgsql/15/data/postgresql.conf
или
echo "wal_level=replica" >> /var/lib/pgsql/15/data/postgresql.auto.conf
echo "archive_mode=on" >> /var/lib/pgsql/15/data/postgresql.auto.conf
echo "archive_command='/usr/local/bin/wal-g/wal-g-pg wal-push \"%p\" >> /var/lib/pgsql/15/data/log_wal_g/archive_command.log 2>&1' " >> /var/lib/pgsql/15/data/postgresql.auto.conf
echo "archive_timeout=60" >> /var/lib/pgsql/15/data/postgresql.auto.conf 
echo "restore_command='/usr/local/bin/wal-g/wal-g-pg wal-fetch \"%f\" \"%p\" >> /var/lib/pgsql/15/data/log_wal_g/restore_command.log 2>&1' " >> /var/lib/pgsql/15/data/postgresql.auto.conf

sudo systemctl restart postgresql-15
sudo systemctl status postgresql-15
</pre>


#### 1.3)наполняем данными 

<pre> 
su - postgres

-- Создадим новую базу данных
psql -c "CREATE DATABASE otus;"

-- Таблицу в этой базе данных и заполним ее тестовыми данными
psql otus -c "create table test(i int);"
psql otus -c "insert into test values (10), (20), (30);"
psql otus -c "select * from test;"
</pre>

#### 1.4) выполняем бекап

<pre> 
выполнил бэкап
su - postgres
su - postgres
su - postgres

-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
</pre>

#### 1.5) смотрим на наличие ошибок 

<pre> 
cat /var/lib/pgsql/15/data/log/postgresql-Sun.log
cat /var/lib/pgsql/15/data/log_wal_g/archive_command.log

-bash-4.2$ cat /var/lib/pgsql/15/data/log_wal_g/archive_command.log
INFO: 2023/05/21 00:29:08.700980 FILE PATH: 00000001000000000000000C.br
INFO: 2023/05/21 00:30:05.936070 FILE PATH: 00000001000000000000000D.br
INFO: 2023/05/21 00:30:06.353350 FILE PATH: 00000001000000000000000E.00000060.backup.br
INFO: 2023/05/21 00:30:06.361288 FILE PATH: 00000001000000000000000E.br
INFO: 2023/05/21 00:30:40.390422 FILE PATH: 00000001000000000000000F.br
INFO: 2023/05/21 00:30:56.695581 FILE PATH: 000000010000000000000010.br
INFO: 2023/05/21 00:30:56.824121 FILE PATH: 000000010000000000000011.00000028.backup.br
INFO: 2023/05/21 00:30:56.842308 FILE PATH: 000000010000000000000011.br
</pre>

#### 1.6) смотрим какие бекапы есть в наличии

<pre> 
/usr/local/bin/wal-g/wal-g-pg backup-list
/usr/local/bin/wal-g/wal-g-pg backup-list
/usr/local/bin/wal-g/wal-g-pg backup-list

-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-list
name                                                     modified                  wal_segment_backup_start
base_00000001000000000000000E                            2023-05-21T00:30:06+03:00 00000001000000000000000E
base_000000010000000000000011_D_00000001000000000000000E 2023-05-21T00:30:56+03:00 000000010000000000000011
base_000000010000000000000013_D_000000010000000000000011 2023-05-21T00:32:42+03:00 000000010000000000000013
</pre>

#### 1.7) обновляем наши данные

<pre> 
psql otus -c "UPDATE test SET i = 3 WHERE i = 30"
psql otus -c "select * from test;"
</pre>

#### 1.8) делаем еше один бекап

<pre> 
-- make delta
/usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
</pre>

#### 1.9) смотрим снова на лист бекапов

<pre> 
-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-list
name                                                     modified                  wal_segment_backup_start
base_00000001000000000000000E                            2023-05-21T00:30:06+03:00 00000001000000000000000E
base_000000010000000000000011_D_00000001000000000000000E 2023-05-21T00:30:56+03:00 000000010000000000000011
base_000000010000000000000013_D_000000010000000000000011 2023-05-21T00:32:42+03:00 000000010000000000000013
base_000000010000000000000016_D_000000010000000000000013 2023-05-21T00:39:49+03:00 000000010000000000000016
</pre>

#### 1.10) бекапы физически лежат сдесь

<pre> 
cd /var/postgre_backup
-bash-4.2$ ls -l
total 8
drwxr-xr-x. 6 postgres postgres 4096 May 21 00:39 basebackups_005
drwx------. 2 postgres postgres 4096 May 21 00:39 wal_005
cd basebackups_005
-bash-4.2$ cd basebackups_005
-bash-4.2$ ls -l
total 16
drwxr-xr-x. 3 postgres postgres  76 May 21 00:30 base_00000001000000000000000E
-rw-r--r--. 1 postgres postgres 379 May 21 00:30 base_00000001000000000000000E_backup_stop_sentinel.json
drwxr-xr-x. 3 postgres postgres  76 May 21 00:30 base_000000010000000000000011_D_00000001000000000000000E
-rw-r--r--. 1 postgres postgres 503 May 21 00:30 base_000000010000000000000011_D_00000001000000000000000E_backup_stop_sentinel.json
drwxr-xr-x. 3 postgres postgres  76 May 21 00:32 base_000000010000000000000013_D_000000010000000000000011
-rw-r--r--. 1 postgres postgres 530 May 21 00:32 base_000000010000000000000013_D_000000010000000000000011_backup_stop_sentinel.json
drwxr-xr-x. 3 postgres postgres  76 May 21 00:39 base_000000010000000000000016_D_000000010000000000000013
-rw-r--r--. 1 postgres postgres 528 May 21 00:39 base_000000010000000000000016_D_000000010000000000000013_backup_stop_sentinel.json
</pre>


#### 1.11) restore последняя резервная копия


<pre> 
-- https://github.com/wal-g/wal-g/blob/master/docs/PostgreSQL.md

# удаляю данные
rm -rf /var/lib/pgsql/15/data

###WAL-G также может получить последнюю резервную копию, используя LATEST:
### wal-g backup-fetch ~/extract/to/here LATEST

/usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data LATEST
/usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data LATEST
/usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data LATEST


### сделаем файл для восстановления из архивов wal и стартуем postgresql
su - postgres 
touch /var/lib/pgsql/15/data/recovery.signal
systemctl restart postgresql-15.service

2023-05-21 01:00:49.910 MSK [5651] LOG:  database system was interrupted; last known up at 2023-05-21 00:39:49 MSK
2023-05-21 01:00:50.118 MSK [5651] LOG:  starting archive recovery
2023-05-21 01:00:50.177 MSK [5651] LOG:  restored log file "000000010000000000000016" from archive
2023-05-21 01:00:50.253 MSK [5651] LOG:  redo starts at 0/16000028
2023-05-21 01:00:50.304 MSK [5651] LOG:  restored log file "000000010000000000000017" from archive
2023-05-21 01:00:50.405 MSK [5651] LOG:  consistent recovery state reached at 0/16000100
2023-05-21 01:00:50.405 MSK [5646] LOG:  database system is ready to accept read-only connections
2023-05-21 01:00:50.433 MSK [5651] LOG:  redo done at 0/17000110 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.18 s
2023-05-21 01:00:50.498 MSK [5651] LOG:  restored log file "000000010000000000000017" from archive
2023-05-21 01:00:50.584 MSK [5651] LOG:  selected new timeline ID: 2
2023-05-21 01:00:50.713 MSK [5651] LOG:  archive recovery complete
2023-05-21 01:00:50.719 MSK [5649] LOG:  checkpoint starting: end-of-recovery immediate wait
2023-05-21 01:00:50.752 MSK [5649] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=0.004 s, sync=0.006 s, total=0.039 s; sync files=2, longest=0.004 s, av
erage=0.003 s; distance=32768 kB, estimate=32768 kB
2023-05-21 01:00:50.761 MSK [5646] LOG:  database system is ready to accept connections

### как видим последнее наше изменение UPDATE test SET i = 3 WHERE i = 30 все ОКК!
-bash-4.2$ psql otus -c "select * from test;"
 i
----
 10
 20
  3
(3 rows)
</pre>


#### 1.12) restore на определюный бекап и Point-In-Time-Recovery (PITR) 

<pre> 
rm -rf /var/lib/pgsql/15/data
systemctl restart postgresql-15.service

[root@zk4 wal-g]# systemctl restart postgresql-15.service
Job for postgresql-15.service failed because the control process exited with error code. See "systemctl status postgresql-15.service" and "journalctl -xe" for details.
</pre>


### список бекапов

<pre> 
su - postgres
/usr/local/bin/wal-g/wal-g-pg backup-list
/usr/local/bin/wal-g/wal-g-pg backup-list
/usr/local/bin/wal-g/wal-g-pg backup-list

base_00000001000000000000000E                            2023-05-21T00:30:06+03:00 00000001000000000000000E
base_000000010000000000000011_D_00000001000000000000000E 2023-05-21T00:30:56+03:00 000000010000000000000011
base_000000010000000000000013_D_000000010000000000000011 2023-05-21T00:32:42+03:00 000000010000000000000013
base_000000010000000000000016_D_000000010000000000000013 2023-05-21T00:39:49+03:00 000000010000000000000016
</pre>


### восстанавливаюсь на определенный бекап
### /usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data backup_name

<pre>
/usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data base_000000010000000000000013_D_000000010000000000000011
/usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data base_000000010000000000000013_D_000000010000000000000011
/usr/local/bin/wal-g/wal-g-pg backup-fetch /var/lib/pgsql/15/data base_000000010000000000000013_D_000000010000000000000011
</pre>

### сделаем файл для восстановления из архивов wal и стартуем postgresql

<pre>
su - postgres 
touch /var/lib/pgsql/15/data/recovery.signal
chown postgres:postgres /var/lib/pgsql/15/data/recovery.signal
</pre>


### настироим параметры для Point-In-Time-Recovery (PITR) 
<pre>
vim /var/lib/pgsql/15/data/postgresql.auto.conf
recovery_target_time = '2023-05-21 00:32:42.000'
recovery_target_action = 'promote'
recovery_target_inclusive = 'on'

systemctl restart postgresql-15.service
</pre>

### УРА ВСЕ ПОЛУЧИЛОСЬ!!!

<pre>
-bash-4.2$ psql otus -c "select * from test;"
 i
----
 10
 20
 30
(3 rows)
</pre>


### удаляю ненужные файлы остатки после восстановления

<pre>
rm -rf /var/lib/pgsql/15/data/tablespace_map.old
rm -rf /var/lib/pgsql/15/data/backup_label.old
</pre>

### удаляю строки из postgresql.auto.conf

<pre>
vim /var/lib/pgsql/15/data/postgresql.auto.conf
recovery_target_time = '2023-05-21 00:32:42.000'
recovery_target_action = 'promote'
recovery_target_inclusive = 'on'
</pre>


#### 2)

#### Задание повышенной сложности* под нагрузкой* бэкап снимаем с реплики**

#### 2.1) Создаем два кластера postgresql для обединения в потоковую репликацию


<pre>
создал 2 vm centos core 2 озу 4gb 
otus_vm1_centos ip 192.168.2.4
otus_vm2_centos ip 192.168.2.8


sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum -y repolist
sudo yum install postgresql15-contrib

sudo /usr/pgsql-15/bin/postgresql-15-setup initdb

systemctl enable postgresql-15.service
systemctl status postgresql-15.service
systemctl restart postgresql-15.service

применил настройки:
ALTER SYSTEM SET
 max_connections = '100';
ALTER SYSTEM SET
 shared_buffers = '1GB';
ALTER SYSTEM SET
 effective_cache_size = '3GB';
ALTER SYSTEM SET
 maintenance_work_mem = '256MB';
ALTER SYSTEM SET
 checkpoint_completion_target = '0.9';
ALTER SYSTEM SET
 wal_buffers = '16MB';
ALTER SYSTEM SET
 default_statistics_target = '100';
ALTER SYSTEM SET
 random_page_cost = '1.1';
ALTER SYSTEM SET
 effective_io_concurrency = '200';
ALTER SYSTEM SET
 work_mem = '5242kB';
ALTER SYSTEM SET
 min_wal_size = '2GB';
ALTER SYSTEM SET
 max_wal_size = '8GB';
ALTER SYSTEM SET
 max_worker_processes = '4';
ALTER SYSTEM SET
 max_parallel_workers_per_gather = '2';
ALTER SYSTEM SET
 max_parallel_workers = '4';
ALTER SYSTEM SET
 max_parallel_maintenance_workers = '2';

systemctl restart postgresql-15.service
</pre>


#### 2.2) подготовительные мероприятия для настройки асинхронной репликации for linux


<pre> 
#otus_vm1_centos ip 192.168.2.4
firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source address="192.168.2.8" port protocol="tcp" port="5432" accept"


#otus_vm2_centos ip 192.168.2.8
firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source address="192.168.2.4" port protocol="tcp" port="5432" accept"

firewall-cmd --reload		
firewall-cmd --list-all
</pre>


#### 2.3) подготовительные мероприятия для настройки асинхронной репликации for postgres

<pre> 
sudo -u postgres psql -c "ALTER SYSTEM SET wal_level = 'replica';"
sudo -u postgres psql -c "ALTER SYSTEM SET listen_addresses TO '*';"
sudo -u postgres psql -c "ALTER SYSTEM SET synchronous_commit TO 'on';"
systemctl restart postgresql-15.service
</pre>

### Теперь включаем на VM1 (он же будет мастером режим асинхронной репликации)
### создаю на VM1 мастере пользователя который будет отвечать за репликацию 

<pre> 
sudo -u postgres psql -c "CREATE ROLE user_replicator WITH REPLICATION PASSWORD 'klJlkdfsjImdsksdmsd98' LOGIN;"
</pre>


### доступ для роли user_replicator

<pre> 
vim /var/lib/pgsql/15/data/pg_hba.conf
host    replication     user_replicator          192.168.2.8/32            scram-sha-256
</pre>


#### создаю слот на мастере VM1

<pre> 
sudo -u postgres psql -c "SELECT * FROM pg_create_physical_replication_slot('my_slot_replication');"
</pre>

### проверяю слот

<pre> 
su - postgres
psql
SELECT * FROM pg_replication_slots; \gx

-[ RECORD 1 ]-------+--------------------
slot_name           | my_slot_replication
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | f
active_pid          |
xmin                |
catalog_xmin        |
restart_lsn         |
confirmed_flush_lsn |
wal_status          |
safe_wal_size       |
two_phase           | f
</pre>


### на VM2 останавливаю кластер удаляю данные 

<pre> 
sudo systemctl stop postgresql-15
sudo systemctl status postgresql-15
sudo -u postgres rm -rf /var/lib/pgsql/15/data/*
sudo -u postgres pg_basebackup -h 192.168.2.4 -D /var/lib/pgsql/15/data -U user_replicator -P -v -R -X stream -S my_slot_replication

pass klJlkdfsjImdsksdmsd98

### важная заметка пока удаляю конфигурацию о архивировании журналов через wal-g пока на реплике не настроен сам wal-g. чтобы реплика корректно стартанула
archive_command='/usr/local/bin/wal-g/wal-g-pg wal-push "%p" >> /var/lib/pgsql/15/data/log_wal_g/archive_command.log 2>&1'
archive_timeout=60
restore_command='/usr/local/bin/wal-g/wal-g-pg wal-fetch "%f" "%p" >> /var/lib/pgsql/15/data/log_wal_g/restore_command.log 2>&1

#### и добавить параметр
archive_mode = 'off'
hot_standby = 'on'


### удаляю ненужные файлы остатки после восстановления
rm -rf /var/lib/pgsql/15/data/backup_label.old
</pre>

#проверяю что появилась дополнителная информация по слоту репликации

<pre> 
cat /var/lib/pgsql/15/data/postgresql.auto.conf
</pre>

### запускаю кластер на VM2

<pre> 
sudo systemctl restart postgresql-15
</pre>

### Сейчас VM2 сервер является репликой (находится в режиме восстановления):

<pre> 
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"

pg_is_in_recovery
-------------------
 t
(1 row)
</pre>



### на главном сервере VM1 проверяем репликацию

<pre> 
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 7250
usesysid         | 16388
usename          | user_replicator
application_name | walreceiver
client_addr      | 192.168.2.8
client_hostname  |
client_port      | 59178
backend_start    | 2023-05-21 11:41:06.95554+03
backend_xmin     |
state            | streaming
sent_lsn         | 0/26442670
write_lsn        | 0/26442670
flush_lsn        | 0/26442670
replay_lsn       | 0/26442670
write_lag        | 00:00:00.000705
flush_lag        | 00:00:00.002072
replay_lag       | 00:00:00.002202
sync_priority    | 0
sync_state       | async
reply_time       | 2023-05-21 11:42:40.438807+03


все ОК!
</pre>


#### 2.4) установка настройка wal-g на реплике проделаю по аналогии в задании 1 выше

### обязательный нюанс не добавляюм строку restore_command!!!!! и параметр  archive_mode=always на реплике!!!!!

<pre> 
vim /var/lib/pgsql/15/data/postgresql.conf
или
echo "wal_level=replica" >> /var/lib/pgsql/15/data/postgresql.auto.conf
echo "archive_mode=always" >> /var/lib/pgsql/15/data/postgresql.auto.conf
echo "archive_command='/usr/local/bin/wal-g/wal-g-pg wal-push \"%p\" >> /var/lib/pgsql/15/data/log_wal_g/archive_command.log 2>&1' " >> /var/lib/pgsql/15/data/postgresql.auto.conf
echo "archive_timeout=60" >> /var/lib/pgsql/15/data/postgresql.auto.conf 
### не добавляем !!!!!!!echo "restore_command='/usr/local/bin/wal-g/wal-g-pg wal-fetch \"%f\" \"%p\" >> /var/lib/pgsql/15/data/log_wal_g/restore_command.log 2>&1' " >> /var/lib/pgsql/15/data/postgresql.auto.conf

sudo systemctl restart postgresql-15
sudo systemctl status postgresql-15
</pre>

### убедимся что все работает на матере выполняю

<pre> 
SELECT pg_switch_wal();
SELECT pg_switch_wal();
SELECT pg_switch_wal();
</pre>


### проверяю нет ли ошибок на реплике 

<pre> 
cat /var/lib/pgsql/15/data/log/postgresql-Sun.log
cat /var/lib/pgsql/15/data/log_wal_g/archive_command.log

[root@client8 wal-g]# cat /var/lib/pgsql/15/data/log_wal_g/archive_command.log
INFO: 2023/05/21 14:01:58.777165 FILE PATH: 00000004000000000000002C.br
INFO: 2023/05/21 14:02:00.553924 FILE PATH: 00000004000000000000002D.br
INFO: 2023/05/21 14:04:00.493370 FILE PATH: 00000004000000000000002E.br

вижу все ок wal сегменты ра реплике архивируются!!!!!!!!
</pre>


#### 2.5) выполняю нагрузку на мастере vm1 при помощи  sysbench создам базу test и буду вставлять большое количество данных а в этот момент выполню бекап на реплике!!!!!

<pre>
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench

### генерирую большую вставку данных на мастере
sudo -u postgres psql -c "CREATE ROLE test LOGIN SUPERUSER PASSWORD 'test'"
sudo -u postgres psql -c "CREATE DATABASE test"
sudo -u postgres sysbench \
--db-driver=pgsql \
--oltp-table-size=1000000 \
--oltp-tables-count=30 \
--threads=1 \
--pgsql-host=localhost \
--pgsql-port=5432 \
--pgsql-user=test \
--pgsql-password=test \
--pgsql-db=test \
/usr/share/sysbench/tests/include/oltp_legacy/parallel_prepare.lua \
run
</pre>


#### 2.5)  выполню бекап на реплике пока идет массированная вставка данных на мастере!!!!!

<pre>
выполнил бэкап
su - postgres
su - postgres
su - postgres

-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data

-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-push /var/lib/pgsql/15/data
INFO: 2023/05/21 14:13:50.110471 Couldn't find previous backup. Doing full backup.
INFO: 2023/05/21 14:13:50.122503 Calling pg_start_backup()
INFO: 2023/05/21 14:13:50.145186 Starting a new tar bundle
INFO: 2023/05/21 14:13:50.145211 Walking ...
INFO: 2023/05/21 14:13:50.145388 Starting part 1 ...
INFO: 2023/05/21 14:13:58.009309 Packing ...
INFO: 2023/05/21 14:13:58.009706 Finished writing part 1.
INFO: 2023/05/21 14:13:58.009757 Starting part 2 ...
INFO: 2023/05/21 14:13:58.009793 /global/pg_control
INFO: 2023/05/21 14:13:58.010071 Finished writing part 2.
INFO: 2023/05/21 14:13:58.010079 Calling pg_stop_backup()
INFO: 2023/05/21 14:13:58.020444 Starting part 3 ...
INFO: 2023/05/21 14:13:58.020488 backup_label
INFO: 2023/05/21 14:13:58.020494 tablespace_map
INFO: 2023/05/21 14:13:58.020662 Finished writing part 3.
INFO: 2023/05/21 14:13:58.026890 Wrote backup with name base_00000004000000000000002E
</pre>

#### 2.6) ИТОГО ВСЕ ПОЛУЧИЛОСЬ БЕКАП НА РЕПЛИКЕ СНЯТ!!!!! 

<pre>
-bash-4.2$ /usr/local/bin/wal-g/wal-g-pg backup-list
name                          modified                  wal_segment_backup_start
base_00000004000000000000002E 2023-05-21T14:13:58+03:00 00000004000000000000002E
</pre>

1