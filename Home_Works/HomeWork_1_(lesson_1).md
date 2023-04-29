#### *Отчет о выполнении домашнего задания:*


> создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, например postgres2023-, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально на уровне GCP)
> далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами - 1-2 ядра, 2-4Гб памяти, любой линукс, на курсе Ubuntu 100%
* **_Решил, что это задание буду делать в Облаке Яндекса_**
  * Генерируем ssh-ключи (private и public) для доступа к виртуальной машине.
    * ssh-keygen -t rsa -b 2048
      * eugin_yandex_key
  * Правим права для ssh-ключей
    * chmod 600 ~/.ssh/eugin_yandex_key.pub
  * Проверяем права для ssh-ключей
    * ls -lh ~/.ssh/
  * сохраняем вывод публичного ключа
    * cat ~/.ssh/eugin_yandex_key.pub 
  * Создал виртуальную машину в Яндекс Облаке:
    * Имя машины: postgres-20230429
    * CPU: 2
    * RAM: 2 Gb 
    * SSD: 8 Gb 
    * Login user: eugin


> добавить свой ssh ключ в GCE metadata
* **_Это сделал выше т.к. без этого не возможно создать виртуальную машину в Яндексе_**


> зайти удаленным ssh (первая сессия), не забывайте про ssh-add
* **_ssh -i ~/.ssh/eugin_yandex_key eugin@158.160.15.128_**


> поставить PostgreSQL из пакетов apt install
* **_sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15_**


> зайти вторым ssh (вторая сессия)
* **_Зашел_**


> запустить везде psql из под пользователя postgres
* **_Запустил_**
  * sudo -u postgres psql 


> выключить auto commit
* **_Выключил_**
  * \set AUTOCOMMIT OFF


> сделать в первой сессии новую таблицу и наполнить ее данными
* **_Создал и вставил данные_**
  * create table persons(id serial, first_name text, second_name text);
  * insert into persons(first_name, second_name) values('ivan', 'ivanov');
  * insert into persons(first_name, second_name) values('petr', 'petrov');
  * commit;


> посмотреть текущий уровень изоляции: show transaction isolation level
* **_show transaction isolation level;_**
  * read committed


> начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
* **_begin;_**


> в первой сессии добавить новую запись
> insert into persons(first_name, second_name) values('sergey', 'sergeev');
> сделать select * from persons во второй сессии
> видите ли вы новую запись и если да то почему?
* **_Не видим. У нас включен режим изоляции read commited (показывать только зафиксированные транзакции), но выключен autocommit._**


> завершить первую транзакцию - commit;
> сделать select * from persons во второй сессии
> видите ли вы новую запись и если да то почему?
* **_Видим т.к. транзакция была зафиксирована в первой сессии_**


> завершите транзакцию во второй сессии
* **_commit;_**


> начать новые но уже repeatable read транзакции - set transaction isolation level repeatable read;
> в первой сессии добавить новую запись
> insert into persons(first_name, second_name) values('sveta', 'svetova');
> сделать select * from persons во второй сессии
> видите ли вы новую запись и если да то почему?
* **_Не видим т.к. при уровне изоляции repeatable read, создается снимок данных на момент начала транзакции и все изменения не попадут в рамках текущей транзакции_**


> завершить первую транзакцию - commit;
* **_commit;_**


> сделать select * from persons во второй сессии
> видите ли вы новую запись и если да то почему?
* **_Не видим т.к. при уровне изоляции repeatable read, создается снимок данных на момент начала транзакции и все изменения не попадут в рамках текущей транзакции_**


> завершить вторую транзакцию
* **_commit;_**


> сделать select * from persons во второй сессии
> видите ли вы новую запись и если да то почему?
* **_Теперь видим. Потому что мы начинаем новую транзакцию с новым снимком и уже новыми данными_**


