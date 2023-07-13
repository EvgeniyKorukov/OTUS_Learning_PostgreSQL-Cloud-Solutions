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
    | Прерываемая | нет |
    
  * Устанавливаем PostgreSQL 15


***

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

Потесировать dataset с чикагскими такси
Или залить 10Гб данных и протестировать скорость запросов в сравнении с 1 инстансом PostgreSQL
Описать что и как делали и с какими проблемами столкнулись      
