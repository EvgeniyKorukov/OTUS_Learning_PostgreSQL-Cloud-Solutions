<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Работа с горизонтально масштабируемым кластером" </h2></div>

***

> ### Развернуть CockroachDB в GKE или GCE
  * Создаем 3 ВМ в YandexCloud для CockroachDB
      * Общие параметры для всех 3х ВМ с фиксированным внутренним IPv4.   
         :hammer_and_wrench: Параметр | :memo: Значение |
        --------------:|---------------| 
        | Операционная система | `Ubuntu 20.04 LTS` |
        | Зона доступности | `ru-central1-b` |
        | Платформа | `Intel Cascade Lake	` |
        | vCPU | `2` |
        | Гарантированная доля vCPU | `100%` |
        | RAM | `4 ГБ` |
        | Тип диска | `SSD` | 
        | Объём дискового пространства | `30 ГБ` |
        | Макс. IOPS (чтение / запись) | `1000 / 1000` |
        | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
        | Прерываемая | :ballot_box_with_check: |
    * Индивидуальные параметры каждой ВМ:
        :hammer_and_wrench: Название ВМ | :memo: Внутренний IPv4 | Описание |
        --------------:|---------------|---------------|
        | **`pg-cdb1`** | `10.129.0.21` | Сервер с CockroachDB |      
        | **`pg-cdb2`** | `10.129.0.22` | Сервер с CockroachDB |  
        | **`pg-cdb3`** | `10.129.0.23` | Сервер с CockroachDB |          

     * Создание трех ВМ `pg-cdb*`
       ```console
       yc compute instance create \
         --name pg-cdb1 \
         --hostname pg-cdb1 \
         --cores 2 \
         --memory 4 \
         --create-boot-disk size=30G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.21 \
         --zone ru-central1-b \
         --core-fraction 100 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       yc compute instance create \
         --name pg-cdb2 \
         --hostname pg-cdb2 \
         --cores 2 \
         --memory 4 \
         --create-boot-disk size=30G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.22 \
         --zone ru-central1-b \
         --core-fraction 100 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       yc compute instance create \
         --name pg-cdb3 \
         --hostname pg-cdb3 \
         --cores 2 \
         --memory 4 \
         --create-boot-disk size=30G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.23 \
         --zone ru-central1-b \
         --core-fraction 100 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       ```
  * Разворачиваем и запускаем CockroachDB
    * Скачиваем, меняем права. Делаем на всех 3х ВМ
      ```console
      wget -qO- https://binaries.cockroachdb.com/cockroach-v21.1.6.linux-amd64.tgz | tar  xvz && sudo cp -i cockroach-v21.1.6.linux-amd64/cockroach /usr/local/bin/ && sudo mkdir -p /opt/cockroach && sudo chown ubuntu:ubuntu /opt/cockroach
      ```
    * Генерируем сертикаты. ❗️Только на одной ноде
      ```console
      mkdir certs my-safe-directory
      cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
      cockroach cert create-node localhost pg-cdb1 pg-cdb2 pg-cdb3 --certs-dir=certs --ca-key=my-safe-directory/ca.key --overwrite
      cockroach cert create-client root --certs-dir=certs --ca-key=my-safe-directory/ca.key
      ```
    * Проверяем статус сертикатов там, где их генерировали.
      ```console    
      cockroach cert list --certs-dir=certs

      ubuntu@pg-cdb1:~$ cockroach cert list --certs-dir=certs
      Certificate directory: certs
        Usage  | Certificate File |    Key File     |  Expires   |                    Notes                     | Error
      ---------+------------------+-----------------+------------+----------------------------------------------+--------
        CA     | ca.crt           |                 | 2033/07/09 | num certs: 1                                 |
        Node   | node.crt         | node.key        | 2028/07/05 | addresses: localhost,pg-cdb1,pg-cdb2,pg-cdb3 |
        Client | client.root.crt  | client.root.key | 2028/07/05 | user: root                                   |
      (3 rows)
      ubuntu@pg-cdb1:~$ 
      ```
      
    * Копируем сертикаты на остальные ноды. Сначала изменив пароль у пользователя `ubuntu`. По умолчанию мы его не знаем, но нам ничего не мешает зайти в root `sudo su -` или `sudo passwd ubuntu` и там поменять его
      ```console
      scp -r /home/ubuntu/certs ubuntu@10.129.0.22:/home/ubuntu
      scp -r /home/ubuntu/certs ubuntu@10.129.0.23:/home/ubuntu
      ```
      ```console
      ubuntu@pg-cdb1:~$ scp -r /home/ubuntu/certs ubuntu@10.129.0.22:/home/ubuntu
      The authenticity of host '10.129.0.22 (10.129.0.22)' can't be established.
      ECDSA key fingerprint is SHA256:qaCFcn1ZW7XRRwIjXO5aAETwGyPG1+OcU0ZB+9U/4N4.
      Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
      Warning: Permanently added '10.129.0.22' (ECDSA) to the list of known hosts.
      ubuntu@10.129.0.22's password: 
      client.root.crt                                                                   100% 1143     1.6MB/s   00:00    
      node.key                                                                          100% 1679     3.3MB/s   00:00    
      ca.crt                                                                            100% 1151     3.1MB/s   00:00    
      client.root.key                                                                   100% 1675     3.9MB/s   00:00    
      node.crt                                                                          100% 1229     3.2MB/s   00:00    
      ubuntu@pg-cdb1:~$ 
      ubuntu@pg-cdb1:~$ scp -r /home/ubuntu/certs ubuntu@10.129.0.23:/home/ubuntu
      The authenticity of host '10.129.0.23 (10.129.0.23)' can't be established.
      ECDSA key fingerprint is SHA256:70EW9dfrAEUqj7ZBT7lVh5aXQdrKMzuj5zdSyPX6syg.
      Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
      Warning: Permanently added '10.129.0.23' (ECDSA) to the list of known hosts.
      ubuntu@10.129.0.23's password: 
      client.root.crt                                                                   100% 1143     1.6MB/s   00:00    
      node.key                                                                          100% 1679     3.5MB/s   00:00    
      ca.crt                                                                            100% 1151     2.8MB/s   00:00    
      client.root.key                                                                   100% 1675     3.8MB/s   00:00    
      node.crt                                                                          100% 1229     1.9MB/s   00:00    
      ubuntu@pg-cdb1:~$ 
      ```
    * Стартуем `CockroachDB` на всех ВМ, разница только параметре `--advertise-addr=pg-cdb*`
      ```console
      cockroach start --certs-dir=certs --advertise-addr=pg-cdb1 --join=pg-cdb1,pg-cdb2,pg-cdb3 --cache=.25 --max-sql-memory=.25 --background

      ubuntu@pg-cdb1:~$ cockroach start --certs-dir=certs --advertise-addr=pg-cdb1 --join=pg-cdb1,pg-cdb2,pg-cdb3 --cache=.25 --max-sql-memory=.25 --background
      *
      * INFO: initial startup completed.
      * Node will now attempt to join a running cluster, or wait for `cockroach init`.
      * Client connections will be accepted after this completes successfully.
      * Check the log file(s) for progress. 
      *
      ubuntu@pg-cdb1:~$ 
      ```
      ```console
      cockroach start --certs-dir=certs --advertise-addr=pg-cdb2 --join=pg-cdb1,pg-cdb2,pg-cdb3 --cache=.25 --max-sql-memory=.25 --background

      ubuntu@pg-cdb2:~$ cockroach start --certs-dir=certs --advertise-addr=pg-cdb2 --join=pg-cdb1,pg-cdb2,pg-cdb3 --cache=.25 --max-sql-memory=.25 --background
      *
      * INFO: initial startup completed.
      * Node will now attempt to join a running cluster, or wait for `cockroach init`.
      * Client connections will be accepted after this completes successfully.
      * Check the log file(s) for progress. 
      *
      ubuntu@pg-cdb2:~$ 
      ```
       ```console
      cockroach start --certs-dir=certs --advertise-addr=pg-cdb3 --join=pg-cdb1,pg-cdb2,pg-cdb3 --cache=.25 --max-sql-memory=.25 --background

      ubuntu@pg-cdb3:~$ cockroach start --certs-dir=certs --advertise-addr=pg-cdb3 --join=pg-cdb1,pg-cdb2,pg-cdb3 --cache=.25 --max-sql-memory=.25 --background
      *
      * INFO: initial startup completed.
      * Node will now attempt to join a running cluster, or wait for `cockroach init`.
      * Client connections will be accepted after this completes successfully.
      * Check the log file(s) for progress. 
      *
      ubuntu@pg-cdb3:~$ 
      ```     
     * Выполняем инициализацию. ❗️Только на одной ноде
       ```console
       cockroach init --certs-dir=certs --host=pg-cdb1
       
       ubuntu@pg-cdb1:~$ cockroach init --certs-dir=certs --host=pg-cdb1
       Cluster successfully initialized
       ubuntu@pg-cdb1:~$       
       ```
    * Смотрим статус
      ```console
      cockroach node status --certs-dir=certs

      ubuntu@pg-cdb1:~$ cockroach node status --certs-dir=certs
        id |    address    |  sql_address  |  build  |         started_at         |         updated_at         | locality | is_available | is_live
      -----+---------------+---------------+---------+----------------------------+----------------------------+----------+--------------+----------
         1 | pg-cdb1:26257 | pg-cdb1:26257 | v21.1.6 | 2023-07-02 13:50:15.428139 | 2023-07-02 13:50:42.455051 |          | true         | true
         2 | pg-cdb3:26257 | pg-cdb3:26257 | v21.1.6 | 2023-07-02 13:50:16.891729 | 2023-07-02 13:50:43.904164 |          | true         | true
         3 | pg-cdb2:26257 | pg-cdb2:26257 | v21.1.6 | 2023-07-02 13:50:16.999445 | 2023-07-02 13:50:44.035955 |          | true         | true
      (3 rows)
      ubuntu@pg-cdb1:~$ 
      ```    
 
