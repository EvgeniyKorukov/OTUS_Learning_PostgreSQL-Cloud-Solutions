<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Работа с горизонтально масштабируемым кластером" </h2></div>

***

> ### Развернуть CockroachDB в GKE или GCE
  * Создаем 4 ВМ в YandexCloud (postgresx1 + CocroachDBx3)
      * Общие параметры для всех 4х ВМ с фиксированным внутренним IPv4.   
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
        | **`pg-srv1`** | `10.129.0.20` | Сервер с СУБД PostgreSQL 15 |
        | **`pg-cdb1`** | `10.129.0.21` | Сервер с CocroachDB |      
        | **`pg-cdb2`** | `10.129.0.22` | Сервер с CocroachDB |  
        | **`pg-cdb3`** | `10.129.0.23` | Сервер с CocroachDB |          

     * Создание ВМ `pg-srv`
       ```console
       yc compute instance create \
         --name pg-srv \
         --hostname pg-srv \
         --cores 2 \
         --memory 4 \
         --create-boot-disk size=30G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.20 \
         --zone ru-central1-b \
         --core-fraction 100 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       ```
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

***      
> ### Потесировать dataset с чикагскими такси
> ### Или залить 10Гб данных и протестировать скорость запросов в сравнении с 1 инстансом PostgreSQL
> ### Описать что и как делали и с какими проблемами столкнулись
***

