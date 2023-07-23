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


  * Text
    ```bash
  
    ```
    ```console
  
    ```    


    
    
