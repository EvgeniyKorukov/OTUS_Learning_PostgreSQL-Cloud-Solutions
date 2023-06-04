<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Введение в Kubernetes" </h2></div>

***

> ### Устанавливаем minikube
  * Создаем ВМ в YandexCloud
    ```console
    yc compute instance create \
      --name k8s-srv \
      --hostname k8s-srv \
      --cores 2 \
      --memory 4 \
      --create-boot-disk size=15G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
      --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.30 \
      --zone ru-central1-b \
      --core-fraction 20 \
      --preemptible \
      --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
    ```
    :hammer_and_wrench: Параметр | :memo: Значение |
    --------------:|---------------| 
    | Имя ВМ | **`k8s-srv`** |
    | Внешний ip | `158.160.31.143` |
    | Внктренний ip | `10.129.0.30` |        
    | Операционная система | `Ubuntu 20.04 LTS` |
    | Зона доступности | `ru-central1-b` |
    | Платформа | `Intel Cascade Lake	` |
    | vCPU | `2` |
    | Гарантированная доля vCPU | `20%` |
    | RAM | `4 ГБ` |
    | Тип диска | `SSD` | 
    | Объём дискового пространства | `15 ГБ` |
    | Макс. IOPS (чтение / запись) | `1000 / 1000` |
    | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
    | Прерываемая | :ballot_box_with_check: |
    * Подключаемся к ВМ
    ```console
    ssh ubuntu@158.160.31.143
    ```

***

> ### Разворачиваем PostgreSQL 14 через манифест
  * Text
    ```console
    ```
***

> ### Topic1
  * Text
    ```console
    ```
***

> ### Topic1
  * Text
    ```console
    ```
***

