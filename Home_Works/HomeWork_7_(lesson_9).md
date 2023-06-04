<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Введение в Kubernetes" </h2></div>

***

> ### Устанавливаем minikube
  * Создаем ВМ в YandexCloud
    ```console
    yc compute instance create \
      --name k8s-srv \
      --hostname k8s-srv \
      --cores 4 \
      --memory 8 \
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
    | Внешний ip | `158.160.14.6` |
    | Внктренний ip | `10.129.0.30` |        
    | Операционная система | `Ubuntu 20.04 LTS` |
    | Зона доступности | `ru-central1-b` |
    | Платформа | `Intel Cascade Lake	` |
    | vCPU | `4` |
    | Гарантированная доля vCPU | `20%` |
    | RAM | `8 ГБ` |
    | Тип диска | `SSD` | 
    | Объём дискового пространства | `15 ГБ` |
    | Макс. IOPS (чтение / запись) | `1000 / 1000` |
    | Макс. bandwidth (чтение / запись) | `15 МБ/с / 15 МБ/с` |
    | Прерываемая | :ballot_box_with_check: |
    * Подключаемся к ВМ
    ```console
    ssh ubuntu@158.160.14.6
    ```
    
  * Установка Docker т.к. будем работать через его движок
    * [Install Docker](https://docs.docker.com/engine/install/ubuntu/)
      ```console
      sudo apt-get update
      sudo apt-get install ca-certificates curl gnupg
      sudo install -m 0755 -d /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      sudo chmod a+r /etc/apt/keyrings/docker.gpg
      echo \
        "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
        "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
        sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
      sudo apt-get update
      sudo apt-get update
      sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
      ```
  * Установка kubectl   
     * [Install kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/) 
       ```console
       curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
       chmod +x ./kubectl
       sudo mv ./kubectl /usr/local/bin/kubectl
       kubectl version --client  
       ```
    
 * Установка minikube
   * [Install minikube](	https://minikube.sigs.k8s.io/docs/start/)
      ```console
      curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
      sudo install minikube-linux-amd64 /usr/local/bin/minikube
      minikube start
      minikube dashboard
      ```





#Proxy for port (external access)
	https://stackoverflow.com/questions/47173463/how-to-access-local-kubernetes-minikube-dashboard-remotely
		kubectl proxy --address='0.0.0.0' --disable-filter=true

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

