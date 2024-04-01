Подключение к ВМ созданной в YC (yandex cloud) 
>ssh -i ~/.ssh/id_ed25519 komarov@158.160.130.76

Установка PG
>sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15

Проверяем запуск кластера
>sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

Заходим по пользователем postgres
>sudo -u postgres psql

Создаем таблицу
>postgres=# create table cntNUmber (num int not null);
CREATE TABLE

Наполяем данными
>postgres=# insert into cntNumber select generate_series(1,100);
INSERT 0 100

Останавливаем кластер Postgres
>sudo -u postgres pg_ctlcluster 15 main stop

Создал диск в YC 
Примонтировал к ВМ

Проверяем примонтирвоанный диск 
>lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  15G  0 disk
├─vda1 252:1    0   1M  0 part
└─vda2 252:2    0  15G  0 part /
vdb    252:16   0  10G  0 disk --Новый примонтированный диск

Установка parted согласно инструкции
>sudo apt update
>sudo apt install parted

Выбираем стандарт разделения GPT 
>sudo parted /dev/vdb mklabel gpt

Создаем новый раздел
>sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%

Проверяем 
>lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  15G  0 disk
├─vda1 252:1    0   1M  0 part
└─vda2 252:2    0  15G  0 part /
vdb    252:16   0  10G  0 disk
└─vdb1 252:17   0  10G  0 part --Новый раздел который занимает 100% диска

Создаем файловую системк для нового раздела
>sudo mkfs.ext4 -L datapartition /dev/vdb1
mke2fs 1.45.5 (07-Jan-2020)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: ad0fbded-ea34-4ebd-bcee-d32968b0690a
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

Проверяем
>sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT 

NAME   FSTYPE LABEL         UUID                                 MOUNTPOINT
vda
├─vda1
└─vda2 ext4                 be2c7c06-cc2b-4d4b-96c6-e3700932b129 /
vdb
└─vdb1 ext4   datapartition ad0fbded-ea34-4ebd-bcee-d32968b0690a

Создал каталог 
sudo mkdir -p /mnt/data

Монтирую файловую систему
sudo mount -o defaults /dev/vdb1 /mnt/data

Проверяем
>df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
udev            1.9G     0  1.9G   0% /dev
/dev/vda2        15G  2.8G   12G  20% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data

Рестарт кластера
>sudo pg_ctlcluster 15 main restart

Проверяем на месте ли диск
>lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0  15G  0 disk
├─vda1 252:1    0   1M  0 part
└─vda2 252:2    0  15G  0 part /
vdb    252:16   0  10G  0 disk
└─vdb1 252:17   0  10G  0 part /mnt/data

Монтируем раздел диска в директорию
>sudo mkdir /mnt/vdb1
>sudo mount /dev/vdb1 /mnt/vdb1

Разрешим запись на диск всем пользователям
sudo chmod a+w /mnt/vdb1

Выполняю вход под суперюзером
>sudo su postgres
Переходим в каталог лиска
>postgres@komarov-vm:/home/komarov$ cd /mnt/vdb1
Создаем дирректоию для табличного пространства
>postgres@komarov-vm:/mnt/vdb1$ mkdir tmptblspc_kom

Создаю табличное пространство в директории
>postgres@komarov-vm:/mnt/vdb1$ create tablespace komarov_ts location '/mnt/vdb1/tmptblspc_kom';

Создаем табличное простарство в кластере
>postgres=# create tablespace komarov_ts location '/mnt/vdb1/tmptblspc_kom';

Проверяем
>postgres=# \db
               List of tablespaces
    Name    |  Owner   |        Location
------------+----------+-------------------------
 komarov_ts | postgres | /mnt/vdb1/tmptblspc_kom
 pg_default | postgres |
 pg_global  | postgres |

Создал БД в своем табличном пространстве
>postgres=# create database test_kom tablespace komarov_ts;

Создал таблицы для теста переноса данных между табличными пространствами
>test_kom=# create table test_kom1 (id int);
CREATE TABLE
>test_kom=# insert into test_kom1 select g.x from generate_series(1,10) as g(x);
INSERT 0 10
>test_kom=# create table test_kom2 (id int);
CREATE TABLE
>test_kom=# drop table test_kom2;
DROP TABLE
>test_kom=# create table test_kom2 (id int) tablespace pg_default;
CREATE TABLE
>test_kom=# insert into test_kom2 select g.x from generate_series(1,10) as g(x);
INSERT 0 10
>test_kom=# select tablename,tablespace  from pg_tables where schemaname='public';
 tablename | tablespace
-----------+------------
 test_kom1 |
 test_kom2 | pg_default

Перенес таблицу из табличного пространства по умолчанию в свое табличное пространство
>test_kom=# alter table test_kom2 set tablespace komarov_ts;
ALTER TABLE
>test_kom=# select tablename,tablespace  from pg_tables where schemaname='public';
 tablename | tablespace
-----------+------------
 test_kom1 |
 test_kom2 |


Проверил содержимое двух ранее созданных таблиц 
>test_kom=# select*from test_kom1;
 id
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 rows)

>test_kom=# select*from test_kom2;
 id
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 rows)
