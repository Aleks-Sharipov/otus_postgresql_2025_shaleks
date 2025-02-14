Установка Docker:
```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
```
Запускаем Docker и активируем его на автозапуск следующей командой:
```bash
sudo systemctl enable --now docker
```
Проверяем статус Docker
```bash
systemctl status docker
```
Смотрим что всё работает:
```css
     Active: active (running) since Thu 2025-02-13 20:47:06 MSK; 55s ago
```
Проверяем наличие в репозитории Docker Hub Postgresql и устанавливаем его
```bash
docker search postgres
docker pull postgres
```
Проверяем наличие образа Postgresql в Docker
```bash
docker images
```
и смотрим, что он есть
```css
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
postgres     latest    de869b203456   23 hours ago   494MB
```
Создаём контейнер с Postgresql
```bash
docker run --rm --name postgres -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres
```
и проверяем что он поднялся
```bash
docker ps
```
```css
CONTAINER ID   IMAGE      COMMAND                  CREATED              STATUS              PORTS                                       NAMES
0293af82f90a   postgres   "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   postgres
```
Подключаемся к контейнеру с Postgresql, входим по пользователем postgres и запускаем клиент
```bash
docker exec -it postgres bash
su postgres
psql
```
Создали таблицу persons и загрузили в неё пару строк
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
Создаём сеть
```bash
sudo docker network create pg-net
```
Создаём контейнер pg-server
```bash
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres
```
Запускам его
```bash
sudo docker run -it --rm --network pg-net --name pg-client postgres psql -h pg-server -U postgres
```
Создаём в нём базу
```sql
CREATE DATABASE otus;
```
И там создадим таблицу, заполнив данными
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
Теперь будем удалять контейнер и потом создавать его по новой, для проверки наличия данных в нём

Проверим запущенные контейнеры
```bash
docker ps -a
```
```css
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                       NAMES
3fd539262bf4   postgres   "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```
Остановим и затем удалим контейнер 3fd539262bf4 
```bash
docker stop 3fd539262bf4
```
```bash
docker rm 3fd539262bf4
```
Проверим, что нет ничего
```bash
docker ps -a
```
```css
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
Создаём повторно контейнер
```bash
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres
```
Подключаемся к экземпляру Postgresql к базе Otus
```bash
psql -p 5432 -U postgres -h 89.169.160.37 -d otus -W
```
Проверяем наличие таблицы в базе Otus с помощью \dt и смотрим, что она на месте
```css
          List of relations
 Schema |   Name   | Type  |  Owner
--------+----------+-------+----------
 public | persons  | table | postgres
(1 rows)
```

