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
* **_sudo docker run --name pg-srv -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=P@$$w@rd -d postgres:14_**


> развернуть контейнер с клиентом postgres
* **_Решил_**
  * Генерируем


> подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
* **_Решил_**
  * Генерируем

>  подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/Аналоги
* **_Решил_**
  * Генерируем


>  удалить контейнер с сервером
* **_sudo docker rm pg-srv --force_**


>  создать его заново
* **_sudo docker run --name pg-srv -v /var/lib/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=P@$$w@rd -d postgres:14_**


>  подключится снова из контейнера с клиентом к контейнеру с сервером
* **_Решил_**
  * Генерируем

>  проверить, что данные остались на месте
* **_Решил_**
  * Генерируем

>  оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами
  * **_Решил_**
  * Генерируем

