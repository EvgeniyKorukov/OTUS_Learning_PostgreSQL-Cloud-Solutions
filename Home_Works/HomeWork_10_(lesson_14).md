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
> ### Или залить 10Гб данных и протестировать скорость запросов в сравнении с 1 инстансом PostgreSQL
> ### Описать что и как делали и с какими проблемами столкнулись
***

