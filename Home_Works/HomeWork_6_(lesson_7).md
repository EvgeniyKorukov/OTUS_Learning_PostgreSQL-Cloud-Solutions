<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Кластер Patroni on-premise" </h2></div>

***

> ### 1. Создаем 3 ВМ для etcd + 3 ВМ для Patroni +1 HA proxy (при проблемах можно на 1 хосте развернуть)
  * Создаем 3 ВМ для etcd в YandexCloud
    * Общие параметры для всех 3х ВМ для etcd с фиксированным внутренним IPv4. Команды по созданию ВМ прилагать не буду т.к. они типовые
         :hammer_and_wrench: Параметр | :memo: Значение |
        --------------:|---------------| 
        | Операционная система | `Ubuntu 20.04 LTS` |
        | Зона доступности | `ru-central1-b` |
        | Платформа | `Intel Cascade Lake	` |
        | vCPU | `2` |
        | Гарантированная доля vCPU | `20%` |
        | RAM | `2 ГБ` |
        | Тип диска | `SSD` | 
        | Объём дискового пространства | `5 ГБ` |
        | Макс. IOPS (чтение / запись) | `1000 / 1000` |
        | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
        | Прерываемая | :ballot_box_with_check: |
    * Индивидуальные параметры каждой ВМ для etcd
        :hammer_and_wrench: Название ВМ | :memo: Внутренний IPv4 |
        --------------:|---------------|
        | **`etcd1`** | `10.129.0.11` |
        | **`etcd2`** | `10.129.0.12` |      
        | **`etcd3`** | `10.129.0.13` |
     * На каждой ВМ устанавливаем etcd и останавливаем etcd, чтобы изменить конфигурацию
       ```bash
       sudo apt update && sudo apt upgrade -y && sudo apt install -y etcd
       sudo systemctl stop etcd
       ```
     * На каждой ВМ с etcd:
       * Формируем конфигурацию для etcd
       * Делаем сервис etcd автозапускаемым
       * Запускаем etcd
       * **❗️используем FQDM, полное имя хоста (сервера) получаем командой `hostname -f`**
         ```bash
         cat > temp.cfg << EOF 
         ETCD_NAME="$(hostname)"
         ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
         ETCD_ADVERTISE_CLIENT_URLS="http://$(hostname -f):2379"
         ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
         ETCD_INITIAL_ADVERTISE_PEER_URLS="http://$(hostname -f):2380"
         ETCD_INITIAL_CLUSTER_TOKEN="PatroniCluster"
         ETCD_INITIAL_CLUSTER="etcd1=http://etcd1.ru-central1.internal:2380,etcd2=http://etcd2.ru-central1.internal:2380,etcd3=http://etcd3.ru-central1.internal:2380"
         ETCD_INITIAL_CLUSTER_STATE="new"
         ETCD_DATA_DIR="/var/lib/etcd"
         EOF
         cat temp.cfg | sudo tee -a /etc/default/etcd

         sudo systemctl enable etcd

         sudo systemctl start etcd
         ```
     * Проверяем статус etcd на одной из ВМ
       ```console
       ubuntu@etcd1:~$ etcdctl cluster-health
       member 61f991406239b07 is healthy: got healthy result from http://etcd1.ru-central1.internal:2379
       member 3376d9633be63daf is healthy: got healthy result from http://etcd3.ru-central1.internal:2379
       member 64b82de471857189 is healthy: got healthy result from http://etcd2.ru-central1.internal:2379
       cluster is healthy
       ubuntu@etcd1:~$ 
       ```
  * Создаем 3 ВМ для PostgreSQL 15+Patroni+PGBouncer в YandexCloud
    * Общие параметры для всех 3х ВМ для postgres+patroni с фиксированным внутренним IPv4. Команды по созданию ВМ прилагать не буду т.к. они типовые
         :hammer_and_wrench: Параметр | :memo: Значение |
        --------------:|---------------| 
        | Операционная система | `Ubuntu 20.04 LTS` |
        | Зона доступности | `ru-central1-b` |
        | Платформа | `Intel Cascade Lake	` |
        | vCPU | `2` |
        | Гарантированная доля vCPU | `20%` |
        | RAM | `4 ГБ` |
        | Тип диска | `SSD` | 
        | Объём дискового пространства | `10 ГБ` |
        | Макс. IOPS (чтение / запись) | `1000 / 1000` |
        | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
        | Прерываемая | :ballot_box_with_check: |
    * Индивидуальные параметры каждой ВМ для etcd
        :hammer_and_wrench: Название ВМ | :memo: Внутренний IPv4 |
        --------------:|---------------|
        | **`pg-srv1`** | `10.129.0.21` |
        | **`pg-srv2`** | `10.129.0.22` |      
        | **`pg-srv3`** | `10.129.0.23` |
     * На каждой ВМ устанавливаем PostgreSQL 15 и проверяем, что экземпляры запустились (вывод установки приводить не стал т.к. в этом нет особого смысла)
         ```bash
         sudo apt update && sudo apt upgrade -y -q && echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee -a /etc/apt/sources.list.d/pgdg.list && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15

         pg_lsclusters
         ```
       * Вывод pg_lsclusters с каждой ВМ:
         ```console
         ubuntu@pg-srv1:~$ hostname; pg_lsclusters
         pg-srv1
         Ver Cluster Port Status Owner    Data directory              Log file
         15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
         ubuntu@pg-srv1:~$        
         ```
         ```console
         ubuntu@pg-srv2:~$ hostname; pg_lsclusters
         pg-srv2
         Ver Cluster Port Status Owner    Data directory              Log file
         15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
         ubuntu@pg-srv2:~$ 
         ```
         ```console
         ubuntu@pg-srv3:~$ pg_lsclusters
         Ver Cluster Port Status Owner    Data directory              Log file
         15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
         ubuntu@pg-srv3:~$ 
         ```  
  * Проверяем доступность по сети ВМ c Postgres и etcd
    ```console
    ubuntu@pg-srv1:~$ 
    ubuntu@pg-srv1:~$ ping etcd3 -c 2
    PING etcd3.ru-central1.internal (10.129.0.13) 56(84) bytes of data.
    64 bytes from etcd3.ru-central1.internal (10.129.0.13): icmp_seq=1 ttl=61 time=0.357 ms
    64 bytes from etcd3.ru-central1.internal (10.129.0.13): icmp_seq=2 ttl=61 time=0.395 ms

    --- etcd3.ru-central1.internal ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1006ms
    rtt min/avg/max/mdev = 0.357/0.376/0.395/0.019 ms
    ubuntu@pg-srv1:~$ 
    ubuntu@pg-srv1:~$ ping etcd2.ru-central1.internal -c 2
    PING etcd2.ru-central1.internal (10.129.0.12) 56(84) bytes of data.
    64 bytes from etcd2.ru-central1.internal (10.129.0.12): icmp_seq=1 ttl=61 time=1.21 ms
    64 bytes from etcd2.ru-central1.internal (10.129.0.12): icmp_seq=2 ttl=61 time=0.358 ms

    --- etcd2.ru-central1.internal ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1002ms
    rtt min/avg/max/mdev = 0.358/0.783/1.209/0.425 ms
    ubuntu@pg-srv1:~$ 
    ```
    ```console
    ubuntu@etcd2:~$ ping pg-srv3 -c2
    PING pg-srv3.ru-central1.internal (10.129.0.23) 56(84) bytes of data.
    64 bytes from pg-srv3.ru-central1.internal (10.129.0.23): icmp_seq=1 ttl=61 time=2.00 ms
    64 bytes from pg-srv3.ru-central1.internal (10.129.0.23): icmp_seq=2 ttl=61 time=0.403 ms

    --- pg-srv3.ru-central1.internal ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1002ms
    rtt min/avg/max/mdev = 0.403/1.199/1.996/0.796 ms
    ubuntu@etcd2:~$ 
    ubuntu@etcd2:~$ ping pg-srv1.ru-central1.internal -c2
    PING pg-srv1.ru-central1.internal (10.129.0.21) 56(84) bytes of data.
    64 bytes from pg-srv1.ru-central1.internal (10.129.0.21): icmp_seq=1 ttl=61 time=0.351 ms
    64 bytes from pg-srv1.ru-central1.internal (10.129.0.21): icmp_seq=2 ttl=61 time=0.441 ms

    --- pg-srv1.ru-central1.internal ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1002ms
    rtt min/avg/max/mdev = 0.351/0.396/0.441/0.045 ms
    ubuntu@etcd2:~$ 
    ```    
  * Создаем 1 ВМ для HA Proxy в YandexCloud
    * Общие параметры для 1 ВМ для HA Proxy с фиксированным внутренним IPv4. Команды по созданию ВМ прилагать не буду т.к. они типовые
         :hammer_and_wrench: Параметр | :memo: Значение |
        --------------:|---------------| 
        | Операционная система | `Ubuntu 20.04 LTS` |
        | Зона доступности | `ru-central1-b` |
        | Платформа | `Intel Cascade Lake	` |
        | vCPU | `2` |
        | Гарантированная доля vCPU | `20%` |
        | RAM | `2 ГБ` |
        | Тип диска | `SSD` | 
        | Объём дискового пространства | `5 ГБ` |
        | Макс. IOPS (чтение / запись) | `1000 / 1000` |
        | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
        | Прерываемая | :ballot_box_with_check: |
    * Индивидуальные параметры каждой ВМ для etcd
        :hammer_and_wrench: Название ВМ | :memo: Внутренний IPv4 |
        --------------:|---------------|
        | **`haproxy1`** | `10.129.0.31` |
        | **`haproxy2`** | `10.129.0.32` |      
    
