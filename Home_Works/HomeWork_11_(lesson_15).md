<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "PostgreSQL и Google Kubernetes Engine" </h2></div>

***
> ### Домашнее задание
> ### Работа c PostgreSQL в Kubernetes

> ### Цель:
> ### запустить HA и multi master PostgreSQL кластер в Kubernetes

> ### Описание/Пошаговая инструкция выполнения домашнего задания:
> ### Развернуть CitusDB в GKE (ЯО или аналоги), залить 10 Гб чикагского такси. Шардировать. Оценить производительность. Описать проблемы, с которыми столкнулись




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
    id: e9b7u63em26t6bunev8p
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-30T15:04:48Z"
    name: yc-auto-subnet-0
    network_id: enpdrc6d9pcuoed3inbk
    zone_id: ru-central1-a
    v4_cidr_blocks:
      - 192.168.0.0/24
    
    Creating subnet yc-auto-subnet-1
    id: e2lhl3c2fmcufr0hth9v
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-30T15:04:49Z"
    name: yc-auto-subnet-1
    network_id: enpdrc6d9pcuoed3inbk
    zone_id: ru-central1-b
    v4_cidr_blocks:
      - 192.168.1.0/24
    
    Creating subnet yc-auto-subnet-2
    id: b0c7kd92k2n9hko1t7ts
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-30T15:04:50Z"
    name: yc-auto-subnet-2
    network_id: enpdrc6d9pcuoed3inbk
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
    id: aje4fcq3sh1crv24coqb
    folder_id: b1g59qc1dbgj9fu1qp9t
    created_at: "2023-07-30T15:06:44.452804344Z"
    name: k8s-sa-b1g59qc1dbgj9fu1qp9t

    user@comp-beelink ~ $ 
    user@comp-beelink ~ $ SA_ID=$(yc iam service-account get --name k8s-sa-${FOLDER_ID} --format json | jq .id -r)
    user@comp-beelink ~ $ yc resource-manager folder add-access-binding --id $FOLDER_ID --role admin --subject serviceAccount:$SA_ID
    done (3s)
    effective_deltas:
      - action: ADD
        access_binding:
          role_id: admin
          subject:
            id: aje4fcq3sh1crv24coqb
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
     done (7m25s)
     id: catk45uvebrfarkprv13
     folder_id: b1g59qc1dbgj9fu1qp9t
     created_at: "2023-07-30T15:09:02Z"
     name: k8s-cluster
     status: RUNNING
     health: HEALTHY
     network_id: enpdrc6d9pcuoed3inbk
     master:
       zonal_master:
         zone_id: ru-central1-a
         internal_v4_address: 192.168.0.27
         external_v4_address: 158.160.117.178
       version: "1.24"
       endpoints:
         internal_v4_endpoint: https://192.168.0.27
         external_v4_endpoint: https://158.160.117.178
       master_auth:
         cluster_ca_certificate: |
           -----BEGIN CERTIFICATE-----
           MIIC5zCCAc+gAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
           cm5ldGVzMB4XDTIzMDczMDE1MDkwNVoXDTMzMDcyNzE1MDkwNVowFTETMBEGA1UE
           AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJ9I
           dnIWD3hqjynFf6miYTBoZjD+ccHxCMaOCnUbzSP4sgYCsjBM52kyjFbeOyC6Z0G2
           yl+ZYD7v1GMBeprWIBooxjZksQD+JFfsE505NNruQb7ApaR1i2jvyW3nZMHY19Wt
           T7vFVcgLfU5gDCMVFEEGC0IXdwt2tyxAfTzTVvfjfg71AWm7w6iYzLapH3Wp9LF1
           aL1XahNP7pSQS81RmdvRtVFKb99TMLcvmx5diilfVQSDYO7JEWON55jqVfSeeGak
           rQ/I/L6RLEB2eFWq3kaplYVqRq3KHr011SRMaeconor9R43KN4pm1rqnbr+gwzv4
           yNuupoIomj8FA6IRelsCAwEAAaNCMEAwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
           /wQFMAMBAf8wHQYDVR0OBBYEFM6hO6ORvazHrQBLL743XVTlEgUaMA0GCSqGSIb3
           DQEBCwUAA4IBAQB4Ne/zLS01KZHr707Ge79Hl95FcWlLBYf+08QDIOceQw/+GDsu
           koTWN//ovZ5nZRdmN9pqngZV0HCsNZvZWp4ZI8zrxjh/5sV0TrVCq2PrsN/1RkKG
           q2IePGpvoHXH1ANDEPUB6WFH6VktTvmJmKw71aGQFCm7iHD/c2asECJkdXyAiw1O
           pp1unGcq9/al2aRoQAQmoU0+EZyb+FLfyWgdaXcOfU8onWspODW6kogGp2zk05Rb
           QCbQyaZDerK1Cl7+NuQErXKgIsvoJDIwQ75+UAaREhTMynEUb0ZOPuBXnb5l5ona
           Q9718IMVxnwcQnSKa7oMZyvNFc+LkV5x+FTa
           -----END CERTIFICATE-----
       version_info:
         current_version: "1.24"
       maintenance_policy:
         auto_upgrade: true
         maintenance_window:
           anytime: {}
     ip_allocation_policy:
       cluster_ipv4_cidr_block: 10.112.0.0/16
       node_ipv4_cidr_mask_size: "24"
       service_ipv4_cidr_block: 10.96.0.0/16
     service_account_id: aje4fcq3sh1crv24coqb
     node_service_account_id: aje4fcq3sh1crv24coqb
     release_channel: REGULAR
    
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
     done (2m14s)
     id: catiim1a2b7u3ntc0ccg
     cluster_id: catk45uvebrfarkprv13
     created_at: "2023-07-30T15:18:23Z"
     name: k8s-cluster-ng
     status: RUNNING
     node_template:
       platform_id: standard-v2
       resources_spec:
         memory: "4294967296"
         cores: "2"
         core_fraction: "50"
       boot_disk_spec:
         disk_type_id: network-ssd
         disk_size: "103079215104"
       v4_address_spec: {}
       scheduling_policy: {}
       network_interface_specs:
         - subnet_ids:
             - e2lhl3c2fmcufr0hth9v
           primary_v4_address_spec: {}
       network_settings: {}
       container_runtime_settings: {}
       container_network_settings: {}
     scale_policy:
       fixed_scale:
         size: "2"
     allocation_policy:
       locations:
         - zone_id: ru-central1-b
           subnet_id: e2lhl3c2fmcufr0hth9v
     deploy_policy:
       max_expansion: "3"
     instance_group_id: cl1uqmtaebo1f1an5r7b
     node_version: "1.24"
     version_info:
       current_version: "1.24"
     maintenance_policy:
       auto_upgrade: true
       auto_repair: true
       maintenance_window:
         anytime: {}
    
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
      NAME                        STATUS   ROLES    AGE     VERSION
      cl1uqmtaebo1f1an5r7b-ecuf   Ready    <none>   2m30s   v1.24.8
      cl1uqmtaebo1f1an5r7b-izoz   Ready    <none>   2m16s   v1.24.8
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
      | catiim1a2b7u3ntc0ccg | k8s-cluster-ng | cl1uqmtaebo1f1an5r7b | 2023-07-30 15:18:23 | RUNNING |    2 |
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
      | epdphr3ob22b675tp83a           | cl1uqmtaebo1f1an5r7b-ecuf | 2 50% core(s), 4.0 GB of       | 96.0 GB ssd | READY  |
      | RUNNING_ACTUAL                 |                           | memory                         |             |        |
      | epd7r3nfbeauf83pi1q0           | cl1uqmtaebo1f1an5r7b-izoz | 2 50% core(s), 4.0 GB of       | 96.0 GB ssd | READY  |
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
      service/kubernetes   ClusterIP   10.96.128.1   <none>        443/TCP   10m
      user@comp-beelink ~ $ 
      ```         
    

  * ?Установка [helm v.2](https://helm.sh/docs/intro/install/) 
    ```bash
    curl -fsSL -o /tmp/helm-v2.17.0-linux-amd64.tar.gz https://get.helm.sh/helm-v2.17.0-linux-amd64.tar.gz
    tar -zxvf /tmp/helm-v2.17.0-linux-amd64.tar.gz -C /tmp
    sudo mv /tmp/linux-amd64/helm /usr/local/bin/helm
    sudo mv /tmp/linux-amd64/tiller /usr/local/bin/tiller
    ```



  * ?Установка `tiller`
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

***

  * Yandex Cloud не умеет брать образы с [DockerHub](https://hub.docker.com/) и чтобы работать с образами внутри K8s от YC надо использовать `Container Registry`
    * [Как начать работать с Container Registry](https://cloud.yandex.ru/docs/container-registry/quickstart/)
    * [Пошаговые инструкции для Container Registry](https://cloud.yandex.ru/docs/container-registry/operations/)
    * [Example from YC](https://github.com/nar3k/yc-public-tasks/blob/master/k8s/README.md)
      * Создаем реестр в `Container Registry`
        ```bash
        yc container registry create --name my-first-registry
        ```
        ```console
        user@comp-beelink ~ $ yc container registry create --name my-first-registry
        done (1s)
        id: crpe7qnn5mr8hn92376k
        folder_id: b1g59qc1dbgj9fu1qp9t
        name: my-first-registry
        status: ACTIVE
        created_at: "2023-07-30T15:39:10.114Z"
        
        user@comp-beelink ~ $ 
        ```
        
      * Аутентифицирумся в `Container Registry` с помощью [Docker Credential helper](https://cloud.yandex.ru/docs/container-registry/operations/authentication#cred-helper)
      * Сконфигурируем `Docker` для использования `docker-credential-yc`
        ```bash
        yc container registry configure-docker
        ```
        ```console
        user@comp-beelink ~ $ yc container registry configure-docker
        docker configured to use yc --profile "default" for authenticating "cr.yandex" container registries
        Credential helper is configured in '/home/user/.docker/config.json'
        user@comp-beelink ~ $ 
        ```

      * Качаем Docker-образ из репозитория [DockerHub](https://hub.docker.com/). 
        ```bash
        docker pull postgres:14
        ```
        ```console
        user@comp-beelink ~ $ docker pull postgres:14
        14: Pulling from library/postgres
        648e0aadf75a: Already exists 
        f715c8c55756: Already exists 
        b11a1dc32c8c: Already exists 
        f29e8ba9d17c: Already exists 
        78af88a8afb0: Already exists 
        b74279c188d9: Already exists 
        6e3e5bf64fd2: Already exists 
        b62a2c2d2ce5: Already exists 
        765629a1b92d: Already exists 
        365d9a245882: Already exists 
        aeb308034f5e: Already exists 
        ddb205754449: Already exists 
        d403994c1833: Already exists 
        Digest: sha256:eecfb2ab484dadaae790866c9e5090a67deeee76417f072786569bea03f53e3f
        Status: Downloaded newer image for postgres:14
        docker.io/library/postgres:14
        user@comp-beelink ~ $ 
        ```       

      * Присваиваем скачанному Docker-образу тег вида cr.yandex/<ID реестра>/<имя Docker-образа>:<тег>
        ```bash
        REGISTRY_ID=$(yc container registry get --name my-first-registry  --format json | jq .id -r)
        docker tag postgres:14 cr.yandex/$REGISTRY_ID/postgres:14        
        ```
        ```console
        user@comp-beelink ~ $ 
        user@comp-beelink ~ $ docker tag postgres:14 cr.yandex/$REGISTRY_ID/postgres:14  
        user@comp-beelink ~ $ 
        ```

      * Загружаем Docker-образ в репозиторий `Container Registry`
        ```bash
        docker push cr.yandex/$REGISTRY_ID/postgres:14
        ```
        ```console
        user@comp-beelink ~ $ docker push  cr.yandex/$REGISTRY_ID/postgres:14        
        The push refers to repository [cr.yandex/crpe7qnn5mr8hn92376k/postgres]
        5c3930591e04: Pushed 
        c07097663ecd: Pushed 
        e69910c04ba4: Pushed 
        4a3d39be60d2: Pushed 
        22d1c656babb: Pushed 
        84186af28e30: Pushed 
        9f1d5955e197: Pushed 
        e49f832d248f: Pushed 
        adab9cb10294: Pushed 
        b9dd30974b55: Pushed 
        aabef479524d: Pushed 
        a014511bc8c2: Pushed 
        c6e34807c2d5: Pushed 
        14: digest: sha256:55247e19106a3998b54d8f06d0c5070285a7f47b20f34a56c23ed9a75573d0e5 size: 3040
        user@comp-beelink ~ $ 
        ```

      * Посмотреть имеющиеся образы 
        ```bash
        yc container image list
        ```
        ```console
        user@comp-beelink ~ $ yc container image list
        +----------------------+---------------------+-------------------------------+------+-----------------+
        |          ID          |       CREATED       |             NAME              | TAGS | COMPRESSED SIZE |
        +----------------------+---------------------+-------------------------------+------+-----------------+
        | crpk7strd7g40mj8tjau | 2023-07-30 16:15:37 | crpe7qnn5mr8hn92376k/postgres |   14 | 141.1 MB        |
        +----------------------+---------------------+-------------------------------+------+-----------------+
        
        user@comp-beelink ~ $ 
        ```
        
***

      * texxt
        ```bash
        
        ```
        ```console

        ```






    https://github.com/bakdata/citus-k8s-membership-manager
