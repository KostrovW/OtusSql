# Занятие 2 (Транзакции)
## Создаем VM
* создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере

Yandex
* далее создать инстанс виртуальной машины с дефолтными параметрами

lesson2
* добавить свой ssh ключ в metadata ВМ

ssh-keygen -t ed25519
* зайти удаленным ssh (первая сессия), не забывайте про ssh-add

ssh -i <keyfile> <name>@<vm_ip>
* поставить PostgreSQL

sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc
* зайти вторым ssh (вторая сессия)

ssh -i <secondkeyfile> -l <seconduser> <name>@<vm_ip>
* запустить везде psql из под пользователя postgres

sudo -u postgres psql

sudo su postgres

psql
* создать БД

create database iso;

\c iso
## задание с вопросами
* выключить auto commit

\set AUTOCOMMIT OFF

\echo :AUTOCOMMIT
* сделать в первой сессии новую таблицу и наполнить ее данными 

create table persons(id serial, first_name text, second_name text); 

insert into persons(first_name, second_name) values('ivan', 'ivanov'); 

insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
* посмотреть текущий уровень изоляции: 

show transaction isolation level;

**read committed**
* начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

commit;
* в первой сессии добавить новую запись 

insert into persons(first_name, second_name) values('sergey', 'sergeev');
* сделать выборку во второй сессии

select * from persons;
* **видите ли вы новую запись и если да то почему?**

**не вижу, потому что при read committed запрещено грязное чтение и транзакция с записью не подтверждена**
* завершить первую транзакцию

commit;
* сделать выборку во второй сессии

select * from persons
* **видите ли вы новую запись и если да то почему?**

**вижу, потому что транзакция подтверждена, и изменения из неё при read committed видимы все**
* завершите транзакцию во второй сессии

commit;
* начать новые но уже repeatable read транзации

set transaction isolation level repeatable read;
* в первой сессии добавить новую запись 

insert into persons(first_name, second_name) values('sveta', 'svetova');
* сделать выборку во второй сессии

select * from persons
* **видите ли вы новую запись и если да то почему?**

**не вижу, потому что запрещено грязное чтение**
* завершить первую транзакцию

commit;
* сделать выборку во второй сессии

select * from persons
* **видите ли вы новую запись и если да то почему?**

**не вижу, потому что при repeatable read эта выборка уже была сделана и не может измениться извне в течение транзакции**
* завершить вторую транзакцию

commit;
* сделать выборку во второй сессии

select * from persons
* **видите ли вы новую запись и если да то почему?**

**вижу, потому что запись подтверждена и выборка делается первый раз в транзакции, показывая все подтвержденные записи**