***

> ### 2. Инициализируем кластер
  * Устанавливаем и настраиваем Patroni на одной из нод, в моем случае это `pg-srv1`:
    * Ставим Python
       ```console
       ubuntu@pg-srv1:~$ sudo apt-get install -y python3 python3-pip git mc
       ubuntu@pg-srv1:~$ sudo pip3 install psycopg2-binary     
       ```
    * Останавливаем и удаляем экземлпяр PostgreSQL который запускается по-умолчанию
       ```console
       ubuntu@pg-srv1:~$ 
       ubuntu@pg-srv1:~$ sudo pg_dropcluster 15 main --stop
       ubuntu@pg-srv1:~$ 
       ```
    * Проверяем, что остановили PostgreSQL
       ```console
       ubuntu@pg-srv1:~$ pg_lsclusters 
       Ver Cluster Port Status Owner Data directory Log file
       ubuntu@pg-srv1:~$ 
       ```
    * Устанавливаем Patroni
       ```console
       ubuntu@pg-srv1:~$ sudo pip3 install patroni[etcd]
       ``` 
    * Создаем symlink
       ```bash
       ubuntu@pg-srv1:~$ sudo ln -s /usr/local/bin/patroni /bin/patroni
       ubuntu@pg-srv1:~$ 
       ```
    * Создаем сервис в ОС для запуска Patroni
       ```console
       ubuntu@pg-srv1:~$ cat > temp.cfg << EOF 
       [Unit]
       Description=High availability PostgreSQL Cluster
       After=syslog.target network.target
       [Service]
       Type=simple
       User=postgres
       Group=postgres
       ExecStart=/usr/local/bin/patroni /etc/patroni.yml
       KillMode=process
       TimeoutSec=30
       Restart=no
       [Install]
       WantedBy=multi-user.target
       EOF
       cat temp.cfg | sudo tee -a /etc/systemd/system/patroni.service
       ```
       ```console
       ubuntu@pg-srv1:~$ cat temp2.cfg
       [Unit]
       Description=High availability PostgreSQL Cluster
       After=syslog.target network.target
       [Service]
       Type=simple
       User=postgres
       Group=postgres
       ExecStart=/usr/local/bin/patroni /etc/patroni.yml
       KillMode=process
       TimeoutSec=30
       Restart=no
       [Install]
       WantedBy=multi-user.target
       ubuntu@pg-srv1:~$ 
       ```
    * Делаем сервис Patroni автозапусакаемым       
       ```console
       ubuntu@pg-srv1:~$ sudo systemctl enable patroni
       ```
    * Создаем конфиг для Patroni
       ```console
       cat > temp2.cfg << EOF 
       scope: patroni_cluster
       name: $(hostname)
       restapi:
         listen: $(hostname -I | tr -d " "):8008
         connect_address: $(hostname -I | tr -d " "):8008
       etcd:
         hosts: etcd1.ru-central1.internal:2379,etcd2.ru-central1.internal:2379,etcd3.ru-central1.internal:2379
       bootstrap:
         dcs:
           ttl: 30
           loop_wait: 10
           retry_timeout: 10
           maximum_lag_on_failover: 1048576
           postgresql:
             use_pg_rewind: true
             parameters:
         initdb: 
         - encoding: UTF8
         - data-checksums
         pg_hba: 
         - host replication replicator 10.129.0.0/24 md5
         - host all all 10.129.0.0/24 md5
         users:
           admin:
             password: admin_321
             options:
               - createrole
               - createdb
       postgresql:
         listen: 127.0.0.1, $(hostname -I | tr -d " "):5432
         connect_address: $(hostname -I | tr -d " "):5432
         data_dir: /var/lib/postgresql/15/main
         bin_dir: /usr/lib/postgresql/15/bin
         pgpass: /tmp/pgpass0
         authentication:
           replication:
             username: replicator
             password: rep-pass_321
           superuser:
             username: postgres
             password: zalando_321
           rewind:  
             username: rewind_user
             password: rewind_password_321
         parameters:
           unix_socket_directories: '.'
       tags:
           nofailover: false
           noloadbalance: false
           clonefrom: false
           nosync: false
       EOF
       cat temp2.cfg | sudo tee -a /etc/patroni.yml
       ```
       ```console
       ubuntu@pg-srv1:~$ cat temp2.cfg
       scope: patroni_cluster
       name: pg-srv1
       restapi:
         listen: 10.129.0.21:8008
         connect_address: 10.129.0.21:8008
       etcd:
         hosts: etcd1.ru-central1.internal:2379,etcd2.ru-central1.internal:2379,etcd3.ru-central1.internal:2379
       bootstrap:
         dcs:
           ttl: 30
           loop_wait: 10
           retry_timeout: 10
           maximum_lag_on_failover: 1048576
           postgresql:
             use_pg_rewind: true
             parameters:
         initdb: 
         - encoding: UTF8
         - data-checksums
         pg_hba: 
         - host replication replicator 10.129.0.0/24 md5
         - host all all 10.129.0.0/24 md5
         users:
           admin:
             password: admin_321
             options:
               - createrole
               - createdb
       postgresql:
         listen: 127.0.0.1, 10.129.0.21:5432
         connect_address: 10.129.0.21:5432
         data_dir: /var/lib/postgresql/15/main
         bin_dir: /usr/lib/postgresql/15/bin
         pgpass: /tmp/pgpass0
         authentication:
           replication:
             username: replicator
             password: rep-pass_321
           superuser:
             username: postgres
             password: zalando_321
           rewind:  
             username: rewind_user
             password: rewind_password_321
         parameters:
           unix_socket_directories: '.'
       tags:
           nofailover: false
           noloadbalance: false
           clonefrom: false
           nosync: false
       ubuntu@pg-srv1:~$ 

       ```
    * Запускаем службу с Patroni
       ```console
       ubuntu@pg-srv1:~$ sudo systemctl start patroni
       ubuntu@pg-srv1:~$ sudo systemctl status patroni
       ● patroni.service - High availability PostgreSQL Cluster
            Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: enabled)
            Active: active (running) since Wed 2023-05-31 20:35:19 UTC; 4s ago
          Main PID: 19882 (patroni)
             Tasks: 5 (limit: 4646)
            Memory: 22.7M
            CGroup: /system.slice/patroni.service
                    └─19882 /usr/bin/python3 /usr/local/bin/patroni /etc/patroni.yml

       May 31 20:35:19 pg-srv1 systemd[1]: Started High availability PostgreSQL Cluster.
       May 31 20:35:20 pg-srv1 patroni[19882]: 2023-05-31 20:35:20,037 INFO: Selected new etcd server http://etcd2.ru-cent>
       May 31 20:35:20 pg-srv1 patroni[19882]: 2023-05-31 20:35:20,045 INFO: No PostgreSQL configuration items changed, no>
       May 31 20:35:20 pg-srv1 patroni[19882]: 2023-05-31 20:35:20,056 INFO: Lock owner: None; I am pg-srv1
       May 31 20:35:20 pg-srv1 patroni[19882]: 2023-05-31 20:35:20,061 INFO: waiting for leader to bootstrap
       ubuntu@pg-srv1:~$ 
       ```
    * Проверяем список запущенных нод. 
       ```console
       ubuntu@pg-srv1:~$ sudo patronictl -c /etc/patroni.yml list
       + Cluster: patroni_cluster ------+---------+----+-----------+
       | Member  | Host        | Role   | State   | TL | Lag in MB |
       +---------+-------------+--------+---------+----+-----------+
       | pg-srv1 | 10.129.0.21 | Leader | running |  1 |           |
       +---------+-------------+--------+---------+----+-----------+
       ubuntu@pg-srv1:~$ 
       ```
  * Устанавливаем и настраиваем Patroni на остальных двух нодах, в моем случае это `pg-srv2` и `pg-srv2`:
    * Ставим Python
    * Останавливаем и удаляем экземлпяр PostgreSQL который запускается по-умолчанию
    * Устанавливаем Patroni
    * Создаем symlink
       ```console
       sudo apt install -y python3 python3-pip git mc && sudo pip3 install psycopg2-binary && sudo systemctl stop postgresql@15-main && sudo -u postgres pg_dropcluster 15 main && sudo pip3 install       patroni[etcd] && sudo ln -s /usr/local/bin/patroni /bin/patroni   
       ```
    * Создаем сервис в ОС для запуска Patroni 
    * **❗️Здесь привожу только скрипт для генерации т.к. на всех хостах он одинаковый**
       ```console
       cat > temp.cfg << EOF 
       [Unit]
       Description=High availability PostgreSQL Cluster
       After=syslog.target network.target
       [Service]
       Type=simple
       User=postgres
       Group=postgres
       ExecStart=/usr/local/bin/patroni /etc/patroni.yml
       KillMode=process
       TimeoutSec=30
       Restart=no
       [Install]
       WantedBy=multi-user.target
       EOF
       cat temp.cfg | sudo tee -a /etc/systemd/system/patroni.service
       ```
    * Делаем файл конфигурации для Patroni
    * **❗️Здесь привожу только скрипт для генерации т.к. разница будет в имени хоста и его ip-адресе** 
       ```console
       cat > temp2.cfg << EOF 
       scope: patroni_cluster
       name: $(hostname)
       restapi:
         listen: $(hostname -I | tr -d " "):8008
         connect_address: $(hostname -I | tr -d " "):8008
       etcd:
         hosts: etcd1.ru-central1.internal:2379,etcd2.ru-central1.internal:2379,etcd3.ru-central1.internal:2379
       bootstrap:
         dcs:
           ttl: 30
           loop_wait: 10
           retry_timeout: 10
           maximum_lag_on_failover: 1048576
           postgresql:
             use_pg_rewind: true
             parameters:
         initdb: 
         - encoding: UTF8
         - data-checksums
         pg_hba: 
         - host replication replicator 10.129.0.0/24 md5
         - host all all 10.129.0.0/24 md5
         users:
           admin:
             password: admin_321
             options:
               - createrole
               - createdb
       postgresql:
         listen: 127.0.0.1, $(hostname -I | tr -d " "):5432
         connect_address: $(hostname -I | tr -d " "):5432
         data_dir: /var/lib/postgresql/15/main
         bin_dir: /usr/lib/postgresql/15/bin
         pgpass: /tmp/pgpass0
         authentication:
           replication:
             username: replicator
             password: rep-pass_321
           superuser:
             username: postgres
             password: zalando_321
           rewind:  
             username: rewind_user
             password: rewind_password_321
         parameters:
           unix_socket_directories: '.'
       tags:
           nofailover: false
           noloadbalance: false
           clonefrom: false
           nosync: false
       EOF
       cat temp2.cfg | sudo tee -a /etc/patroni.yml
       ```
    * Делаем службу Patroni автозапускаемой и запускаем ее
       ```console
       sudo systemctl enable patroni && sudo systemctl start patroni 
       ```
    * Проверяем список запущенных нод. 
       ```console
       ubuntu@pg-srv1:~$ sudo patronictl -c /etc/patroni.yml list
       + Cluster: patroni_cluster -------+---------+----+-----------+
       | Member  | Host        | Role    | State   | TL | Lag in MB |
       +---------+-------------+---------+---------+----+-----------+
       | pg-srv1 | 10.129.0.21 | Leader  | running |  1 |           |
       | pg-srv2 | 10.129.0.22 | Replica | running |  1 |         0 |
       | pg-srv3 | 10.129.0.23 | Replica | running |  1 |         0 |
       +---------+-------------+---------+---------+----+-----------+
       ubuntu@pg-srv1:~$ 
       ```   
  * Устанавливаем и настраиваем PGBouncer на всех 3х ВМ с Postgres и Patroni:
    * Установка PGBouncer
      ```console
      sudo apt install -y pgbouncer
      ```
    * Создаем файл-конфигурации `pgbouncer.ini`
    * Создаем файл с пользователями и паролями `userlist.txt`
    * Перезапуск сервиса PGBouncer
    * Формирование файла pgpass для без парольного входа
    * 
       
***

> ### 3. Проверяем отказоустойсивость
***

> ### 4. *настраиваем бэкапы через wal-g или pg_probackup
***
