<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Массивно параллельные кластера PostgreSQL" </h2></div>

***
> ### Развернуть Greenplum в Yandex Cloud


  * Создаем ВМ для Postgres Client в YandexCloud для ДЗ 
    ```console
    yc compute instance create \
      --name pg-client \
      --hostname pg-client \
      --cores 4 \
      --memory 4 \
      --create-boot-disk size=30G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
      --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4 \
      --zone ru-central1-b \
      --core-fraction 100 \
      --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
    ```
    :hammer_and_wrench: Параметр | :memo: Значение |
    --------------:|---------------| 
    | Имя ВМ | **`pg-client`** |
    | Операционная система | `Ubuntu 20.04 LTS` |
    | Зона доступности | `ru-central1-b` |
    | Платформа | `Intel Cascade Lake	` |
    | vCPU | `4` |
    | Гарантированная доля vCPU | `100%` |
    | RAM | `4 ГБ` |
    | Тип диска | `SSD` | 
    | Объём дискового пространства | `30 ГБ` |
    | Макс. IOPS (чтение / запись) | `1000 / 1000` |
    | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
    | Прерываемая | `нет` |
    
  * Устанавливаем PostgreSQL Client
    ```bash
    sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-client
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
      ```bash
      ubuntu@pg-client:~$ sudo mkdir /gset
      ubuntu@pg-client:~$ sudo chmod 777 /gset
      ubuntu@pg-client:~$ cd /gset/
      ubuntu@pg-client:/gset$ gsutil -m cp gs://chicago10/taxi.csv.* .
      ```
      ```console
      ubuntu@pg-client:/gset$ gsutil -m cp gs://chicago10/taxi.csv.* .
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
      ubuntu@pg-client:/gset$ 
      ```
  * Создаем GreenPlum Service в YandexCloud 
    :hammer_and_wrench: Параметр | :memo: Значение |
    --------------:|---------------| 
    | Имя кластера | **`greenplum-otus`** |
    | Окружение | `PRESTABLE` |
    | Версия | `6.22` |
    | Зона доступности | `ru-central1-b` |
    | Публичный доступ | `да` |
    | Имя пользователя | `gpuser` |
    | Доступ из консоли управления | `да` |
    | Master server | `1` |
    | Segment server | `2` |
    | Менеджер подключений(Режим) | `SESSION` | 


***
> ### Потесировать dataset с чикагскими такси
> ### Или залить 10Гб данных и протестировать скорость запросов в сравнении с 1 инстансом PostgreSQL
  * Создаем таблицу
    ```console
    ubuntu@pg-client:~$ psql "host=rc1b-lentkjkk726l2qua.mdb.yandexcloud.net,rc1b-tj4tptft4opim2bd.mdb.yandexcloud.net \
          port=6432 \
          sslmode=verify-full \
          dbname=postgres \
          user=gpuser \
          target_session_attrs=read-write"
    Password for user gpuser: 
    
    psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1), server 9.4.26)
    SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
    Type "help" for help.
    
    postgres=> select version();
                                                                                                          version                                                                                                      
    -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
     PostgreSQL 9.4.26 (Greenplum Database 6.22.2-mdb+dev.2.g7e0a4f3ab0 build dev-oss) on x86_64-pc-linux-gnu, compiled by gcc-6 (Ubuntu 6.5.0-2ubuntu1~18.04) 6.5.0 20181026, 64-bit compiled on Mar  9 2023 13:06:19
    (1 row)
     
    postgres=> 
    postgres=> create table taxi_trips (
    postgres(> unique_key text,
    postgres(> taxi_id text,
    postgres(> trip_start_timestamp TIMESTAMP,
    postgres(> trip_end_timestamp TIMESTAMP,
    postgres(> trip_seconds bigint,
    postgres(> trip_miles numeric,
    postgres(> pickup_census_tract bigint,
    postgres(> dropoff_census_tract bigint,
    postgres(> pickup_community_area bigint,
    postgres(> dropoff_community_area bigint,
    postgres(> fare numeric,
    postgres(> tips numeric,
    postgres(> tolls numeric,
    postgres(> extras numeric,
    postgres(> trip_total numeric,
    postgres(> payment_type text,
    postgres(> company text,
    postgres(> pickup_latitude numeric,
    postgres(> pickup_longitude numeric,
    postgres(> pickup_location text,
    postgres(> dropoff_latitude numeric,
    postgres(> dropoff_longitude numeric,
    postgres(> dropoff_location text
    postgres(> );
    NOTICE:  Table doesn't have 'DISTRIBUTED BY' clause -- Using column named 'unique_key' as the Greenplum Database data distribution key for this table.
    HINT:  The 'DISTRIBUTED BY' clause determines the distribution of data. Make sure column(s) chosen are the optimal data distribution key to minimize skew.
    CREATE TABLE
    postgres=> \dt
              List of relations
     Schema |    Name    | Type  | Owner  
    --------+------------+-------+--------
     public | taxi_trips | table | gpuser
    (1 row)
    
    postgres=> 
    ```
  * Загружаем загружаем данные в Postgres через [COPY](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/ref_guide-sql_commands-COPY.html?hWord=N4IghgNiBcIDpwMYHsAOBPEBfIA) в psql  
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
  * Загружаем загружаем данные в GreenPlum через COPY и bash-скрипт 
      ```bash
      ubuntu@pg-client:~$ vim loader.sh && chmod +x loader.sh
      
      ubuntu@pg-client:~$ cat loader.sh 
      time for f in /gset/taxi.csv.*
        do
          echo -e "Processing $f file..."
          psql "host=rc1b-lentkjkk726l2qua.mdb.yandexcloud.net,rc1b-tj4tptft4opim2bd.mdb.yandexcloud.net \
            port=6432 \
            sslmode=verify-full \
            dbname=postgres \
            user=gpuser \
                password=gpuser \
            target_session_attrs=read-write" \
            -c "\COPY taxi_trips FROM '$f' WITH (FORMAT csv, DELIMITER ',', HEADER 'true');"
        done
      ubuntu@pg-client:~$ 
      ```
      ```bash
      time for f in /gset/taxi.csv.*
        do
          echo -e "Processing $f file..."
          psql "host=rc1b-lentkjkk726l2qua.mdb.yandexcloud.net,rc1b-tj4tptft4opim2bd.mdb.yandexcloud.net \
            port=6432 \
            sslmode=verify-full \
            dbname=postgres \
            user=gpuser \
                password=gpuser \
            target_session_attrs=read-write" \
            -c "\COPY taxi_trips FROM '$f' WITH (FORMAT csv, DELIMITER ',', HEADER 'true');"
        done
      ```
      
      ```console
      ubuntu@pg-client:~$ . loader.sh 
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
      
      real    11m43.367s
      user    0m15.529s
      sys     0m20.323s
      ubuntu@pg-client:~$ 
      ```
      
***      
> ### Сравнить скорость выполнения запросов на GreenPlum и PosgreSQL
  * Данные по загрузке по PostgreSQL взял из [своей ДЗ](https://github.com/EvgeniyKorukov/OTUS_Learning_PostgreSQL-Cloud-Solutions/blob/main/Home_Works/HomeWork_8_(lesson_10).md)
  * Результы тестов
    | Тест | GreenPlum | PostgreSQL |
    |--------------|---------------|---------------| 
    | Загрузка через консоль | `---` | `578395.630 ms (09:38.396)` |
    | Загрузка через bash-скрипт | `11m43.367s` | `13m2.157s` |
    | Запрос количества записей | `0m2.509s` | `3m21.361s` |
    | Запрос из BigQuery | `0m7.027ss` | `3m20.996s` |
    * Запрос из BigQuery
      ```sql
      SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c 
      FROM taxi_trips
      group by payment_type
      order by 3;
      ```
    * Логи замеров
      ```console
      ubuntu@pg-client:~$ time psql "host=rc1b-lentkjkk726l2qua.mdb.yandexcloud.net,rc1b-tj4tptft4opim2bd.mdb.yandexcloud.net \
            port=6432 \
            sslmode=verify-full \
            dbname=postgres \
            user=gpuser \
        password=gpuser \
            target_session_attrs=read-write" -c "select count(*) from taxi_trips;"
        count   
      ----------
       26753683
      (1 row)
      
      
      real    0m2.509s
      user    0m0.035s
      sys     0m0.005s
      ubuntu@pg-client:~$ 
      
      ubuntu@test-srv:~$ time sudo -u postgres psql -d otus -c "select count(*) from taxi_trips;"
        count
      ----------
       26753682
      (1 row)


      real    3m21.361s
      user    0m0.032s
      sys     0m0.027s
      ubuntu@test-srv:~$

      ubuntu@pg-client:~$ time psql "host=rc1b-lentkjkk726l2qua.mdb.yandexcloud.net,rc1b-tj4tptft4opim2bd.mdb.yandexcloud.net \
            port=6432 \
            sslmode=verify-full \
            dbname=postgres \
            user=gpuser \
        password=gpuser \
            target_session_attrs=read-write" -c "SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c FROM taxi_trips group by payment_type order by 3;"
       payment_type | tips_percent |    c     
      --------------+--------------+----------
       Prepaid      |            0 |        6
       Way2ride     |           12 |       27
       Split        |           17 |      180
       Dispute      |            0 |     5596
       Pcard        |            2 |    13575
       No Charge    |            0 |    26294
       Mobile       |           16 |    61256
       Prcard       |            1 |    86053
       Unknown      |            0 |   103869
       Credit Card  |           17 |  9224956
       Cash         |            0 | 17231871
      (11 rows)
      
      
      real    0m7.027s
      user    0m0.028s
      sys     0m0.012s
      ubuntu@pg-client:~$ 

      
      ubuntu@test-srv:~$ time sudo -u postgres psql -d otus -c "SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c FROM taxi_trips group by payment_type order by 3;"
      | payment_type | tips_percent |    c
      --------------+--------------+----------
       Prepaid      |            0 |        6
       Way2ride     |           12 |       27
       Split        |           17 |      180
       Dispute      |            0 |     5596
       Pcard        |            2 |    13575
       No Charge    |            0 |    26294
       Mobile       |           16 |    61256
       Prcard       |            1 |    86053
       Unknown      |            0 |   103869
       Credit Card  |           17 |  9224956
       Cash         |            0 | 17231870
      (11 rows)


      real    3m20.996s
      user    0m0.048s
      sys     0m0.013s
      ubuntu@test-srv:~$
      ```
***

> ### Описать что и как делали и с какими проблемами столкнулись
  * Трудностей, как таковых, не было. Тут скорее борьба с ограничениями YandexCloud
  * GreenPlum хорош при правильном распределении данных по сегментным серверам (ключ дистрибуции [DISTRIBUTED BY](https://docs.vmware.com/en/VMware-Greenplum/7/greenplum-database/admin_guide-ddl-ddl-table.html?hWord=N4IghgNiBcICIEkDKAVASggQgVRQUTgAJMBNEAXyA)). В моем случае ключ дистрибуции `DISTRIBUTED RANDOMLY`, но у меня и два сегментных сервера-как раз самое то.
  * В YandexCloud есть ограничения на то, что на мастере GreenPlum нельязя поставить утилиты для загрузки данных, такие как [gpfdist](https://docs.vmware.com/en/VMware-Greenplum/6/greenplum-database/utility_guide-ref-gpfdist.html) или [gpload](https://docs.vmware.com/en/VMware-Greenplum/6/greenplum-database/utility_guide-ref-gpload.html)
  * Пять месяцев назад проходил курсы администраторов по ADB (ArenaData, GreenPlum) и сдавал сложный экзамен, чтобы дали сертификат. Плюс, поработал с ним на BareMetal и пощупал его в работе, поэтому было не сложно.