***      
> ### Потесировать dataset с чикагскими такси
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
      ubuntu@pg-cdb1:~$ sudo mkdir /gset
      ubuntu@pg-cdb1:~$ sudo chmod 777 /gset
      ubuntu@pg-cdb1:~$ cd /gset/
      ubuntu@pg-cdb1:/gset$ gsutil -m cp gs://chicago10/taxi.csv.* .
      ```
      ```console
      ubuntu@pg-cdb1:/gset$ gsutil -m cp gs://chicago10/taxi.csv.* .
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
      ubuntu@pg-cdb1:/gset$ 
      ```
***      
> ### Или залить 10Гб данных и протестировать скорость запросов в сравнении с 1 инстансом PostgreSQL
 * Создаем базу
   ```console
   ubuntu@pg-cdb1:~$ cockroach sql --certs-dir=certs
   #
   # Welcome to the CockroachDB SQL shell.
   # All statements must be terminated by a semicolon.
   # To exit, type: \q.
   #
   # Server version: CockroachDB CCL v21.1.6 (x86_64-unknown-linux-gnu, built 2021/07/20 15:30:39, go1.15.11) (same version as client)
   # Cluster ID: 30351012-bf45-4551-9d58-fc462e21d3d0
   No entry for terminal type "xterm-256color";
   using dumb terminal settings.
   #
   # Enter \? for a brief introduction.
   #
   root@:26257/defaultdb> \l
     database_name | owner | primary_region | regions | survival_goal
   ----------------+-------+----------------+---------+----------------
     defaultdb     | root  | NULL           | {}      | NULL
     postgres      | root  | NULL           | {}      | NULL
     system        | node  | NULL           | {}      | NULL
   (3 rows)
   
   Time: 5ms total (execution 4ms / network 0ms)
   
   root@:26257/defaultdb> create database otus;
   CREATE DATABASE
   
   Time: 53ms total (execution 53ms / network 0ms)
   
   root@:26257/defaultdb> \l
     database_name | owner | primary_region | regions | survival_goal
   ----------------+-------+----------------+---------+----------------
     defaultdb     | root  | NULL           | {}      | NULL
     otus          | root  | NULL           | {}      | NULL
     postgres      | root  | NULL           | {}      | NULL
     system        | node  | NULL           | {}      | NULL
   (4 rows)
   
   Time: 4ms total (execution 4ms / network 0ms)
   
   root@:26257/defaultdb> 
   ```
 * Создаем таблицу
   ```console
   ubuntu@pg-cdb1:~$ cockroach sql --certs-dir=certs -d otus
   #
   # Welcome to the CockroachDB SQL shell.
   # All statements must be terminated by a semicolon.
   # To exit, type: \q.
   #
   # Server version: CockroachDB CCL v21.1.6 (x86_64-unknown-linux-gnu, built 2021/07/20 15:30:39, go1.15.11) (same version as client)
   # Cluster ID: 30351012-bf45-4551-9d58-fc462e21d3d0
   No entry for terminal type "xterm-256color";
   using dumb terminal settings.
   #
   # Enter \? for a brief introduction.
   #
   root@:26257/otus> create table taxi_trips (
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
   
   Time: 69ms total (execution 69ms / network 0ms)
   
   root@:26257/otus> 
   ```
 * Загружать данные в `cockroach` используя [IMPORT INTO](https://www.cockroachlabs.com/docs/stable/import-into.html) через [cockroach nodelocal upload](https://www.cockroachlabs.com/docs/v21.1/cockroach-nodelocal-upload#upload-a-file)
   * Скрипт для загрузки `taxi_trips`
     ```bash
     ubuntu@pg-cdb1:~$ cat loader.sh 
     clear 
     
     #Upload into localnode
     time for f in /gset/taxi.csv.*
       do
         # sf - short file name or name without full path
         sf=`echo $f | sed "s/.*\///"`
         echo -e "Processing $sf file..."
         cockroach --certs-dir=/home/ubuntu/certs nodelocal upload $f $sf 
       done
     
     #Import CSV's into taxi_trips
     time for f in /home/ubuntu/cockroach-data/extern/taxi.csv.*
       do
         sf=`echo $f | sed "s/.*\///"`
         echo -e "Processing $sf file..."
         cockroach --certs-dir=/home/ubuntu/certs sql -d otus -e "IMPORT INTO taxi_trips (unique_key, taxi_id, trip_start_timestamp, trip_end_timestamp, trip_seconds, trip_miles, pickup_census_tract, dropoff_census_tract, pickup_community_area, dropoff_community_area, fare, tips, tolls, extras, trip_total, payment_type, company, pickup_latitude, pickup_longitude, pickup_location, dropoff_latitude, dropoff_longitude, dropoff_location) CSV DATA ('nodelocal://1/$sf') WITH DELIMITER=',', skip='1',nullif = '';" 
       done
     ubuntu@pg-cdb1:~$ 
     ```
     ```console
     ubuntu@pg-cdb1:~$ . loader.sh 
     Processing taxi.csv.000000000000 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000000
     Processing taxi.csv.000000000001 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000001
     Processing taxi.csv.000000000002 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000002
     Processing taxi.csv.000000000003 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000003
     Processing taxi.csv.000000000004 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000004
     Processing taxi.csv.000000000005 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000005
     Processing taxi.csv.000000000006 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000006
     Processing taxi.csv.000000000007 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000007
     Processing taxi.csv.000000000008 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000008
     Processing taxi.csv.000000000009 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000009
     Processing taxi.csv.000000000010 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000010
     Processing taxi.csv.000000000011 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000011
     Processing taxi.csv.000000000012 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000012
     Processing taxi.csv.000000000013 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000013
     Processing taxi.csv.000000000014 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000014
     Processing taxi.csv.000000000015 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000015
     Processing taxi.csv.000000000016 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000016
     Processing taxi.csv.000000000017 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000017
     Processing taxi.csv.000000000018 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000018
     Processing taxi.csv.000000000019 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000019
     Processing taxi.csv.000000000020 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000020
     Processing taxi.csv.000000000021 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000021
     Processing taxi.csv.000000000022 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000022
     Processing taxi.csv.000000000023 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000023
     Processing taxi.csv.000000000024 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000024
     Processing taxi.csv.000000000025 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000025
     Processing taxi.csv.000000000026 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000026
     Processing taxi.csv.000000000027 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000027
     Processing taxi.csv.000000000028 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000028
     Processing taxi.csv.000000000029 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000029
     Processing taxi.csv.000000000030 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000030
     Processing taxi.csv.000000000031 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000031
     Processing taxi.csv.000000000032 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000032
     Processing taxi.csv.000000000033 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000033
     Processing taxi.csv.000000000034 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000034
     Processing taxi.csv.000000000035 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000035
     Processing taxi.csv.000000000036 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000036
     Processing taxi.csv.000000000037 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000037
     Processing taxi.csv.000000000038 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000038
     Processing taxi.csv.000000000039 file...
     successfully uploaded to nodelocal://1/taxi.csv.000000000039
     
     real    19m54.853s
     user    1m20.522s
     sys     0m36.579s
     
     
     Processing taxi.csv.000000000000 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730314503782401 | succeeded |                  1 | 668818 |             0 | 243990434
     (1 row)
     
     Time: 31.349s
     
     Processing taxi.csv.000000000001 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730417792581633 | succeeded |                  1 | 669331 |             0 | 244070640
     (1 row)
     
     Time: 29.101s
     
     Processing taxi.csv.000000000002 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730513654284289 | succeeded |                  1 | 668352 |             0 | 243995091
     (1 row)
     
     Time: 29.131s
     
     Processing taxi.csv.000000000003 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730609696374785 | succeeded |                  1 | 669666 |             0 | 244137027
     (1 row)
     
     Time: 33.895s
     
     Processing taxi.csv.000000000004 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730721297661953 | succeeded |                  1 | 667789 |             0 | 247225345
     (1 row)
     
     Time: 29.814s
     
     Processing taxi.csv.000000000005 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730819476226049 | succeeded |                  1 | 628731 |             0 | 246075934
     (1 row)
     
     Time: 35.760s
     
     Processing taxi.csv.000000000006 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881730937171902465 | succeeded |                  1 | 635790 |             0 | 245463528
     (1 row)
     
     Time: 39.047s
     
     Processing taxi.csv.000000000007 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731065634291713 | succeeded |                  1 | 669381 |             0 | 243913294
     (1 row)
     
     Time: 41.837s
     
     Processing taxi.csv.000000000008 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731203268476929 | succeeded |                  1 | 670047 |             0 | 243988176
     (1 row)
     
     Time: 34.642s
     
     Processing taxi.csv.000000000009 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731317273001985 | succeeded |                  1 | 672486 |             0 | 244308572
     (1 row)
     
     Time: 29.092s
     
     Processing taxi.csv.000000000010 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731413098561537 | succeeded |                  1 | 669053 |             0 | 243862287
     (1 row)
     
     Time: 32.095s
     
     Processing taxi.csv.000000000011 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731518857281537 | succeeded |                  1 | 681872 |             0 | 244005898
     (1 row)
     
     Time: 28.325s
     
     Processing taxi.csv.000000000012 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731612166258689 | succeeded |                  1 | 662644 |             0 | 243582979
     (1 row)
     
     Time: 34.399s
     
     Processing taxi.csv.000000000013 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731725457129473 | succeeded |                  1 | 669215 |             0 | 243835921
     (1 row)
     
     Time: 34.784s
     
     Processing taxi.csv.000000000014 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731839976767489 | succeeded |                  1 | 919506 |             0 | 248963706
     (1 row)
     
     Time: 31.233s
     
     Processing taxi.csv.000000000015 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881731942815301633 | succeeded |                  1 | 671838 |             0 | 243854536
     (1 row)
     
     Time: 30.642s
     
     Processing taxi.csv.000000000016 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732043724324865 | succeeded |                  1 | 671997 |             0 | 243910744
     (1 row)
     
     Time: 33.621s
     
     Processing taxi.csv.000000000017 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732154396016641 | succeeded |                  1 | 672941 |             0 | 243840376
     (1 row)
     
     Time: 28.396s
     
     Processing taxi.csv.000000000018 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732247959568385 | succeeded |                  1 | 669019 |             0 | 243887799
     (1 row)
     
     Time: 32.867s
     
     Processing taxi.csv.000000000019 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732356157046785 | succeeded |                  1 | 669107 |             0 | 243837378
     (1 row)
     
     Time: 36.695s
     
     Processing taxi.csv.000000000020 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732476878782465 | succeeded |                  1 | 668928 |             0 | 243829342
     (1 row)
     
     Time: 29.515s
     
     Processing taxi.csv.000000000021 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732574097473537 | succeeded |                  1 | 669285 |             0 | 243853441
     (1 row)
     
     Time: 32.702s
     
     Processing taxi.csv.000000000022 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732681733046273 | succeeded |                  1 | 670989 |             0 | 243866465
     (1 row)
     
     Time: 34.993s
     
     Processing taxi.csv.000000000023 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732796937535489 | succeeded |                  1 | 671264 |             0 | 243934166
     (1 row)
     
     Time: 34.753s
     
     Processing taxi.csv.000000000024 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881732911396356097 | succeeded |                  1 | 669396 |             0 | 243839239
     (1 row)
     
     Time: 37.543s
     
     Processing taxi.csv.000000000025 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733034913497089 | succeeded |                  1 | 670978 |             0 | 243911613
     (1 row)
     
     Time: 30.176s
     
     Processing taxi.csv.000000000026 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733134365556737 | succeeded |                  1 | 680867 |             0 | 244041048
     (1 row)
     
     Time: 41.058s
     
     Processing taxi.csv.000000000027 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733269489778689 | succeeded |                  1 | 682085 |             0 | 244126765
     (1 row)
     
     Time: 35.411s
     
     Processing taxi.csv.000000000028 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733386294427649 | succeeded |                  1 | 629646 |             0 | 246176382
     (1 row)
     
     Time: 35.882s
     
     Processing taxi.csv.000000000029 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733504736493569 | succeeded |                  1 | 670200 |             0 | 243834923
     (1 row)
     
     Time: 28.818s
     
     Processing taxi.csv.000000000030 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733599755567105 | succeeded |                  1 | 679909 |             0 | 244281952
     (1 row)
     
     Time: 32.571s
     
     Processing taxi.csv.000000000031 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733707098685441 | succeeded |                  1 | 673360 |             0 | 243895674
     (1 row)
     
     Time: 33.198s
     
     Processing taxi.csv.000000000032 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733816392646657 | succeeded |                  1 | 629327 |             0 | 246153220
     (1 row)
     
     Time: 31.558s
     
     Processing taxi.csv.000000000033 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881733920324517889 | succeeded |                  1 | 630228 |             0 | 246215221
     (1 row)
     
     Time: 29.118s
     
     Processing taxi.csv.000000000034 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881734016252444673 | succeeded |                  1 | 631065 |             0 | 246327369
     (1 row)
     
     Time: 34.045s
     
     Processing taxi.csv.000000000035 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881734128330375169 | succeeded |                  1 | 631470 |             0 | 246382250
     (1 row)
     
     Time: 34.490s
     
     Processing taxi.csv.000000000036 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881734241879228417 | succeeded |                  1 | 670630 |             0 | 244037752
     (1 row)
     
     Time: 34.299s
     
     Processing taxi.csv.000000000037 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881734355040043009 | succeeded |                  1 | 688140 |             0 | 244239156
     (1 row)
     
     Time: 30.536s
     
     Processing taxi.csv.000000000038 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881734455767367681 | succeeded |                  1 | 628478 |             0 | 246058457
     (1 row)
     
     Time: 30.968s
     
     Processing taxi.csv.000000000039 file...
             job_id       |  status   | fraction_completed |  rows  | index_entries |   bytes
     ---------------------+-----------+--------------------+--------+---------------+------------
       881734558032789505 | succeeded |                  1 | 629855 |             0 | 246219602
     (1 row)
     
     Time: 31.439s
     
     
     real    22m6.629s
     user    0m5.333s
     sys     0m2.789s
     ubuntu@pg-cdb1:~$ 
     ```
   
***      
> ### Сравнить скорость выполнения запросов на PosgreSQL и CockroachDB
  * Данные по загрузке по PostgreSQL взял из [своей ДЗ](https://github.com/EvgeniyKorukov/OTUS_Learning_PostgreSQL-Cloud-Solutions/blob/main/Home_Works/HomeWork_8_(lesson_10).md)
  * Результы тестов
    | Тест | CockroachDB | PostgreSQL |
    |--------------|---------------|---------------| 
    | Загрузка через консоль | `---` | `578395.630 ms (09:38.396)` |
    | Загрузка через bash-скрипт | `22m6.629s` | `13m2.157s` |
    | Запрос количества записей | `2m45.450s` | `3m21.361s` |
    | Запрос из BigQuery | `2m45.842s` | `3m20.996s` |
    * Запрос из BigQuery
      ```sql
      SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c 
      FROM taxi_trips
      group by payment_type
      order by 3;
      ```
    * Логи замеров
      ```console
      ubuntu@pg-cdb1:~$ time cockroach --certs-dir=/home/ubuntu/certs sql -d otus -e "select count(*) from taxi_trips;"  
         count
      ------------
        26753683
      (1 row)
      
      Time: 163.118s
      
      
      real    2m45.450s
      user    0m0.296s
      sys     0m0.151s
      ubuntu@pg-cdb1:~$ 
      
      ubuntu@test-srv:~$ time sudo -u postgres psql -d otus -c "select count(*) from taxi_trips;"
        count
      ----------
       26753682
      (1 row)


      real    3m21.361s
      user    0m0.032s
      sys     0m0.027s
      ubuntu@test-srv:~$

      ubuntu@pg-cdb1:~$ time cockroach --certs-dir=/home/ubuntu/certs sql -d otus -e "SELECT payment_type, round(sum(tips)/sum(trip_total)*100, 0) + 0 as tips_percent, count(*) as c FROM taxi_trips group by payment_type order by 3;"
        payment_type | tips_percent |    c
      ---------------+--------------+-----------
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
      
      Time: 165.709s
      
      
      real    2m45.842s
      user    0m0.228s
      sys     0m0.101s
      ubuntu@pg-cdb1:~$ 
      
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
    

> ### Описать что и как делали и с какими проблемами столкнулись
***

