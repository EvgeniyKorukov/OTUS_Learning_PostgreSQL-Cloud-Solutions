#### *Отчет о выполнении домашнего задания:*


> создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a или ЯО/VirtualBox
* **_Создал виртуальную машину с Ubuntu 20.04 в Yandex Cloud_**

> поставьте на нее PostgreSQL 15 через sudo apt
* **_sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15_**

> проверьте что кластер запущен через sudo -u postgres pg_lsclusters
  ```console
  eugin@pg-test:~$ sudo -u postgres pg_lsclusters
  Ver Cluster Port Status Owner    Data directory              Log file
  15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
  ```

> зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
> * postgres=# create table test(c1 text);
> * postgres=# insert into test values('1');
> * \q
```console
eugin@pg-test:~$ sudo -u postgres psql
psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
eugin@pg-test:~$ 
```

> остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
```console
eugin@pg-test:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
eugin@pg-test:~$ 
```

> создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB - или аналог в другом > облаке/виртуализации
> добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
> проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее  всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
> перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
* **_Создал диск 7 Гб и подключил его к ВМ от YandexCloud_**
  * Подготовил и настроил новый диск
  ```console
  eugin@pg-test:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
  NAME   FSTYPE SIZE MOUNTPOINT LABEL
  vda             5G            
  ├─vda1          1M            
  └─vda2 ext4     5G /          
  vdb             7G            
  eugin@pg-test:~$ 
  eugin@pg-test:~$ sudo parted -l | grep Error
  Error: /dev/vdb: unrecognised disk label
  eugin@pg-test:~$ 
  eugin@pg-test:~$ sudo parted /dev/vdb mklabel gpt
  Information: You may need to update /etc/fstab.
  eugin@pg-test:~$
  eugin@pg-test:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%  
  Information: You may need to update /etc/fstab.
  eugin@pg-test:~$
  eugin@pg-test:~$ sudo lsblk
  NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
  vda    252:0    0   5G  0 disk 
  ├─vda1 252:1    0   1M  0 part 
  └─vda2 252:2    0   5G  0 part /
  vdb    252:16   0   7G  0 disk 
  └─vdb1 252:17   0   7G  0 part 
  eugin@pg-test:~$ 
  eugin@pg-test:~$ sudo mkfs.ext4 -L datapartition /dev/vdb1
  mke2fs 1.45.5 (07-Jan-2020)
  Creating filesystem with 1834496 4k blocks and 458752 inodes
  Filesystem UUID: 91464a8f-4e14-4308-92bf-f40535729ec3
  Superblock backups stored on blocks: 
          32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

  Allocating group tables: done                            
  Writing inode tables: done                            
  Creating journal (16384 blocks): done
  Writing superblocks and filesystem accounting information: done 

  eugin@pg-test:~$ sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
  NAME   FSTYPE LABEL         UUID                                 MOUNTPOINT
  vda                                                              
  ├─vda1                                                           
  └─vda2 ext4                 be2c7c06-cc2b-4d4b-96c6-e3700932b129 /
  vdb                                                              
  └─vdb1 ext4   datapartition 91464a8f-4e14-4308-92bf-f40535729ec3 
  eugin@pg-test:~$ 
  ```
> сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/  
  * **_Создал точку монтирования, настроил ее, внес данные в fstab и прмонтировал ее_**
  ```console
  eugin@pg-test:~$ sudo mkdir -p /mnt/data
  eugin@pg-test:~$
  eugin@pg-test:~$ sudo chown -R postgres:postgres /mnt/data
  eugin@pg-test:~$ 
  eugin@pg-test:~$ ls -ll /mnt/
  total 4
  drwxr-xr-x 2 postgres postgres 4096 May 13 21:14 data
  eugin@pg-test:~$ 
  eugin@pg-test:~$ sudo vim /etc/fstab 
  eugin@pg-test:~$ 
  eugin@pg-test:~$ cat /etc/fstab 
  # /etc/fstab: static file system information.
  #
  # Use 'blkid' to print the universally unique identifier for a
  # device; this may be used with UUID= as a more robust way to name devices
  # that works even if disks are added and removed. See fstab(5).
  #
  # <file system> <mount point>   <type>  <options>       <dump>  <pass>
  # / was on /dev/vda2 during installation
  UUID=be2c7c06-cc2b-4d4b-96c6-e3700932b129 /               ext4    errors=remount-ro 0       1
  UUID=91464a8f-4e14-4308-92bf-f40535729ec3 /mnt/data ext4 defaults 0 2 LABEL=datapartition /mnt/data ext4 defaults 0 2
  eugin@pg-test:~$ 
  eugin@pg-test:~$ sudo mount -a
  eugin@pg-test:~$ 
  eugin@pg-test:~$ df -h -x tmpfs
  Filesystem      Size  Used Avail Use% Mounted on
  udev            1.9G     0  1.9G   0% /dev
  /dev/vda2       4.9G  2.8G  1.9G  60% /
  /dev/vdb1       6.8G   24K  6.5G   1% /mnt/data
  eugin@pg-test:~$ 
  ```

