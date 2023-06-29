<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Работа с кластером высокой доступности" </h2></div>

***

> ### Вариант 2 Introducing pg_auto_failover: Open source extension for automated failover and high-availability in PostgreSQL
  * Создаем 3 ВМ в YandexCloud (postgres*2+1 monitor)
    * Общие параметры для всех 3х ВМ с фиксированным внутренним IPv4. 
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
    * Индивидуальные параметры каждой ВМ:
        :hammer_and_wrench: Название ВМ | :memo: Внутренний IPv4 | Описание |
        --------------:|---------------|---------------|
        | **`pg_srv1`** | `10.129.0.21` | Сервер с СУБД Potgres |
        | **`pg_srv2`** | `10.129.0.22` | Сервер с СУБД Potgres |      
        | **`pg_mon`** | `10.129.0.23` | Сервер `monitor` для [pg_auto_failover](https://github.com/hapostgres/pg_auto_failover) |
     * На каждой ВМ устанавливаем ...
       ```bash
       sudo apt update && sudo apt upgrade -y 
       ```




***
> ### Вариант Для гурманов Настройка Active/Passive PostgreSQL Cluster с использованием Pacemaker, Corosync, и DRBD (CentOS 5,5)
