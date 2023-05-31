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
  * Создаем 3 ВМ для PostgreSQL 15+Patroni в YandexCloud
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
  * Создаем 2 ВМ для HA Proxy в YandexCloud
    * Общие параметры для всех 2х ВМ для HA Proxy с фиксированным внутренним IPv4. Команды по созданию ВМ прилагать не буду т.к. они типовые
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
***

> ### 3. Проверяем отказоустойсивость
***

> ### 4. *настраиваем бэкапы через wal-g или pg_probackup
***
