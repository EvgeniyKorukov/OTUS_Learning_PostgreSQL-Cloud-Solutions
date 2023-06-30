<div align="center"><h2> Отчет о выполнении домашнего задания по теме: "Работа с кластером высокой доступности" </h2></div>

***

> ### Вариант 2 Introducing pg_auto_failover: Open source extension for automated failover and high-availability in PostgreSQL
***
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
        | **`pg-srv1`** | `10.129.0.21` | Сервер с СУБД PostgreSQL 15 |
        | **`pg-srv2`** | `10.129.0.22` | Сервер с СУБД PostgreSQL 15 |      
        | **`pg-mon`** | `10.129.0.23` | Сервер `monitor` для [pg_auto_failover](https://github.com/hapostgres/pg_auto_failover) |
     * Создание ВМ `pg-srv1`
       ```console
       yc compute instance create \
         --name pg-srv1 \
         --hostname pg-srv1 \
         --cores 2 \
         --memory 2 \
         --create-boot-disk size=5G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.21 \
         --zone ru-central1-b \
         --core-fraction 20 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       ```
     * Создание ВМ `pg-srv2`
       ```console
       yc compute instance create \
         --name pg-srv2 \
         --hostname pg-srv2 \
         --cores 2 \
         --memory 2 \
         --create-boot-disk size=5G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.22 \
         --zone ru-central1-b \
         --core-fraction 20 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       ```
     * Создание ВМ `pg-mon`
       ```console
       yc compute instance create \
         --name pg-mon \
         --hostname pg-mon \
         --cores 2 \
         --memory 2 \
         --create-boot-disk size=5G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2004-lts \
         --network-interface subnet-name=default-ru-central1-b,nat-ip-version=ipv4,ipv4-address=10.129.0.23 \
         --zone ru-central1-b \
         --core-fraction 20 \
         --preemptible \
         --metadata-from-file ssh-keys=/home/eugink/.ssh/eugin_yandex_key.pub
       ```
***

  * Настройка ВМ `pg-mon`
       * Устанавливаем PostgreSQL 15
         <pre><details><summary>Вывод терминала</summary>
         ubuntu@pg-mon:~$ sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update
         Hit:1 http://mirror.yandex.ru/ubuntu focal InRelease
         Get:2 http://mirror.yandex.ru/ubuntu focal-updates InRelease [114 kB]
         Get:3 http://mirror.yandex.ru/ubuntu focal-backports InRelease [108 kB]
         Get:4 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 Packages [2,678 kB]
         Get:5 http://mirror.yandex.ru/ubuntu focal-updates/main i386 Packages [848 kB]
         Get:6 http://mirror.yandex.ru/ubuntu focal-updates/main Translation-en [447 kB]
         Get:7 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 c-n-f Metadata [16.9 kB]
         Get:8 http://mirror.yandex.ru/ubuntu focal-updates/restricted amd64 Packages [2,045 kB]
         Get:9 http://mirror.yandex.ru/ubuntu focal-updates/restricted Translation-en [286 kB]
         Get:10 http://mirror.yandex.ru/ubuntu focal-updates/universe amd64 Packages [1,079 kB]
         Get:11 http://mirror.yandex.ru/ubuntu focal-updates/universe i386 Packages [734 kB]
         Get:12 http://mirror.yandex.ru/ubuntu focal-updates/universe Translation-en [257 kB]
         Get:13 http://security.ubuntu.com/ubuntu focal-security InRelease [114 kB]                    
         Get:14 http://security.ubuntu.com/ubuntu focal-security/main amd64 Packages [2,296 kB]
         Get:15 http://security.ubuntu.com/ubuntu focal-security/main i386 Packages [618 kB]           
         Get:16 http://security.ubuntu.com/ubuntu focal-security/main Translation-en [365 kB]          
         Get:17 http://security.ubuntu.com/ubuntu focal-security/main amd64 c-n-f Metadata [13.0 kB]   
         Get:18 http://security.ubuntu.com/ubuntu focal-security/restricted amd64 Packages [1,937 kB]  
         Get:19 http://security.ubuntu.com/ubuntu focal-security/restricted Translation-en [271 kB]    
         Get:20 http://security.ubuntu.com/ubuntu focal-security/universe amd64 Packages [851 kB]      
         Get:21 http://security.ubuntu.com/ubuntu focal-security/universe i386 Packages [603 kB]       
         Get:22 http://security.ubuntu.com/ubuntu focal-security/universe Translation-en [176 kB]      
         Fetched 15.9 MB in 32s (495 kB/s)                                                             
         Reading package lists... Done
         Building dependency tree       
         Reading state information... Done
         5 packages can be upgraded. Run 'apt list --upgradable' to see them.
         Reading package lists... Done
         Building dependency tree       
         Reading state information... Done
         Calculating upgrade... Done
         The following NEW packages will be installed:
           linux-headers-5.4.0-153 linux-headers-5.4.0-153-generic linux-image-5.4.0-153-generic
           linux-modules-5.4.0-153-generic linux-modules-extra-5.4.0-153-generic
         The following packages will be upgraded:
           accountsservice libaccountsservice0 linux-generic linux-headers-generic linux-image-generic
         5 upgraded, 5 newly installed, 0 to remove and 0 not upgraded.
         5 standard LTS security updates
         Need to get 77.3 MB of archives.
         After this operation, 380 MB of additional disk space will be used.
         Get:1 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 accountsservice amd64 0.6.55-0ubuntu12~20.04.6 [61.4 kB]
         Get:2 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 libaccountsservice0 amd64 0.6.55-0ubuntu12~20.04.6 [72.7 kB]
         Get:3 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-modules-5.4.0-153-generic amd64 5.4.0-153.170 [15.0 MB]
         Get:4 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-image-5.4.0-153-generic amd64 5.4.0-153.170 [10.5 MB]
         Get:5 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-modules-extra-5.4.0-153-generic amd64 5.4.0-153.170 [39.2 MB]
         Get:6 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-generic amd64 5.4.0.153.150 [1,904 B]          
         Get:7 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-image-generic amd64 5.4.0.153.150 [2,604 B]    
         Get:8 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-headers-5.4.0-153 all 5.4.0-153.170 [11.0 MB]  
         Get:9 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-headers-5.4.0-153-generic amd64 5.4.0-153.170 [1,363 kB]
         Get:10 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 linux-headers-generic amd64 5.4.0.153.150 [2,464 B] 
         Fetched 77.3 MB in 21s (3,629 kB/s)                                                                                
         (Reading database ... 102594 files and directories currently installed.)
         Preparing to unpack .../0-accountsservice_0.6.55-0ubuntu12~20.04.6_amd64.deb ...
         Unpacking accountsservice (0.6.55-0ubuntu12~20.04.6) over (0.6.55-0ubuntu12~20.04.5) ...
         Preparing to unpack .../1-libaccountsservice0_0.6.55-0ubuntu12~20.04.6_amd64.deb ...
         Unpacking libaccountsservice0:amd64 (0.6.55-0ubuntu12~20.04.6) over (0.6.55-0ubuntu12~20.04.5) ...
         Selecting previously unselected package linux-modules-5.4.0-153-generic.
         Preparing to unpack .../2-linux-modules-5.4.0-153-generic_5.4.0-153.170_amd64.deb ...
         Unpacking linux-modules-5.4.0-153-generic (5.4.0-153.170) ...
         Selecting previously unselected package linux-image-5.4.0-153-generic.
         Preparing to unpack .../3-linux-image-5.4.0-153-generic_5.4.0-153.170_amd64.deb ...
         Unpacking linux-image-5.4.0-153-generic (5.4.0-153.170) ...
         Selecting previously unselected package linux-modules-extra-5.4.0-153-generic.
         Preparing to unpack .../4-linux-modules-extra-5.4.0-153-generic_5.4.0-153.170_amd64.deb ...
         Unpacking linux-modules-extra-5.4.0-153-generic (5.4.0-153.170) ...
         Preparing to unpack .../5-linux-generic_5.4.0.153.150_amd64.deb ...
         Unpacking linux-generic (5.4.0.153.150) over (5.4.0.152.149) ...
         Preparing to unpack .../6-linux-image-generic_5.4.0.153.150_amd64.deb ...
         Unpacking linux-image-generic (5.4.0.153.150) over (5.4.0.152.149) ...
         Selecting previously unselected package linux-headers-5.4.0-153.
         Preparing to unpack .../7-linux-headers-5.4.0-153_5.4.0-153.170_all.deb ...
         Unpacking linux-headers-5.4.0-153 (5.4.0-153.170) ...
         Selecting previously unselected package linux-headers-5.4.0-153-generic.
         Preparing to unpack .../8-linux-headers-5.4.0-153-generic_5.4.0-153.170_amd64.deb ...
         Unpacking linux-headers-5.4.0-153-generic (5.4.0-153.170) ...
         Preparing to unpack .../9-linux-headers-generic_5.4.0.153.150_amd64.deb ...
         Unpacking linux-headers-generic (5.4.0.153.150) over (5.4.0.152.149) ...
         Setting up linux-headers-5.4.0-153 (5.4.0-153.170) ...
         Setting up linux-modules-5.4.0-153-generic (5.4.0-153.170) ...
         Setting up linux-image-5.4.0-153-generic (5.4.0-153.170) ...
         I: /boot/vmlinuz.old is now a symlink to vmlinuz-5.4.0-152-generic
         I: /boot/initrd.img.old is now a symlink to initrd.img-5.4.0-152-generic
         I: /boot/vmlinuz is now a symlink to vmlinuz-5.4.0-153-generic
         I: /boot/initrd.img is now a symlink to initrd.img-5.4.0-153-generic
         Setting up linux-headers-5.4.0-153-generic (5.4.0-153.170) ...
         Setting up libaccountsservice0:amd64 (0.6.55-0ubuntu12~20.04.6) ...
         Setting up accountsservice (0.6.55-0ubuntu12~20.04.6) ...
         Setting up linux-headers-generic (5.4.0.153.150) ...
         Setting up linux-modules-extra-5.4.0-153-generic (5.4.0-153.170) ...
         Setting up linux-image-generic (5.4.0.153.150) ...
         Setting up linux-generic (5.4.0.153.150) ...
         Processing triggers for dbus (1.12.16-2ubuntu2.3) ...
         Processing triggers for libc-bin (2.31-0ubuntu9.9) ...
         Processing triggers for linux-image-5.4.0-153-generic (5.4.0-153.170) ...
         /etc/kernel/postinst.d/initramfs-tools:
         update-initramfs: Generating /boot/initrd.img-5.4.0-153-generic
         /etc/kernel/postinst.d/zz-update-grub:
         Sourcing file `/etc/default/grub'
         Sourcing file `/etc/default/grub.d/init-select.cfg'
         Generating grub configuration file ...
         Found linux image: /boot/vmlinuz-5.4.0-153-generic
         Found initrd image: /boot/initrd.img-5.4.0-153-generic
         Found linux image: /boot/vmlinuz-5.4.0-152-generic
         Found initrd image: /boot/initrd.img-5.4.0-152-generic
         Found linux image: /boot/vmlinuz-5.4.0-42-generic
         Found initrd image: /boot/initrd.img-5.4.0-42-generic
         done
         OK
         Hit:1 http://mirror.yandex.ru/ubuntu focal InRelease
         Hit:2 http://mirror.yandex.ru/ubuntu focal-updates InRelease                              
         Hit:3 http://mirror.yandex.ru/ubuntu focal-backports InRelease                            
         Get:4 http://apt.postgresql.org/pub/repos/apt focal-pgdg InRelease [117 kB]                              
         Get:5 http://security.ubuntu.com/ubuntu focal-security InRelease [114 kB]                                
         Get:6 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 Packages [265 kB]
         Fetched 496 kB in 1s (632 kB/s)   
         Reading package lists... Done
         N: Skipping acquire of configured file 'main/binary-i386/Packages' as repository 'http://apt.postgresql.org/pub/repos/apt focal-pgdg InRelease' doesn't support architecture 'i386'
         ubuntu@pg-mon:~$ sudo apt-get -y install postgresql-15
         Reading package lists... Done
         Building dependency tree       
         Reading state information... Done
         The following packages were automatically installed and are no longer required:
           linux-headers-5.4.0-42 linux-headers-5.4.0-42-generic linux-image-5.4.0-42-generic
           linux-modules-5.4.0-42-generic linux-modules-extra-5.4.0-42-generic
         Use 'sudo apt autoremove' to remove them.
         The following additional packages will be installed:
           libcommon-sense-perl libgdbm-compat4 libjson-perl libjson-xs-perl libllvm10 libperl5.30 libpq5 libsensors-config
           libsensors5 libtypes-serialiser-perl libxslt1.1 perl perl-modules-5.30 postgresql-client-15
           postgresql-client-common postgresql-common ssl-cert sysstat
         Suggested packages:
           lm-sensors perl-doc libterm-readline-gnu-perl | libterm-readline-perl-perl make libb-debug-perl
           liblocale-codes-perl postgresql-doc-15 openssl-blacklist isag
         The following NEW packages will be installed:
           libcommon-sense-perl libgdbm-compat4 libjson-perl libjson-xs-perl libllvm10 libperl5.30 libpq5 libsensors-config
           libsensors5 libtypes-serialiser-perl libxslt1.1 perl perl-modules-5.30 postgresql-15 postgresql-client-15
           postgresql-client-common postgresql-common ssl-cert sysstat
         0 upgraded, 19 newly installed, 0 to remove and 0 not upgraded.
         Need to get 41.6 MB of archives.
         After this operation, 186 MB of additional disk space will be used.
         Get:1 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 perl-modules-5.30 all 5.30.0-9ubuntu0.4 [2,739 kB]
         Get:2 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 postgresql-client-common all 250.pgdg20.04+1 [93.3 kB]
         Get:3 http://mirror.yandex.ru/ubuntu focal/main amd64 libgdbm-compat4 amd64 1.18.1-5 [6,244 B] 
         Get:4 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 libperl5.30 amd64 5.30.0-9ubuntu0.4 [3,959 kB]
         Get:5 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 perl amd64 5.30.0-9ubuntu0.4 [224 kB]
         Get:6 http://mirror.yandex.ru/ubuntu focal/main amd64 libjson-perl all 4.02000-2 [80.9 kB]
         Get:7 http://mirror.yandex.ru/ubuntu focal/main amd64 ssl-cert all 1.0.39 [17.0 kB]      
         Get:8 http://mirror.yandex.ru/ubuntu focal/main amd64 libcommon-sense-perl amd64 3.74-2build6 [20.1 kB]
         Get:9 http://mirror.yandex.ru/ubuntu focal/main amd64 libtypes-serialiser-perl all 1.0-1 [12.1 kB]
         Get:10 http://mirror.yandex.ru/ubuntu focal/main amd64 libjson-xs-perl amd64 4.020-1build1 [83.7 kB] 
         Get:11 http://mirror.yandex.ru/ubuntu focal/main amd64 libllvm10 amd64 1:10.0.0-4ubuntu1 [15.3 MB]
         Get:12 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 postgresql-common all 250.pgdg20.04+1 [239 kB]
         Get:13 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 libpq5 amd64 15.3-1.pgdg20.04+1 [184 kB]
         Get:14 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 postgresql-client-15 amd64 15.3-1.pgdg20.04+1 [1,680 kB]
         Get:15 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 postgresql-15 amd64 15.3-1.pgdg20.04+1 [16.3 MB]
         Get:16 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 libsensors-config all 1:3.6.0-2ubuntu1.1 [6,052 B]
         Get:17 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 libsensors5 amd64 1:3.6.0-2ubuntu1.1 [27.2 kB]
         Get:18 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 libxslt1.1 amd64 1.1.34-4ubuntu0.20.04.1 [151 kB]
         Get:19 http://mirror.yandex.ru/ubuntu focal-updates/main amd64 sysstat amd64 12.2.0-2ubuntu0.3 [448 kB]
         Fetched 41.6 MB in 2s (18.5 MB/s) 
         Preconfiguring packages ...
         Selecting previously unselected package perl-modules-5.30.
         (Reading database ... 139024 files and directories currently installed.)
         Preparing to unpack .../00-perl-modules-5.30_5.30.0-9ubuntu0.4_all.deb ...
         Unpacking perl-modules-5.30 (5.30.0-9ubuntu0.4) ...
         Selecting previously unselected package libgdbm-compat4:amd64.
         Preparing to unpack .../01-libgdbm-compat4_1.18.1-5_amd64.deb ...
         Unpacking libgdbm-compat4:amd64 (1.18.1-5) ...
         Selecting previously unselected package libperl5.30:amd64.
         Preparing to unpack .../02-libperl5.30_5.30.0-9ubuntu0.4_amd64.deb ...
         Unpacking libperl5.30:amd64 (5.30.0-9ubuntu0.4) ...
         Selecting previously unselected package perl.
         Preparing to unpack .../03-perl_5.30.0-9ubuntu0.4_amd64.deb ...
         Unpacking perl (5.30.0-9ubuntu0.4) ...
         Selecting previously unselected package libjson-perl.
         Preparing to unpack .../04-libjson-perl_4.02000-2_all.deb ...
         Unpacking libjson-perl (4.02000-2) ...
         Selecting previously unselected package postgresql-client-common.
         Preparing to unpack .../05-postgresql-client-common_250.pgdg20.04+1_all.deb ...
         Unpacking postgresql-client-common (250.pgdg20.04+1) ...
         Selecting previously unselected package ssl-cert.
         Preparing to unpack .../06-ssl-cert_1.0.39_all.deb ...
         Unpacking ssl-cert (1.0.39) ...
         Selecting previously unselected package postgresql-common.
         Preparing to unpack .../07-postgresql-common_250.pgdg20.04+1_all.deb ...
         Adding 'diversion of /usr/bin/pg_config to /usr/bin/pg_config.libpq-dev by postgresql-common'
         Unpacking postgresql-common (250.pgdg20.04+1) ...
         Selecting previously unselected package libcommon-sense-perl.
         Preparing to unpack .../08-libcommon-sense-perl_3.74-2build6_amd64.deb ...
         Unpacking libcommon-sense-perl (3.74-2build6) ...
         Selecting previously unselected package libtypes-serialiser-perl.
         Preparing to unpack .../09-libtypes-serialiser-perl_1.0-1_all.deb ...
         Unpacking libtypes-serialiser-perl (1.0-1) ...
         Selecting previously unselected package libjson-xs-perl.
         Preparing to unpack .../10-libjson-xs-perl_4.020-1build1_amd64.deb ...
         Unpacking libjson-xs-perl (4.020-1build1) ...
         Selecting previously unselected package libllvm10:amd64.
         Preparing to unpack .../11-libllvm10_1%3a10.0.0-4ubuntu1_amd64.deb ...
         Unpacking libllvm10:amd64 (1:10.0.0-4ubuntu1) ...
         Selecting previously unselected package libpq5:amd64.
         Preparing to unpack .../12-libpq5_15.3-1.pgdg20.04+1_amd64.deb ...
         Unpacking libpq5:amd64 (15.3-1.pgdg20.04+1) ...
         Selecting previously unselected package libsensors-config.
         Preparing to unpack .../13-libsensors-config_1%3a3.6.0-2ubuntu1.1_all.deb ...
         Unpacking libsensors-config (1:3.6.0-2ubuntu1.1) ...
         Selecting previously unselected package libsensors5:amd64.
         Preparing to unpack .../14-libsensors5_1%3a3.6.0-2ubuntu1.1_amd64.deb ...
         Unpacking libsensors5:amd64 (1:3.6.0-2ubuntu1.1) ...
         Selecting previously unselected package libxslt1.1:amd64.
         Preparing to unpack .../15-libxslt1.1_1.1.34-4ubuntu0.20.04.1_amd64.deb ...
         Unpacking libxslt1.1:amd64 (1.1.34-4ubuntu0.20.04.1) ...
         Selecting previously unselected package postgresql-client-15.
         Preparing to unpack .../16-postgresql-client-15_15.3-1.pgdg20.04+1_amd64.deb ...
         Unpacking postgresql-client-15 (15.3-1.pgdg20.04+1) ...
         Selecting previously unselected package postgresql-15.
         Preparing to unpack .../17-postgresql-15_15.3-1.pgdg20.04+1_amd64.deb ...
         Unpacking postgresql-15 (15.3-1.pgdg20.04+1) ...
         Selecting previously unselected package sysstat.
         Preparing to unpack .../18-sysstat_12.2.0-2ubuntu0.3_amd64.deb ...
         Unpacking sysstat (12.2.0-2ubuntu0.3) ...
         Setting up perl-modules-5.30 (5.30.0-9ubuntu0.4) ...
         Setting up libsensors-config (1:3.6.0-2ubuntu1.1) ...
         Setting up libpq5:amd64 (15.3-1.pgdg20.04+1) ...
         Setting up libllvm10:amd64 (1:10.0.0-4ubuntu1) ...
         Setting up ssl-cert (1.0.39) ...
         Setting up libgdbm-compat4:amd64 (1.18.1-5) ...
         Setting up libsensors5:amd64 (1:3.6.0-2ubuntu1.1) ...
         Setting up libxslt1.1:amd64 (1.1.34-4ubuntu0.20.04.1) ...
         Setting up libperl5.30:amd64 (5.30.0-9ubuntu0.4) ...
         Setting up sysstat (12.2.0-2ubuntu0.3) ...
         
         Creating config file /etc/default/sysstat with new version
         update-alternatives: using /usr/bin/sar.sysstat to provide /usr/bin/sar (sar) in auto mode
         Created symlink /etc/systemd/system/multi-user.target.wants/sysstat.service → /lib/systemd/system/sysstat.service.
         Setting up perl (5.30.0-9ubuntu0.4) ...
         Setting up libjson-perl (4.02000-2) ...
         Setting up postgresql-client-common (250.pgdg20.04+1) ...
         Setting up libcommon-sense-perl (3.74-2build6) ...
         Setting up postgresql-client-15 (15.3-1.pgdg20.04+1) ...
         update-alternatives: using /usr/share/postgresql/15/man/man1/psql.1.gz to provide /usr/share/man/man1/psql.1.gz (psql.1.gz) in auto mode
         Setting up postgresql-common (250.pgdg20.04+1) ...
         Adding user postgres to group ssl-cert
         
         Creating config file /etc/postgresql-common/createcluster.conf with new version
         Building PostgreSQL dictionaries from installed myspell/hunspell packages...
         Removing obsolete dictionary files:
         '/etc/apt/trusted.gpg.d/apt.postgresql.org.gpg' -> '/usr/share/postgresql-common/pgdg/apt.postgresql.org.gpg'
         Created symlink /etc/systemd/system/multi-user.target.wants/postgresql.service → /lib/systemd/system/postgresql.service.
         Setting up libtypes-serialiser-perl (1.0-1) ...
         Setting up postgresql-15 (15.3-1.pgdg20.04+1) ...
         Creating new PostgreSQL cluster 15/main ...
         /usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions
         The files belonging to this database system will be owned by user "postgres".
         This user must also own the server process.
         
         The database cluster will be initialized with locale "en_US.UTF-8".
         The default database encoding has accordingly been set to "UTF8".
         The default text search configuration will be set to "english".
         
         Data page checksums are disabled.
         
         fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
         creating subdirectories ... ok
         selecting dynamic shared memory implementation ... posix
         selecting default max_connections ... 100
         selecting default shared_buffers ... 128MB
         selecting default time zone ... Etc/UTC
         creating configuration files ... ok
         running bootstrap script ... ok
         performing post-bootstrap initialization ... ok
         syncing data to disk ... ok
         Setting up libjson-xs-perl (4.020-1build1) ...
         Processing triggers for systemd (245.4-4ubuntu3.22) ...
         Processing triggers for man-db (2.9.1-1) ...
         Processing triggers for libc-bin (2.31-0ubuntu9.9) ...
         </details></pre> 
       
       * Удаляем кластер, который создавался автоматически
         ```console
         ubuntu@pg-mon:~$ sudo pg_dropcluster 15 main --stop
         ubuntu@pg-mon:~$ pg_lsclusters 
         Ver Cluster Port Status Owner Data directory Log file
         ubuntu@pg-mon:~$ 
         ```

       * Устанавливаем `pg_auto_failover`
           ```console
           ubuntu@pg-mon:~$ sudo apt-get install postgresql-15-auto-failover -y
           Reading package lists... Done
           Building dependency tree       
           Reading state information... Done
           The following packages were automatically installed and are no longer required:
             linux-headers-5.4.0-42 linux-headers-5.4.0-42-generic linux-image-5.4.0-42-generic
             linux-modules-5.4.0-42-generic linux-modules-extra-5.4.0-42-generic
           Use 'sudo apt autoremove' to remove them.
           The following additional packages will be installed:
             pg-auto-failover-cli
           The following NEW packages will be installed:
             pg-auto-failover-cli postgresql-15-auto-failover
           0 upgraded, 2 newly installed, 0 to remove and 0 not upgraded.
           Need to get 822 kB of archives.
           After this operation, 2,182 kB of additional disk space will be used.
           Get:1 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 pg-auto-failover-cli amd64 2.0-2.pgdg20.04+1 [430 kB]
           Get:2 http://apt.postgresql.org/pub/repos/apt focal-pgdg/main amd64 postgresql-15-auto-failover amd64 2.0-2.pgdg20.04+1 [392 kB]
           Fetched 822 kB in 0s (2,437 kB/s)                     
           Selecting previously unselected package pg-auto-failover-cli.
           (Reading database ... 143270 files and directories currently installed.)
           Preparing to unpack .../pg-auto-failover-cli_2.0-2.pgdg20.04+1_amd64.deb ...
           Unpacking pg-auto-failover-cli (2.0-2.pgdg20.04+1) ...
           Selecting previously unselected package postgresql-15-auto-failover.
           Preparing to unpack .../postgresql-15-auto-failover_2.0-2.pgdg20.04+1_amd64.deb ...
           Unpacking postgresql-15-auto-failover (2.0-2.pgdg20.04+1) ...
           Setting up pg-auto-failover-cli (2.0-2.pgdg20.04+1) ...
           Setting up postgresql-15-auto-failover (2.0-2.pgdg20.04+1) ...
           Processing triggers for postgresql-common (250.pgdg20.04+1) ...
           Building PostgreSQL dictionaries from installed myspell/hunspell packages...
           Removing obsolete dictionary files:
           Processing triggers for man-db (2.9.1-1) ...
           ubuntu@pg-mon:~$ 
           ```
       * Правим /etc/hosts
         * `10.129.0.21 pg-srv1.ru-central1.internal pg-srv1`
         * `10.129.0.22 pg-srv2.ru-central1.internal pg-srv2`
         * `127.0.1.1. pg-mon.ru-central1.internal pg-mon` -> `10.129.0.23 pg-mon.ru-central1.internal pg-mon`


             
       * Создаем рабочий каталог+прописываем их в `~/.profile`+применяем их
           ```console
           ubuntu@pg-mon:~$ 
           ubuntu@pg-mon:~$ sudo mkdir /pg_mon && sudo chown -R postgres:postgres /pg_mon
           ubuntu@pg-mon:~$ 
           ubuntu@pg-mon:~$ sudo su - postgres
           postgres@pg-mon:~$ 
           postgres@pg-mon:~$ echo "export PATH=/usr/lib/postgresql/15/bin/:$PATH" >> .profile
           postgres@pg-mon:~$ echo "export PGDATA=/pg_mon" >> .profile
           postgres@pg-mon:~$ cat .profile 
           export PATH=/usr/lib/postgresql/15/bin/:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
           export PGDATA=/pg_mon
           postgres@pg-mon:~$ 
           postgres@pg-mon:~$ . ~/.profile 
           postgres@pg-mon:~$ 
           ```
       * Инициализируем `pg-auto-failover`
           ```console
           postgres@pg-mon:~$ pg_autoctl create monitor --auth trust --ssl-mode require --ssl-self-signed --pgport 6000 --hostname `hostname -s` --pgdata /pg_mon --pgctl /usr/lib/postgresql/15/bin/pg_ctl 
           12:06:41 13946 INFO  Using --ssl-self-signed: pg_autoctl will create self-signed certificates, allowing for encrypted network traffic
           12:06:41 13946 WARN  Self-signed certificates provide protection against eavesdropping; this setup does NOT protect against Man-In-The-Middle attacks nor Impersonation attacks.
           12:06:41 13946 WARN  See https://www.postgresql.org/docs/current/libpq-ssl.html for details
           12:06:41 13946 INFO  Initialising a PostgreSQL cluster at "/pg_mon"
           12:06:41 13946 INFO  /usr/lib/postgresql/15/bin/pg_ctl initdb -s -D /pg_mon --option '--auth=trust'
           12:06:45 13946 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /pg_mon/server.crt -keyout /pg_mon/server.key -subj "/CN=pg-mon"
           12:06:45 13946 INFO  Started pg_autoctl postgres service with pid 13968
           12:06:45 13968 INFO   /usr/bin/pg_autoctl do service postgres --pgdata /pg_mon -v
           12:06:45 13946 INFO  Started pg_autoctl monitor-init service with pid 13969
           12:06:45 13974 INFO   /usr/lib/postgresql/15/bin/postgres -D /pg_mon -p 6000 -h *
           12:06:45 13968 INFO  Postgres is now serving PGDATA "/pg_mon" on port 6000 with pid 13974
           12:06:45 13969 WARN  NOTICE:  installing required extension "btree_gist"
           12:06:45 13969 INFO  Granting connection privileges on 10.129.0.0/24
           12:06:45 13969 WARN  Skipping HBA edits (per --skip-pg-hba) for rule: hostssl "pg_auto_failover" "autoctl_node" 10.129.0.0/24 trust
           12:06:45 13969 INFO  Your pg_auto_failover monitor instance is now ready on port 6000.
           12:06:45 13969 INFO  Monitor has been successfully initialized.
           12:06:45 13946 WARN  pg_autoctl service monitor-init exited with exit status 0
           12:06:45 13968 INFO  Postgres controller service received signal SIGTERM, terminating
           12:06:45 13968 INFO  Stopping pg_autoctl postgres service
           12:06:45 13968 INFO  /usr/lib/postgresql/15/bin/pg_ctl --pgdata /pg_mon --wait stop --mode fast
           12:06:46 13946 INFO  Waiting for subprocesses to terminate.
           12:06:46 13946 INFO  Stop pg_autoctl
           postgres@pg-mon:~$ 
           ```
       * Создаем и запускаем сервис
           ```console
           root@pg-mon:~# pg_autoctl show systemd --pgdata /pg_mon
           22:09:25 14317 INFO  HINT: to complete a systemd integration, run the following commands (as root):
           22:09:25 14317 INFO  pg_autoctl -q show systemd --pgdata "/pg_mon" | tee /etc/systemd/system/pgautofailover.service
           22:09:25 14317 INFO  systemctl daemon-reload
           22:09:25 14317 INFO  systemctl enable pgautofailover
           22:09:25 14317 INFO  systemctl start pgautofailover
           [Unit]
           Description = pg_auto_failover
           
           [Service]
           WorkingDirectory = /var/lib/postgresql
           Environment = 'PGDATA=/pg_mon'
           User = postgres
           ExecStart = /usr/bin/pg_autoctl run
           Restart = always
           StartLimitBurst = 0
           ExecReload = /usr/bin/pg_autoctl reload
           
           [Install]
           WantedBy = multi-user.target
           root@pg-mon:~#
           root@pg-mon:~# vim /etc/systemd/system/pgautofailover.service
           root@pg-mon:~# 
           root@pg-mon:~# systemctl daemon-reload
           root@pg-mon:~# systemctl enable pgautofailover
           Created symlink /etc/systemd/system/multi-user.target.wants/pgautofailover.service → /etc/systemd/system/pgautofailover.service.
           root@pg-mon:~# systemctl start pgautofailover
           ```
           
       * Создаем пользователя для управления
           ```console
           postgres@pg-mon:~$ psql -p 6000 -c "alter user autoctl_node password 'zxcasdqwe'"
           ALTER ROLE
           postgres@pg-mon:~$ 
           ```

       * ?Добавляем доступ для `10.129.0.0/24` в `pg_hba.conf`
         * host    all             all             10.129.0.0/24            trust 
           ```console
           postgres@pg-mon:~$ vim /pg_mon/pg_hba.conf 
           postgres@pg-mon:~$ 
           postgres@pg-mon:~$ 
           postgres@pg-mon:~$ psql -p 6000 -c "select pg_reload_conf()"
            pg_reload_conf 
           ----------------
            t
           (1 row)
           
           postgres@pg-mon:~$ 
           ```           

       * Смотрим статус кластера
           ```console
           postgres@pg-mon:~$ pg_autoctl show state
           Name |  Node |  Host:Port |  TLI: LSN |   Connection |      Reported State |      Assigned State
           -----+-------+------------+-----------+--------------+---------------------+--------------------
           
           postgres@pg-mon:~$ 
           ```
       * Получаем строку подключения к монитору
           ```console
           postgres@pg-mon:~$ pg_autoctl show uri 
                   Type |    Name | Connection String
           -------------+---------+-------------------------------
                monitor | monitor | postgres://autoctl_node@pg-mon:6000/pg_auto_failover?sslmode=require
              formation | default | 
           
           postgres@pg-mon:~$ 
           ```           
           
***

  * Настройка ВМ `pg-srv1` и `pg-srv2`
  * 
       * Устанавливаем PostgreSQL 15
         ```console
         sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
         ```
         
       * Удаляем кластер, который создавался автоматически
         ```console
         sudo pg_dropcluster 15 main --stop
         ```

       * Устанавливаем `pg_auto_failover`
         ```console
         sudo apt-get install postgresql-15-auto-failover -y
         ```
       * Правим /etc/hosts
         * `127.0.1.1 pg-srv1.ru-central1.internal pg-srv1` > `10.129.0.21 pg-srv1.ru-central1.internal pg-srv1`
         * `127.0.1.1 pg-srv2.ru-central1.internal pg-srv2` > `10.129.0.22 pg-srv2.ru-central1.internal pg-srv2`
         * `10.129.0.23 pg-mon.ru-central1.internal pg-mon`

             
       * Создаем рабочий каталог+прописываем их в `~/.profile`+применяем их
           ```console
           sudo mkdir /pg_data && sudo chown -R postgres:postgres /pg_data
           sudo su - postgres
           echo "export PATH=/usr/lib/postgresql/15/bin/:$PATH" >> .profile
           echo "export PGDATA=/pg_data" >> .profile
           . ~/.profile 
           ```
       * Инициализируем `pg-auto-failover`
           ```console
           postgres@pg-mon:~$ pg_autoctl create monitor --auth trust --ssl-mode require --ssl-self-signed --pgport 6000 --hostname `hostname --fqdn`
           22:23:01 14522 INFO  Using --ssl-self-signed: pg_autoctl will create self-signed certificates, allowing for encrypted network traffic
           22:23:01 14522 WARN  Self-signed certificates provide protection against eavesdropping; this setup does NOT protect against Man-In-The-Middle attacks nor Impersonation attacks.
           22:23:01 14522 WARN  See https://www.postgresql.org/docs/current/libpq-ssl.html for details
           22:23:01 14522 INFO  Initialising a PostgreSQL cluster at "/pg_mon"
           22:23:01 14522 INFO  /usr/lib/postgresql/15/bin/pg_ctl initdb -s -D /pg_mon --option '--auth=trust'
           22:23:05 14522 INFO   /usr/bin/openssl req -new -x509 -days 365 -nodes -text -out /pg_mon/server.crt -keyout /pg_mon/server.key -subj "/CN=pg-mon.ru-central1.internal"
           22:23:05 14522 INFO  Started pg_autoctl postgres service with pid 14543
           22:23:05 14543 INFO   /usr/bin/pg_autoctl do service postgres --pgdata /pg_mon -v
           22:23:05 14522 INFO  Started pg_autoctl monitor-init service with pid 14544
           22:23:05 14549 INFO   /usr/lib/postgresql/15/bin/postgres -D /pg_mon -p 6000 -h *
           22:23:05 14543 INFO  Postgres is now serving PGDATA "/pg_mon" on port 6000 with pid 14549
           22:23:06 14544 WARN  NOTICE:  installing required extension "btree_gist"
           22:23:06 14544 INFO  Granting connection privileges on 10.129.0.0/24
           22:23:06 14544 WARN  Skipping HBA edits (per --skip-pg-hba) for rule: hostssl "pg_auto_failover" "autoctl_node" 10.129.0.0/24 trust
           22:23:06 14544 INFO  Your pg_auto_failover monitor instance is now ready on port 6000.
           22:23:06 14544 INFO  Monitor has been successfully initialized.
           22:23:06 14522 WARN  pg_autoctl service monitor-init exited with exit status 0
           22:23:06 14543 INFO  Postgres controller service received signal SIGTERM, terminating
           22:23:06 14543 INFO  Stopping pg_autoctl postgres service
           22:23:06 14543 INFO  /usr/lib/postgresql/15/bin/pg_ctl --pgdata /pg_mon --wait stop --mode fast
           22:23:06 14522 INFO  Waiting for subprocesses to terminate.
           22:23:07 14522 INFO  Stop pg_autoctl
           postgres@pg-mon:~$ 
           ```
       * Создаем и запускаем сервис
           ```console
           root@pg-mon:~# pg_autoctl show systemd --pgdata /pg_mon
           22:09:25 14317 INFO  HINT: to complete a systemd integration, run the following commands (as root):
           22:09:25 14317 INFO  pg_autoctl -q show systemd --pgdata "/pg_mon" | tee /etc/systemd/system/pgautofailover.service
           22:09:25 14317 INFO  systemctl daemon-reload
           22:09:25 14317 INFO  systemctl enable pgautofailover
           22:09:25 14317 INFO  systemctl start pgautofailover
           [Unit]
           Description = pg_auto_failover
           
           [Service]
           WorkingDirectory = /var/lib/postgresql
           Environment = 'PGDATA=/pg_mon'
           User = postgres
           ExecStart = /usr/bin/pg_autoctl run
           Restart = always
           StartLimitBurst = 0
           ExecReload = /usr/bin/pg_autoctl reload
           
           [Install]
           WantedBy = multi-user.target
           root@pg-mon:~#
           root@pg-mon:~# vim /etc/systemd/system/pgautofailover.service
           root@pg-mon:~# 
           root@pg-mon:~# systemctl daemon-reload
           root@pg-mon:~# systemctl enable pgautofailover
           Created symlink /etc/systemd/system/multi-user.target.wants/pgautofailover.service → /etc/systemd/system/pgautofailover.service.
           root@pg-mon:~# systemctl start pgautofailover
           ```
           
       * Создаем пользователя для управления
           ```console
           postgres@pg-mon:~$ psql -p 6000 -c "alter user autoctl_node password 'zxcasdqwe'"
           ALTER ROLE
           postgres@pg-mon:~$ 
           ```

       * Добавляем доступ для `10.129.0.0/24` в `pg_hba.conf`
         * host    all             all             10.129.0.0/24            trust 
           ```console
           postgres@pg-mon:~$ vim /pg_mon/pg_hba.conf 
           postgres@pg-mon:~$ 
           postgres@pg-mon:~$ 
           postgres@pg-mon:~$ psql -p 6000 -c "select pg_reload_conf()"
            pg_reload_conf 
           ----------------
            t
           (1 row)
           
           postgres@pg-mon:~$ 
           ```           

       * Смотрим статус кластера
           ```console
           postgres@pg-mon:~$ pg_autoctl show state
           Name |  Node |  Host:Port |  TLI: LSN |   Connection |      Reported State |      Assigned State
           -----+-------+------------+-----------+--------------+---------------------+--------------------
           
           postgres@pg-mon:~$ 
           ```
       * С
           ```console
           ```

       * С
           ```console
           ```                       
