<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Углубленный анализ производительности. Профилирование. Мониторинг. Оптимизация" </h2></div>


***

> ### Развернуть Постгрес на ВМ
  * Развернул `PostgreSQL 15` под ОС `Ubuntu 22.04`,  на своей ВМ в `Proxmox`
    ```console
    ubuntu@srv-postgres:~$ cat /etc/os-release 
    PRETTY_NAME="Ubuntu 22.04.2 LTS"
    NAME="Ubuntu"
    VERSION_ID="22.04"
    VERSION="22.04.2 LTS (Jammy Jellyfish)"
    VERSION_CODENAME=jammy
    ID=ubuntu
    ID_LIKE=debian
    HOME_URL="https://www.ubuntu.com/"
    SUPPORT_URL="https://help.ubuntu.com/"
    BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
    PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
    UBUNTU_CODENAME=jammy
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo cat /proc/version
    Linux version 5.15.0-71-generic (buildd@lcy02-amd64-044) (gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, GNU ld (GNU Binutils for Ubuntu) 2.38) #78-Ubuntu SMP Tue Apr 18 09:00:29 UTC 2023
    ubuntu@srv-postgres:~$ 
    ubuntu@srv-postgres:~$ sudo pg_lsclusters 
    Ver Cluster Port Status Owner    Data directory              Log file
    15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
    ubuntu@srv-postgres:~$ 
    ```

***

> ### Протестировать pg_bench
* Трям

***

> ### Выставить оптимальные настройки
* Трям

***

> ### Проверить насколько выросла производительность
* Трям

***

> ### Настроить кластер на оптимальную производительность не обращая внимания на стабильность БД
* Трям

***






