<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Работа с большим объемом реальных данных" </h2></div>

***

> ### Подготовительные работы.
  * Создаем ВМ в YandexCloud для ДЗ
    ```console
    yc compute instance create \
      --name test-srv \
      --hostname test-srv \
      --cores 4 \
      --memory 8 \
      --create-boot-disk size=100G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
      --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
      --zone ru-central1-b \
      --core-fraction 100 \
      --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
    ```
    :hammer_and_wrench: Параметр | :memo: Значение |
    --------------:|---------------| 
    | Имя ВМ | **`test-srv`** |
    | Внешний ip | `158.160.17.150` |
    | Операционная система | `Ubuntu 20.04 LTS` |
    | Зона доступности | `ru-central1-b` |
    | Платформа | `Intel Cascade Lake	` |
    | vCPU | `4` |
    | Гарантированная доля vCPU | `100%` |
    | RAM | `8 ГБ` |
    | Тип диска | `SSD` | 
    | Объём дискового пространства | `100 ГБ` |
    | Макс. IOPS (чтение / запись) | `1000 / 1000` |
    | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
    | Прерываемая | нет |
    
  * Устанавливаем PostgreSQL 15
      * Параметры рассчитываем в [PGTune](https://pgtune.leopard.in.ua/) под параметры нашей ВМ, но `DB Type=Data warehouse` и применяем их
          ```console
          ubuntu@test-srv:~$ sudo -u postgres psql
          psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
          Type "help" for help.

          postgres=# ALTER SYSTEM SET
          postgres-#  max_connections = '40';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  shared_buffers = '2GB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  effective_cache_size = '6GB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  maintenance_work_mem = '1GB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  checkpoint_completion_target = '0.9';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  wal_buffers = '16MB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  default_statistics_target = '500';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  random_page_cost = '1.1';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  effective_io_concurrency = '200';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  work_mem = '6553kB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  min_wal_size = '4GB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  max_wal_size = '16GB';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  max_worker_processes = '8';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  max_parallel_workers_per_gather = '4';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  max_parallel_workers = '8';
          ALTER SYSTEM
          postgres=# ALTER SYSTEM SET
          postgres-#  max_parallel_maintenance_workers = '4';
          ALTER SYSTEM
          postgres=# 
          postgres=# \q
          ubuntu@test-srv:~$ sudo pg_ctlcluster 15 main restart
          ubuntu@test-srv:~$ 
          ```
      * Содаем БД для загрузок
          ```console
          ubuntu@test-srv:~$ sudo -u postgres psql -c "create database otus"
          CREATE DATABASE
          ubuntu@test-srv:~$ 
          ```
  * Устанавливаем утилиту [gsutil](https://cloud.google.com/storage/docs/gsutil_install#deb) + [gsutil_GitHub](https://github.com/GoogleCloudPlatform/gsutil)  
      ```console
      sudo apt-get update
      sudo apt-get install apt-transport-https ca-certificates gnupg curl sudo
      echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
      curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
      sudo apt-get update && sudo apt-get install google-cloud-cli
      gcloud init
      ```

  * Загружаем DataSet для ДЗ размером 10 Гб   
      ```console
      ubuntu@test-srv:~$ sudo mkdir /gset
      ubuntu@test-srv:~$ sudo chmod 777 /gset
      ubuntu@test-srv:~$ cd /gset/
      ubuntu@test-srv:/gset$ gsutil -m cp gs://chicago10/taxi.csv.* .
      ```
      ```console
      ubuntu@test-srv:/gset$ gsutil -m cp gs://chicago10/taxi.csv.* .
      Copying gs://chicago10/taxi.csv.000000000000...
      Copying gs://chicago10/taxi.csv.000000000001...                                 
      Copying gs://chicago10/taxi.csv.000000000002...                                 
      Copying gs://chicago10/taxi.csv.000000000003...                                 
      Copying gs://chicago10/taxi.csv.000000000007...                                 
      Copying gs://chicago10/taxi.csv.000000000004...
      Copying gs://chicago10/taxi.csv.000000000011...                                 
      Copying gs://chicago10/taxi.csv.000000000005...                                 
      Copying gs://chicago10/taxi.csv.000000000008...                                 
      Copying gs://chicago10/taxi.csv.000000000006...                                 
      Copying gs://chicago10/taxi.csv.000000000010...                                 
      Copying gs://chicago10/taxi.csv.000000000012...                                 
      Copying gs://chicago10/taxi.csv.000000000009...                                 
      Copying gs://chicago10/taxi.csv.000000000015...
      Copying gs://chicago10/taxi.csv.000000000013...
      Copying gs://chicago10/taxi.csv.000000000014...
      Copying gs://chicago10/taxi.csv.000000000017...
      Copying gs://chicago10/taxi.csv.000000000016...                                 
      Copying gs://chicago10/taxi.csv.000000000018...
      Copying gs://chicago10/taxi.csv.000000000019...
      Copying gs://chicago10/taxi.csv.000000000020...07.2 MiB/s ETA 00:01:16          
      Copying gs://chicago10/taxi.csv.000000000021...05.0 MiB/s ETA 00:01:18          
      Copying gs://chicago10/taxi.csv.000000000022...05.6 MiB/s ETA 00:01:13          
      Copying gs://chicago10/taxi.csv.000000000023...22.3 MiB/s ETA 00:01:02          
      Copying gs://chicago10/taxi.csv.000000000024...87.2 MiB/s ETA 00:01:23          
      Copying gs://chicago10/taxi.csv.000000000025...57.0 MiB/s ETA 00:01:56          
      Copying gs://chicago10/taxi.csv.000000000026...69.1 MiB/s ETA 00:01:29          
      Copying gs://chicago10/taxi.csv.000000000027...70.3 MiB/s ETA 00:01:28          
      Copying gs://chicago10/taxi.csv.000000000028...41.4 MiB/s ETA 00:02:29          
      Copying gs://chicago10/taxi.csv.000000000029... 77.9 MiB/s ETA 00:01:07         
      Copying gs://chicago10/taxi.csv.000000000030... 82.4 MiB/s ETA 00:01:02         
      Copying gs://chicago10/taxi.csv.000000000031... 56.7 MiB/s ETA 00:01:28         
      Copying gs://chicago10/taxi.csv.000000000032... 81.3 MiB/s ETA 00:00:59         
      Copying gs://chicago10/taxi.csv.000000000033... 54.3 MiB/s ETA 00:01:21         
      Copying gs://chicago10/taxi.csv.000000000034... 53.7 MiB/s ETA 00:01:22         
      Copying gs://chicago10/taxi.csv.000000000035... 53.1 MiB/s ETA 00:01:23         
      Copying gs://chicago10/taxi.csv.000000000036... 69.4 MiB/s ETA 00:01:00         
      Copying gs://chicago10/taxi.csv.000000000037... 65.3 MiB/s ETA 00:01:04         
      Copying gs://chicago10/taxi.csv.000000000038... 49.7 MiB/s ETA 00:01:22         
      Copying gs://chicago10/taxi.csv.000000000039... 50.6 MiB/s ETA 00:01:21         
      / [40/40 files][ 10.0 GiB/ 10.0 GiB] 100% Done  32.7 MiB/s ETA 00:00:00         
      Operation completed over 40 objects/10.0 GiB.                                    
      ubuntu@test-srv:/gset$ 
      ```
  * Загружаем загружаем данные в Postgres через COPY в psql  
      ```console
      ubuntu@test-srv:~$ sudo -u postgres psql -d otus
      
      create table taxi_trips (
      unique_key text, 
      taxi_id text, 
      trip_start_timestamp TIMESTAMP, 
      trip_end_timestamp TIMESTAMP, 
      trip_seconds bigint, 
      trip_miles numeric, 
      pickup_census_tract bigint, 
      dropoff_census_tract bigint, 
      pickup_community_area bigint, 
      dropoff_community_area bigint, 
      fare numeric, 
      tips numeric, 
      tolls numeric, 
      extras numeric, 
      trip_total numeric, 
      payment_type text, 
      company text, 
      pickup_latitude numeric, 
      pickup_longitude numeric, 
      pickup_location text, 
      dropoff_latitude numeric, 
      dropoff_longitude numeric, 
      dropoff_location text
      );
      CREATE TABLE

      COPY taxi_trips(unique_key, 
      taxi_id, 
      trip_start_timestamp, 
      trip_end_timestamp, 
      trip_seconds, 
      trip_miles, 
      pickup_census_tract, 
      dropoff_census_tract, 
      pickup_community_area, 
      dropoff_community_area, 
      fare, 
      tips, 
      tolls, 
      extras, 
      trip_total, 
      payment_type, 
      company, 
      pickup_latitude, 
      pickup_longitude, 
      pickup_location, 
      dropoff_latitude, 
      dropoff_longitude, 
      dropoff_location)
      FROM PROGRAM 'awk FNR-1 /gset/taxi.csv.* | cat' DELIMITER ',' CSV HEADER;
      
      COPY 26753682
      Time: 578395.630 ms (09:38.396)
      otus=#
      ```
  * Загружаем загружаем данные в Postgres через COPY и bash-скрипт 
      ```console
      ubuntu@test-srv:~$ sudo -u postgres psql -d otus
      psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
      Type "help" for help.

      otus=# create table taxi_trips2 (
      otus(# unique_key text, 
      otus(# taxi_id text, 
      otus(# trip_start_timestamp TIMESTAMP, 
      otus(# trip_end_timestamp TIMESTAMP, 
      otus(# trip_seconds bigint, 
      otus(# trip_miles numeric, 
      otus(# pickup_census_tract bigint, 
      otus(# dropoff_census_tract bigint, 
      otus(# pickup_community_area bigint, 
      otus(# dropoff_community_area bigint, 
      otus(# fare numeric, 
      otus(# tips numeric, 
      otus(# tolls numeric, 
      otus(# extras numeric, 
      otus(# trip_total numeric, 
      otus(# payment_type text, 
      otus(# company text, 
      otus(# pickup_latitude numeric, 
      otus(# pickup_longitude numeric, 
      otus(# pickup_location text, 
      otus(# dropoff_latitude numeric, 
      otus(# dropoff_longitude numeric, 
      otus(# dropoff_location text
      otus(# );
      CREATE TABLE
      
      postgres@test-srv:~$ vim loader.sh && chmod +x loader.sh
      ```
      
      ```bash
      postgres@test-srv:~$ cat loader.sh 
      time for f in /gset/taxi.csv.*
        do
          echo -e "Processing $f file..."
          psql -d otus -c "\\COPY taxi_trips2 FROM PROGRAM 'cat $f' CSV HEADER"
        done
      postgres@test-srv:~$ 
      ```
      
      ```console
      postgres@test-srv:~$ . loader.sh
      Processing /gset/taxi.csv.000000000000 file...
      COPY 668818
      Processing /gset/taxi.csv.000000000001 file...
      COPY 669331
      Processing /gset/taxi.csv.000000000002 file...
      COPY 668352
      Processing /gset/taxi.csv.000000000003 file...
      COPY 669666
      Processing /gset/taxi.csv.000000000004 file...
      COPY 667789
      Processing /gset/taxi.csv.000000000005 file...
      COPY 628731
      Processing /gset/taxi.csv.000000000006 file...
      COPY 635790
      Processing /gset/taxi.csv.000000000007 file...
      COPY 669381
      Processing /gset/taxi.csv.000000000008 file...
      COPY 670047
      Processing /gset/taxi.csv.000000000009 file...
      COPY 672486
      Processing /gset/taxi.csv.000000000010 file...
      COPY 669053
      Processing /gset/taxi.csv.000000000011 file...
      COPY 681872
      Processing /gset/taxi.csv.000000000012 file...
      COPY 662644
      Processing /gset/taxi.csv.000000000013 file...
      COPY 669215
      Processing /gset/taxi.csv.000000000014 file...
      COPY 919506
      Processing /gset/taxi.csv.000000000015 file...
      COPY 671838
      Processing /gset/taxi.csv.000000000016 file...
      COPY 671997
      Processing /gset/taxi.csv.000000000017 file...
      COPY 672941
      Processing /gset/taxi.csv.000000000018 file...
      COPY 669019
      Processing /gset/taxi.csv.000000000019 file...
      COPY 669107
      Processing /gset/taxi.csv.000000000020 file...
      COPY 668928
      Processing /gset/taxi.csv.000000000021 file...
      COPY 669285
      Processing /gset/taxi.csv.000000000022 file...
      COPY 670989
      Processing /gset/taxi.csv.000000000023 file...
      COPY 671264
      Processing /gset/taxi.csv.000000000024 file...
      COPY 669396
      Processing /gset/taxi.csv.000000000025 file...
      COPY 670978
      Processing /gset/taxi.csv.000000000026 file...
      COPY 680867
      Processing /gset/taxi.csv.000000000027 file...
      COPY 682085
      Processing /gset/taxi.csv.000000000028 file...
      COPY 629646
      Processing /gset/taxi.csv.000000000029 file...
      COPY 670200
      Processing /gset/taxi.csv.000000000030 file...
      COPY 679909
      Processing /gset/taxi.csv.000000000031 file...
      COPY 673360
      Processing /gset/taxi.csv.000000000032 file...
      COPY 629327
      Processing /gset/taxi.csv.000000000033 file...
      COPY 630228
      Processing /gset/taxi.csv.000000000034 file...
      COPY 631065
      Processing /gset/taxi.csv.000000000035 file...
      COPY 631470
      Processing /gset/taxi.csv.000000000036 file...
      COPY 670630
      Processing /gset/taxi.csv.000000000037 file...
      COPY 688140
      Processing /gset/taxi.csv.000000000038 file...
      COPY 628478
      Processing /gset/taxi.csv.000000000039 file...
      COPY 629855

      real    13m2.157s
      user    0m9.998s
      sys     0m40.730s
      postgres@test-srv:~$ 
      ```

***
> ### 1. Выбрать одну из СУБД
  * MySQL
    * Ставим с настройками по умолчанию по инструкции с [MySQL Installation](https://losst.pro/ustanovka-mysql-ubuntu-16-04)
      ```console
      sudo apt install mysql-server mysql-client
      ```
    * Создаем базу otus для экспериментов
      ```console
      ubuntu@test-srv:~$ sudo mysql -u root -e "create database otus"
      ubuntu@test-srv:~$ sudo mysql -u root -e "show databases"
      +--------------------+
      | Database           |
      +--------------------+
      | information_schema |
      | mysql              |
      | otus               |
      | performance_schema |
      | sys                |
      +--------------------+
      ubuntu@test-srv:~$ 
      ```

    * Создаем таблицу `taxi_trips`
      ```console
      ubuntu@test-srv:~$ sudo mysql -u root -D otus
      Enter password:
      Welcome to the MySQL monitor.  Commands end with ; or \g.
      Your MySQL connection id is 39
      Server version: 8.0.33-0ubuntu0.20.04.2 (Ubuntu)

      Copyright (c) 2000, 2023, Oracle and/or its affiliates.

      Oracle is a registered trademark of Oracle Corporation and/or its
      affiliates. Other names may be trademarks of their respective
      owners.

      Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

      mysql> SELECT DATABASE();
      +------------+
      | DATABASE() |
      +------------+
      | otus       |
      +------------+
      1 row in set (0.00 sec)

      mysql> create table taxi_trips (
          -> unique_key text,
          -> taxi_id text,
          -> trip_start_timestamp TIMESTAMP,
          -> trip_end_timestamp TIMESTAMP,
          -> trip_seconds bigint,
          -> trip_miles numeric,
          -> pickup_census_tract bigint,
          -> dropoff_census_tract bigint,
          -> pickup_community_area bigint,
          -> dropoff_community_area bigint,
          -> fare numeric,
          -> tips numeric,
          -> tolls numeric,
          -> extras numeric,
          -> trip_total numeric,
          -> payment_type text,
          -> company text,
          -> pickup_latitude numeric,
          -> pickup_longitude numeric,
          -> pickup_location text,
          -> dropoff_latitude numeric,
          -> dropoff_longitude numeric,
          -> dropoff_location text
          -> );
      Query OK, 0 rows affected (0.05 sec)

      mysql> show tables;
      +----------------+
      | Tables_in_otus |
      +----------------+
      | taxi_trips     |
      +----------------+
      1 row in set (0.00 sec)

      mysql> \q                                                                                                                                                  Bye
      ubuntu@test-srv:~$ 
      ```
    
***

> ### 2. Загрузить в неё данные (от 10 до 100 Гб)
  * Создаем `mysql_load.sh` для загрузки
    ```console
    ubuntu@test-srv:~$ vim ~/mysql_load.sh && chmod +x ~/mysql_load.sh
    ```
    ```bash
    ubuntu@test-srv:~$ cat ~/mysql_load.sh
    time for f in /gset/taxi.csv.*
    do
      echo "Processing $f file..."
      sudo mysql -u root -D otus -e "LOAD DATA INFILE '$f' INTO TABLE taxi_trips FIELDS TERMINATED BY ',' IGNORE 1 ROWS ;"
    done
    ubuntu@test-srv:~$ 
    ```
  * Загружаем данные в MySQL
    ```console
    ubuntu@test-srv:~$ ~/mysql_load.sh
    Processing /gset/taxi.csv.000000000000 file...
    Processing /gset/taxi.csv.000000000001 file...
    Processing /gset/taxi.csv.000000000002 file...
    Processing /gset/taxi.csv.000000000003 file...
    Processing /gset/taxi.csv.000000000004 file...
    Processing /gset/taxi.csv.000000000005 file...
    Processing /gset/taxi.csv.000000000006 file...
    Processing /gset/taxi.csv.000000000007 file...
    Processing /gset/taxi.csv.000000000008 file...
    Processing /gset/taxi.csv.000000000009 file...
    Processing /gset/taxi.csv.000000000010 file...
    Processing /gset/taxi.csv.000000000011 file...
    Processing /gset/taxi.csv.000000000012 file...
    Processing /gset/taxi.csv.000000000013 file...
    Processing /gset/taxi.csv.000000000014 file...
    Processing /gset/taxi.csv.000000000015 file...
    Processing /gset/taxi.csv.000000000016 file...
    Processing /gset/taxi.csv.000000000017 file...
    Processing /gset/taxi.csv.000000000018 file...
    Processing /gset/taxi.csv.000000000019 file...
    Processing /gset/taxi.csv.000000000020 file...
    Processing /gset/taxi.csv.000000000021 file...
    Processing /gset/taxi.csv.000000000022 file...
    Processing /gset/taxi.csv.000000000023 file...
    Processing /gset/taxi.csv.000000000024 file...
    Processing /gset/taxi.csv.000000000025 file...
    Processing /gset/taxi.csv.000000000026 file...
    Processing /gset/taxi.csv.000000000027 file...
    Processing /gset/taxi.csv.000000000028 file...
    Processing /gset/taxi.csv.000000000029 file...
    Processing /gset/taxi.csv.000000000030 file...
    Processing /gset/taxi.csv.000000000031 file...
    Processing /gset/taxi.csv.000000000032 file...
    Processing /gset/taxi.csv.000000000033 file...
    Processing /gset/taxi.csv.000000000034 file...
    Processing /gset/taxi.csv.000000000035 file...
    Processing /gset/taxi.csv.000000000036 file...
    Processing /gset/taxi.csv.000000000037 file...
    Processing /gset/taxi.csv.000000000038 file...
    Processing /gset/taxi.csv.000000000039 file...

    real    35m34.097s
    user    0m0.311s
    sys     0m0.280s
    ubuntu@test-srv:~$ 
    ```
***
> ### 3. Сравнить скорость выполнения запросов на PosgreSQL и выбранной СУБД
  * Text
    ```console
    ```
    
***
> ### 4. Описать что и как делали и с какими проблемами столкнулись
  * Решения для MySQL:
    * ❗️`ERROR 1290 (HY000) at line 1: The MySQL server is running with the --secure-file-priv option so it cannot execute this statement` менял параметры в `/etc/mysql/mysql.conf.d/mysqld.cnf` и перезапускал `MySQL`
      * `secure_file_priv = ""`, чтобы можно было загружать данные из определенного каталога  
      * `local_infile = 1`, чтобы можно было загружать данные из определенного каталога
    * ❗️`ERROR 29 (HY000) at line 1: File '/gset/taxi.csv.000000000039' not found (OS errno 13 - Permission denied)` [HowTo Solve](https://stackoverflow.com/questions/2783313/how-can-i-get-around-mysql-errcode-13-with-select-into-outfile)
      * Добавлял `/gset/* rw,` в `/etc/apparmor.d/usr.sbin.mysql` и обновлял `sudo /etc/init.d/apparmor reload`
    * ❗️`ERROR 1292 (22007) at line 1: Incorrect datetime value: '2017-04-13 13:00:00 UTC' for column 'trip_start_timestamp' at row 1`
      * В `DataSet` в `timestamp` используется `UTC`, что вызывает ошибку, поэтому мы менял параметр `SET GLOBAL SQL_MODE`=`ALLOW_INVALID_DATES` и перезапускал `MySQL`
    * 

***
