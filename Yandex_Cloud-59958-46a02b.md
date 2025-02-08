# Virtual Machines (Compute Cloud) https://cloud.yandex.ru/docs/free-trial/

Создание виртуальной машины:
https://cloud.yandex.ru/docs/compute/quickstart/quick-create-linux

name vm: otus-db-pg-vm-1

Создать сеть:
Каталог: default
Имя: otus-vm-db-pg-net-1

Доступ
username: otus

настройка OpenSSH в Windows
Параметры -> Система -> Дополнительные компоненты -> клиент Open SSH (добавить)
Службы (Service) -> OpenSSH SSH Server (запустить)


Сгенерировать ssh-key:
```bash
ssh-keygen -t rsa -b 2048
name ssh-key: yc_key
chmod 600 ~/.ssh/yc_key.pub
ls -lh ~/.ssh/
cat ~/.ssh/yc_key.pub # в Windows C:\Users\<имя_пользователя>\.ssh\yc_key.pub
```
Подключение к VM:
https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh

```bash
ssh -i ~/yc_key otus@130.193.45.81 # в Windows ssh -i <путь_к_ключу/имя_файла_ключа> <имя_пользователя>@<публичный_IP-адрес_виртуальной_машины>

Установка Postgres:
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc

pg_lsclusters

Установить пароль для Postgres:
sudo -u postgres psql
\password   #12345
\q

Добавить сетевые правила для подключения к Postgres:
cd /etc/postgresql/17/main/
sudo nano /etc/postgresql/17/main/postgresql.conf
#listen_addresses = 'localhost'
listen_addresses = '*'

sudo nano /etc/postgresql/17/main/pg_hba.conf
#host    all             all             127.0.0.1/32            scram-sha-256 password
host    all             all             0.0.0.0/0               scram-sha-256 

sudo pg_ctlcluster 17 main restart

Подключение к Postgres:
psql -h 130.193.45.81 -U postgres

\l


```