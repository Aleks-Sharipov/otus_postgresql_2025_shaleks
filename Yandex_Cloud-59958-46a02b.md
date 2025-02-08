# Virtual Machines (Compute Cloud) https://cloud.yandex.ru/docs/free-trial/

Создал виртуальную машину:
https://console.yandex.cloud/folders/b1gk4o29jc5t2q4s916o/compute/instance/epdpd5ggdaumtu4f4te3/overview

С именем: otus-db-pg-vm-1

Создал подсеть: 
Имя: default-ru-central1-b
Зона: ru-central1-b

Добавил пользователя: otus

Для подключения через командную строку в Windows создал ssh ключ c:\ssh\otus_pgvm.pub

Сгенерировал командой ssh-keygen:
```bash
ssh-keygen -t rsa -b 2048
#Generating public/private rsa key pair.
#Enter file in which to save the key (C:\Users\sh-al/.ssh/id_rsa): c:\ssh\otus_pgvm
```
Добавил сгенерированный ключ в ВМ.

Подключился к ВМ в командной строке:

```bash
ssh -i c:\ssh\otus_pgvm otus@158.160.17.212
```

Запустил установку PostgreSQL && other из скрипта:
```bash
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc
```

После установки подключился к PostgreSQL командой:
```bash
sudo -u postgres psql
```

Установил пароль для PostgreSQL:
```bash
\password
#12345
```

После установки PostgreSQL добавил настройки сетевых правил для подключения к PostgreSQL:
```bash
sudo nano /etc/postgresql/13/main/postgresql.conf
```

Добавил в конфиг:

#listen_addresses = 'localhost'

listen_addresses = '*'

После этого прописал правила для хоста при подключении к PostgreSQL:
```bash
sudo nano /etc/postgresql/13/main/pg_hba.conf
```

Добавил в конфиг:

#host    all             all             127.0.0.1/32            scram-sha-256 password

host    all             all             0.0.0.0/0               scram-sha-256 

Перезапустил PostgreSQL:
```bash
sudo systemctl restart postgresql
```

Подключился к экземпляру PostgreSQL:
```bash
psql -h 158.160.17.212 -U postgres
```