> перенесите содержимое /var/lib/postgresql/15 в /mnt/data - mv /var/lib/postgresql/15 /mnt/data
  ```console
  eugin@pg-test:~$ sudo mv /var/lib/postgresql/15 /mnt/data
  eugin@pg-test:~$ 
  eugin@pg-test:~$ ls /mnt/data/
  15  lost+found
  eugin@pg-test:~$ 
  eugin@pg-test:~$
  eugin@pg-test:~$ sudo chown -R postgres:postgres /mnt/data
  ```

> попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
> напишите получилось или нет и почему
  * **_Не получилось т.к. параметр data_directory в /etc/postgresql/15/main/postgresql.conf ссылается на старое место_** 
  ```console
  eugin@pg-test:~$ sudo -u postgres pg_ctlcluster 15 main start
  Error: /var/lib/postgresql/15/main is not accessible or does not exist
  eugin@pg-test:~$
  eugin@pg-test:~$ grep "data_directory" /etc/postgresql/15/main/postgresql.conf 
  data_directory = '/var/lib/postgresql/15/main'          # use data in another directory
  eugin@pg-test:~$ 
  ```

> задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/14/main который надо поменять и поменяйте его
> напишите что и почему поменяли
  * **_Меняем data_directory='/mnt/data/15/main' в /etc/postgresql/15/main/postgresql.conf_** 
  
> попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
> напишите получилось или нет и почему
  * **_Все получилось т.е. теперь data_directory настроен на новую точку монтрования_**
  ```console
  eugin@pg-test:~$ sudo -u postgres pg_ctlcluster 15 main start
  Warning: the cluster will not be running as a systemd service. Consider using systemctl:
    sudo systemctl start postgresql@15-main
  eugin@pg-test:~$ pg_lsclusters 
  Ver Cluster Port Status Owner    Data directory    Log file
  15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
  eugin@pg-test:~$ 
  ```

> зайдите через через psql и проверьте содержимое ранее созданной таблицы
```console
  eugin@pg-test:~$ sudo -u postgres psql
  psql (15.3 (Ubuntu 15.3-1.pgdg20.04+1))
  Type "help" for help.

  postgres=# \dt
          List of relations
   Schema | Name | Type  |  Owner   
  --------+------+-------+----------
   public | test | table | postgres
  (1 row)

  postgres=# select * from test;
   c1 
  ----
   1
  (1 row)

  postgres=# \q
  eugin@pg-test:~$ 
```

> задание со звездочкой *: не удаляя существующий GCE инстанс/ЯО сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgresql,  перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
* **_Здесь все просто, по сути, просто переносим диск с данными. Такое обычно делают, если меняют одно СХД на сервере, на другое. Или меняют сервер на более мощный, тогда отключается СХД с одного сервера и подключаются к другому. План такой:_**
  1. Останавливаем Postgres на старом сервере.
  2. Отключаем СХД на старом сервере 
  3. Подключаем СХД к новому серверу и монтируем его (правим fstab и монтируем mount -a)
  4. Проверяем, чтобы владельцем точки монтирования был postgres (chown -R postgres:postgres /mountpoint)
  5. Правим файл параметров (postgresql.conf), меня там параметр data_directory
  6. Запускаем postgres (pg_ctlcluster 15 main start)
