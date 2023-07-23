<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "PostgreSQL и Google Kubernetes Engine" </h2></div>

***
> ### Домашнее задание
> ### Работа c PostgreSQL в Kubernetes

> ### Цель:
> ### запустить HA и multi master PostgreSQL кластер в Kubernetes

> ### Описание/Пошаговая инструкция выполнения домашнего задания:
> ### Развернуть CitusDB в GKE (ЯО или аналоги), залить 10 Гб чикагского такси. Шардировать. Оценить производительность. Описать проблемы, с которыми столкнулись

> ### Задание повышенной сложности*
> ### залить все чикагское такси, оценить производительность


***
* Создаем K8s в YandexCloud
  * Создаем сеть
    ```bash
    yc vpc network create --name yc-auto-network
    ```
    ```console
    user@comp-beelink ~ $ yc vpc network create --name yc-auto-network
    id: enph3p1bgr4c8p9klt6d
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-23T16:42:35Z"
    name: yc-auto-network
    
    user@comp-beelink ~ $ 
    ```


  * Создаем подсети в трех зонах доступности для работы группы виртуальных машин
    ```bash
    zones=(a b c)
    
    for i in ${!zones[@]}; do
      echo "Creating subnet yc-auto-subnet-$i"
      yc vpc subnet create --name yc-auto-subnet-$i \
      --zone ru-central1-${zones[$i]} \
      --range 192.168.$i.0/24 \
      --network-name yc-auto-network
    done  
    ```
    ```console
    user@comp-beelink ~ $ zones=(a b c)
    
    for i in ${!zones[@]}; do
      echo "Creating subnet yc-auto-subnet-$i"
      yc vpc subnet create --name yc-auto-subnet-$i \
      --zone ru-central1-${zones[$i]} \
      --range 192.168.$i.0/24 \
      --network-name yc-auto-network
    done
    
    Creating subnet yc-auto-subnet-0
    id: e9brprdm5oo4orje8mue
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-23T16:47:35Z"
    name: yc-auto-subnet-0
    network_id: enph3p1bgr4c8p9klt6d
    zone_id: ru-central1-a
    v4_cidr_blocks:
      - 192.168.0.0/24
    
    Creating subnet yc-auto-subnet-1
    id: e2lqefc3hik3mfucqjlp
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-23T16:47:36Z"
    name: yc-auto-subnet-1
    network_id: enph3p1bgr4c8p9klt6d
    zone_id: ru-central1-b
    v4_cidr_blocks:
      - 192.168.1.0/24
    
    Creating subnet yc-auto-subnet-2
    id: b0cgqjdma4hqccge1o24
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-23T16:47:37Z"
    name: yc-auto-subnet-2
    network_id: enph3p1bgr4c8p9klt6d
    zone_id: ru-central1-c
    v4_cidr_blocks:
      - 192.168.2.0/24
    
    user@comp-beelink ~ $   
    ```


  * Создадим сервисный аккаунт для кластера
    ```bash
    FOLDER_ID=$(yc config get folder-id)
    yc iam service-account create --name k8s-sa-${FOLDER_ID}
    SA_ID=$(yc iam service-account get --name k8s-sa-${FOLDER_ID} --format json | jq .id -r)
    yc resource-manager folder add-access-binding --id $FOLDER_ID --role admin --subject serviceAccount:$SA_ID
    ```
    ```console
    user@comp-beelink ~ $ 
    user@comp-beelink ~ $ FOLDER_ID=$(yc config get folder-id)
    user@comp-beelink ~ $ yc iam service-account create --name k8s-sa-${FOLDER_ID}
    id: ajeqtiiktfo4o90jb34i
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-23T16:55:45.768364206Z"
    name: k8s-sa-b1g59qc1dbgj9fu1qp9t
    user@comp-beelink ~ $ 
    user@comp-beelink ~ $ SA_ID=$(yc iam service-account get --name k8s-sa-${FOLDER_ID} --format json | jq .id -r)
    user@comp-beelink ~ $ yc resource-manager folder add-access-binding --id $FOLDER_ID --role admin --subject serviceAccount:$SA_ID
    done (1s)
    effective_deltas:
      - action: ADD
        access_binding:
          role_id: admin
          subject:
            id: ajeqtiiktfo4o90jb34i
            type: serviceAccount
    
    user@comp-beelink ~ $ 
    ```


  * Создадим мастер
    ```bash
    yc managed-kubernetes cluster create \
     --name k8s-cluster --network-name yc-auto-network \
     --zone ru-central1-a  --subnet-name yc-auto-subnet-0 \
     --public-ip \
     --service-account-id ${SA_ID} --node-service-account-id ${SA_ID} --async  
    ```
    ```console
    user@comp-beelink ~ $ yc managed-kubernetes cluster create \
     --name k8s-cluster --network-name yc-auto-network \
     --zone ru-central1-a  --subnet-name yc-auto-subnet-0 \
     --public-ip \
     --service-account-id ${SA_ID} --node-service-account-id ${SA_ID} --async
    id: catt7dm4a8o6acup4a95
    description: Create cluster
    created_at: "2023-07-23T17:00:59.159478908Z"
    created_by: ajego77nngodoio2bns5
    modified_at: "2023-07-23T17:00:59.159478908Z"
    metadata:
      '@type': type.googleapis.com/yandex.cloud.k8s.v1.CreateClusterMetadata
      cluster_id: catdndm4bgt5ihmcb9o6
    
    user@comp-beelink ~ $ 
    ```


  * Создание группы узлов.
    * Дождаться создания кластера (статус Ready и состояние Healthy)
    * исключаем публичный IP
    ```bash
    yc managed-kubernetes node-group create \
     --name k8s-cluster-ng \
     --cluster-name k8s-cluster \
     --platform-id standard-v2 \
     --cores 2 \
     --memory 4 \
     --core-fraction 50 \
     --disk-type network-ssd \
     --fixed-size 2 \
     --location subnet-name=yc-auto-subnet-1,zone=ru-central1-b \
     --async
    ```
    ```console
    user@comp-beelink ~ $ yc managed-kubernetes node-group create \
     --name k8s-cluster-ng \
     --cluster-name k8s-cluster \
     --platform-id standard-v2 \
     --cores 2 \
     --memory 4 \
     --core-fraction 50 \
     --disk-type network-ssd \
     --fixed-size 2 \
     --location subnet-name=yc-auto-subnet-1,zone=ru-central1-b \
     --async
    id: catiuoaj4ouk13biogk0
    description: Create node group
    created_at: "2023-07-23T17:17:25.216233556Z"
    created_by: ajego77nngodoio2bns5
    modified_at: "2023-07-23T17:17:25.216233556Z"
    metadata:
      '@type': type.googleapis.com/yandex.cloud.k8s.v1.CreateNodeGroupMetadata
      node_group_id: catmkpcc89oa1past7mj
    
    user@comp-beelink ~ $ 
    ```    


  * Подключение к кластеру
    ```bash
    yc managed-kubernetes cluster get-credentials --external --name k8s-cluster  
    ```
    ```console
    user@comp-beelink ~ $ yc managed-kubernetes cluster get-credentials --external --name k8s-cluster --force
    
    Context 'yc-k8s-cluster' was added as default to kubeconfig '/home/user/.kube/config'.
    Check connection to cluster using 'kubectl cluster-info --kubeconfig /home/user/.kube/config'.
    
    Note, that authentication depends on 'yc' and its config profile 'default'.
    To access clusters using the Kubernetes API, please use Kubernetes Service Account.
    user@comp-beelink ~ $ 
    ```


  * Просомтр статусов
    * Статус создания групп узлов
      ```bash
      watch kubectl get nodes
      ```
      ```console
      Every 2,0s: kubectl get nodes                                                                                                                                                                                                                                                         comp-beelink: Sun Jul 23 20:34:34 2023
       
       NAME                        STATUS   ROLES    AGE   VERSION
       cl19brbmpu4t6damncuu-edir   Ready    <none>   15m   v1.24.8
       cl19brbmpu4t6damncuu-ydir   Ready    <none>   15m   v1.24.8
      ```

    * Информация о кластере
      ```bash
      yc container cluster list-node-groups k8s-cluster
      ```
      ```console
      user@comp-beelink ~ $ yc container cluster list-node-groups k8s-cluster
      +----------------------+----------------+----------------------+---------------------+---------+------+
      |          ID          |      NAME      |  INSTANCE GROUP ID   |     CREATED AT      | STATUS  | SIZE |
      +----------------------+----------------+----------------------+---------------------+---------+------+
      | catmkpcc89oa1past7mj | k8s-cluster-ng | cl19brbmpu4t6damncuu | 2023-07-23 17:17:25 | RUNNING |    2 |
      +----------------------+----------------+----------------------+---------------------+---------+------+
      user@comp-beelink ~ $ 
      ```

    * Информация о группе узлов в кластере
      ```bash
      yc container cluster list-nodes k8s-cluster
      ```
      ```console
      user@comp-beelink ~ $ yc container cluster list-nodes k8s-cluster
      +--------------------------------+---------------------------+--------------------------------+-------------+--------+
      |         CLOUD INSTANCE         |      KUBERNETES NODE      |           RESOURCES            |    DISK     | STATUS |
      +--------------------------------+---------------------------+--------------------------------+-------------+--------+
      | epdbmnkqvkb085106opg           | cl19brbmpu4t6damncuu-edir | 2 50% core(s), 4.0 GB of       | 96.0 GB ssd | READY  |
      | RUNNING_ACTUAL                 |                           | memory                         |             |        |
      | epdhs9j9gshman96a12q           | cl19brbmpu4t6damncuu-ydir | 2 50% core(s), 4.0 GB of       | 96.0 GB ssd | READY  |
      | RUNNING_ACTUAL                 |                           | memory                         |             |        |
      +--------------------------------+---------------------------+--------------------------------+-------------+--------+
      
      user@comp-beelink ~ $ 
      ```      
 
 
    * Информация о кластере
      ```bash
      kubectl get all
      ```
      ```console
      user@comp-beelink ~ $ kubectl get all
      NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
      service/kubernetes   ClusterIP   10.96.128.1   <none>        443/TCP   30m
      user@comp-beelink ~ $ 
      ```         
    

  * Установка [helm v.2](https://helm.sh/docs/intro/install/) 
    ```bash
    curl -fsSL -o /tmp/helm-v2.17.0-linux-amd64.tar.gz https://get.helm.sh/helm-v2.17.0-linux-amd64.tar.gz
    tar -zxvf /tmp/helm-v2.17.0-linux-amd64.tar.gz -C /tmp
    sudo mv /tmp/linux-amd64/helm /usr/local/bin/helm
    sudo mv /tmp/linux-amd64/tiller /usr/local/bin/tiller
    ```



  * Установка `tiller`
    ```bash
    cat  > tiller-sa.yaml <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: tiller
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: tiller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: tiller
        namespace: kube-system
    EOF
    kubectl apply -f tiller-sa.yaml
    helm init --service-account tiller  
    ```
    ```console
    user@comp-beelink ~ $ cat  > tiller-sa.yaml <<EOF
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: tiller
      namespace: kube-system
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: tiller
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: tiller
        namespace: kube-system
    EOF
    user@comp-beelink ~ $ kubectl apply -f tiller-sa.yaml
    serviceaccount/tiller unchanged
    clusterrolebinding.rbac.authorization.k8s.io/tiller unchanged
    user@comp-beelink ~ $ helm init --service-account tiller
    Creating /home/user/.helm 
    Creating /home/user/.helm/repository 
    Creating /home/user/.helm/repository/cache 
    Creating /home/user/.helm/repository/local 
    Creating /home/user/.helm/plugins 
    Creating /home/user/.helm/starters 
    Creating /home/user/.helm/cache/archive 
    Creating /home/user/.helm/repository/repositories.yaml 
    Adding stable repo with URL: https://charts.helm.sh/stable 
    Adding local repo with URL: http://127.0.0.1:8879/charts 
    $HELM_HOME has been configured at /home/user/.helm.
    
    Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
    
    Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
    To prevent this, run `helm init` with the --tiller-tls-verify flag.
    For more information on securing your installation see: https://v2.helm.sh/docs/securing_installation/
    user@comp-beelink ~ $ 
    ```    

  * Text
    ```bash
  
    ```
    ```console
  
    ```


  * Text
    ```bash
  
    ```
    ```console
  
    ```    
