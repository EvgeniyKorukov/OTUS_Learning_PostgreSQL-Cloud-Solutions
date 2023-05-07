#### *Отчет о выполнении домашнего задания:*


> сделать в GCE/ЯО/Аналоги инстанс с Ubuntu 20.04
* **_Создал виртуальную машину с Ubuntu 20.04 в Yandex Cloud_**


> поставить на нем Docker Engine
* **_Установку Docker взял с официального сайта https://docs.docker.com/engine/install/ubuntu/_**
  * Set up the repository:
	  * sudo apt-get update && sudo apt-get -y install ca-certificates curl gnupg
  * Add Docker’s official GPG key:
	  * sudo install -m 0755 -d /etc/apt/keyrings && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && sudo chmod a+r /etc/apt/keyrings/docker.gpg
  * Use the following command to set up the repository:
	  * echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  * Install Docker Engine, containerd, and Docker Compose.
	  * sudo apt-get -y update && sudo apt-get -y install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


> сделать каталог /var/lib/postgres
* **_sudo mkdir -p /var/lib/postgres_**


> развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres
* **_sudo docker run --name pg-srv -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=Pass1234 -d postgres:14_**


> развернуть контейнер с клиентом postgres
* **_Подразумевается только развернуть контейнер т.е. только создать, а не запускать_**
	* Вариант 1 (использование стороннего контейнера):
		* sudo docker run -dit --name=pg-client1 codingpuss/postgres-client
	* Вариант 2 (создание образа с помощью Dockerfile):
		* sudo docker build -t pg-client2 -f /tmp/PG_Project/Dockerfile_pg-client .
		* sudo docker run -dit --name=pg-client2 pg-client2
		* Содержимое /tmp/PG_Project/Dockerfile_pg-client:
			* FROM alpine:3.17
			* RUN apk --no-cache add postgresql14-client
			* CMD ["/bin/sh"]


> подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
* **_Вариант 1:_**
	* sudo docker exec -it pg-client1 psql postgresql://postgres:Pass1234@10.129.0.50:5432/postgres
  		* create table t1(i int);
  		* INSERT INTO t1(i) SELECT generate_series(1,1000);
	* или
  	* sudo docker exec -it pg-client1 psql postgresql://postgres:Pass1234@10.129.0.5:5432/postgres -c "create table t1(i int)"
  	* sudo docker exec -it pg-client1 psql postgresql://postgres:Pass1234@10.129.0.5:5432/postgres -c "INSERT INTO t1(i) SELECT generate_series(1,1000)"
* **_Вариант 2:_**
	* sudo docker exec -it pg-client2 psql -h 10.129.0.5 -d postgres -U postgres
  		* create table t1(i int);
  		* INSERT INTO t1(i) SELECT generate_series(1,1000);
	* или
  	* sudo docker exec -it pg-client2 psql -h 10.129.0.5 -d postgres -U postgres -c "create table t1(i int)"
  	* sudo docker exec -it pg-client2 psql -h 10.129.0.5 -d postgres -U postgres -c "INSERT INTO t1(i) SELECT generate_series(1,1000)"


>  подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/Аналоги
* **_Это не является проблемой при наличии общедоступного адреса т.к. порт к СУБД у нас проброшен из контейнера в нашу виртуальную машину с докером._**
  * sudo psql -h 51.250.102.80 -U postgres -d postgres
    * Замечание 1: Мы должны знать пароль от пользователя postgres
    * Замечание 2: Должен быть прописан доступ в pg_hba.conf
    * Замечание 3: Должен быть настроен параметр listen_addresses
    * Замечание 4: Должен быть установлен клиент PosgtreSQL (psql)


>  удалить контейнер с сервером
* **_sudo docker rm pg-srv --force_**


>  создать его заново
* **_sudo docker run --name pg-srv -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=Pass1234 -d postgres:14_**


>  подключится снова из контейнера с клиентом к контейнеру с сервером
* **_На этот раз уже через запуск контейнера с клиентом и потом-удаление контейнера_**
	* Вариант 1 (использование стороннего контейнера, с явным указанием ip):
		* sudo docker run -it --rm --name pg-client jbergknoff/postgresql-client postgresql://postgres:Pass1234@10.129.0.5:5432/postgres
	* Вариант 2 (использование стороннего контейнера, с использованием link и указанием имени сервера):
		* sudo docker run -it --rm --name pg-client --link pg-srv jbergknoff/postgresql-client postgresql://postgres:Pass1234@pg-srv:5432/postgres
	* Вариант 3 (создание образа с помощью Dockerfile, явным указанием ip):
		* Содержимое /tmp/PG_Project/Dockerfile_pg-client:
			* FROM alpine:3.17
			* RUN apk --no-cache add postgresql14-client
			* ENV PGHOST=pg-srv
			* ENV PGPORT=5432
			* ENV PGUSER=postgres
			* ENV PGPASSWORD=Pass1234
			* ENV PGDATABASE=postgres
			* ENTRYPOINT [ "psql" ]
		* sudo docker build -t pg-client2 -f /tmp/PG_Project/Dockerfile_pg-client .
		* sudo docker run -it --rm --name pg-client2 pg-client2 -h 10.129.0.5
	* Вариант 4 (создание образа с помощью Dockerfile, с использованием link и указанием имени сервера):
		* Содержимое /tmp/PG_Project/Dockerfile_pg-client:
			* FROM alpine:3.17
			* RUN apk --no-cache add postgresql14-client
			* ENV PGHOST=pg-srv
			* ENV PGPORT=5432
			* ENV PGUSER=postgres
			* ENV PGPASSWORD=Pass1234
			* ENV PGDATABASE=postgres
			* ENTRYPOINT [ "psql" ]
		* sudo docker build -t pg-client2 -f /tmp/PG_Project/Dockerfile_pg-client .
		* sudo docker run -it --rm --link pg-srv --name pg-client2 pg-client2
	

>  проверить, что данные остались на месте
* **_Данные на месте т.к. они хрантся на нашей виртуалке. Если бы хранили в контейнере Docker, то они удалялись бы при каждой остановке/запуске контейнера**_**


>  оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами
  1. В моем решении с postgres_client есть одна особенность, контенер с клиентом всегда остается запущенным, что не есть правильно. Но в заднии было создать контейнер с клиентом. А правильно было бы-запусить контейнер с клиентом, поработать с ним и потом удалить его (run --rm)
  2. В Dockerfile можно было бы использовать 'ENTRYPOINT [ "psql" ]' вместо 'CMD ["/bin/sh"]'. В этом случаем можно убрать psql из строки с запуском контейнера и это подходит для случая, когда контейнер запускается, а после работы с ним - удаляется. У меня не получилось сделать так, чтобы с 'ENTRYPOINT [ "psql" ]' котейнер с клиентом оставался рабочим, после run, чтобы работать с ним через exec.
  3. При изменении Docker-file (/tmp/PG_Project/Dockerfile_pg-client), не разбрался как обновлять образ 'sudo docker build -t pg-client2 -f /tmp/PG_Project/Dockerfile_pg-client .'. При каждом build, он создает новый образ, а не обновляет текущий. Обновлять пока получается только через удаление существующего образа.

