

## Cоздал виртуальную машину c Ubuntu 20.04/22.04 LTS в ЯО/докере

```
alex@alex:~$ ssh -i ~/.ssh/yc_key otus@158.160.17.212
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-97-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Wed Feb 26 11:05:29 PM UTC 2025

  System load:  0.0                Processes:             140
  Usage of /:   25.3% of 13.63GB   Users logged in:       1
  Memory usage: 7%                 IPv4 address for eth0: 10.129.0.28
  Swap usage:   0%
```

## поставил на нее PostgreSQL 15 через sudo apt
```
otus@otus-db-pg-vm-1:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```

## проверил что кластер запущен через sudo -u postgres pg_lsclusters
```
otus@otus-db-pg-vm-1:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## зашел из под пользователя postgres в psql и созд произвольную таблицу с произвольным содержимым

```sql
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)

postgres=# SELECT * FROM test;
 c1
----
 1
(1 row)
```

## остановил postgres например через sudo -u postgres pg_ctlcluster 15 main stop

```
otus@otus-db-pg-vm-1:~$ sudo systemctl stop postgresql@15-main
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## создал новый диск к ВМ размером 10GB .Добавляю диск в ЯО и подсоединяю к ВМ
## добавил свеже-созданный диск к виртуальной машине - зашел в режим ее редактирования и дальше выбрал пункт attach existing disk
```
otus@otus-db-pg-vm-1:~$ sudo lsblk -o NAME,FSTYPE,SIZE,MOUNTPOINT,LABEL
NAME   FSTYPE     SIZE MOUNTPOINT        LABEL
loop0  squashfs  63.3M /snap/core20/1822
loop1  squashfs 109.9M /snap/lxd/24322
loop2  squashfs  49.8M /snap/snapd/18357
loop3  squashfs  40.4M /snap/snapd/20671
loop4            62.9M /snap/core20/2182
loop5              87M /snap/lxd/27037
vda                18G
├─vda1              1M
└─vda2 ext4        18G /
vdb                10G
```

## проинициализировал диск согласно инструкции и подмонтировал файловую систему

```
otus@otus-db-pg-vm-1:~$ sudo parted /dev/vdb mklabel gpt
Information: You may need to update /etc/fstab.

otus@otus-db-pg-vm-1:~$ sudo mount -a
otus@otus-db-pg-vm-1:~$ sudo parted /dev/vdb mklabel gpt
Warning: The existing disk label on /dev/vdb will be destroyed and all data on this disk will be lost. Do you want to
continue?
Yes/No? yes
Information: You may need to update /etc/fstab.

otus@otus-db-pg-vm-1:~$ sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
Information: You may need to update /etc/fstab.

otus@otus-db-pg-vm-1:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.3M  1 loop /snap/core20/1822
loop1    7:1    0 109.9M  1 loop /snap/lxd/24322
loop2    7:2    0  49.8M  1 loop /snap/snapd/18357
loop3    7:3    0  40.4M  1 loop /snap/snapd/20671
loop4    7:4    0  62.9M  1 loop /snap/core20/2182
loop5    7:5    0    87M  1 loop /snap/lxd/27037
vda    252:0    0    18G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    18G  0 part /
vdb    252:16   0    10G  0 disk
└─vdb1 252:17   0    10G  0 part
otus@otus-db-pg-vm-1:~$ sudo mkfs.ext4 -L dom3 /dev/vdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: 9578ac23-85b9-42aa-8c9b-e2239658c823
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

## перезагрузил инстанс. диск остается примонтированным

```
otus@otus-db-pg-vm-1:~$ sudo pg_ctlcluster 15 main restart
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
otus@otus-db-pg-vm-1:~$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        18G  4.7G   13G  28% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data

```
## сделал пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/

```
otus@otus-db-pg-vm-1:~$ sudo chown -R postgres:postgres /mnt/data/
```

## перенес содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15 /mnt/data

```
otus@otus-db-pg-vm-1:~$ sudo mv /var/lib/postgresql /mnt/data
```

## пытаюсь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start 
#  Кластер не запустился, так как по данному пути его не существует

```
otus@otus-db-pg-vm-1:~$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
15  main    5432 down   <unknown> /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его? напишите что и почему поменяли
В файле postgresql.conf надо поменять data_directory = '/var/lib/postgresql/15/main' на data_directory = '/mnt/data/postgresql/15/main'

```
otus@otus-db-pg-vm-1:~$  sudo nano /etc/postgresql/15/main/postgresql.conf
otus@otus-db-pg-vm-1:~$ sudo systemctl start postgresql
otus@otus-db-pg-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
15  main    5432 online postgres /mnt/data/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

## пытаюсь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
#  Кластер уже запущен после start postgresql, что мы и увидели в предыдущем пункте

```
otus@otus-db-pg-vm-1:~$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Cluster is already running.
```

## зайшел через через psql и проверил содержимое ранее созданной таблицы

```sql
otus@otus-db-pg-vm-1:~$ psql -h localhost -U postgres -d postgres
Password for user postgres:
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.

postgres=# SELECT * FROM test;
 c1
----
 1
(1 row)
```

## данные на месте
