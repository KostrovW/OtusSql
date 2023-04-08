# Занятие 2 (Транзакции)
## Создаем VM
1. создать новый проект в Google Cloud Platform, Яндекс облако или на любых ВМ, докере
Yandex
1. далее создать инстанс виртуальной машины с дефолтными параметрами
lesson2
1. добавить свой ssh ключ в metadata ВМ
ssh-keygen -t ed25519
1. зайти удаленным ssh (первая сессия), не забывайте про ssh-add
ssh -i <keyfile> <name>@<vm_ip>
1. поставить PostgreSQL
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql && sudo apt install unzip && sudo apt -y install mc
1. зайти вторым ssh (вторая сессия)
ssh -i <secondkeyfile> -l <seconduser> <name>@<vm_ip>
1. запустить везде psql из под пользователя postgres
sudo -u postgres psql
sudo su postgres
psql
1. создать БД
create database iso;
\c iso
## задание с вопросами
1. выключить auto commit
\set AUTOCOMMIT OFF
\echo :AUTOCOMMIT
1. сделать в первой сессии новую таблицу и наполнить ее данными 
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
1. посмотреть текущий уровень изоляции: 
show transaction isolation level;
**read committed**
1. начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
1. в первой сессии добавить новую запись 
insert into persons(first_name, second_name) values('sergey', 'sergeev');
1. сделать выборку во второй сессии
select * from persons;
1. **видите ли вы новую запись и если да то почему?**
**не вижу, потому что при read committed запрещено грязное чтение и транзакция с записью не подтверждена**
1. завершить первую транзакцию
commit;
1. сделать выборку во второй сессии
select * from persons
1. **видите ли вы новую запись и если да то почему?**
**вижу, потому что транзакция подтверждена, и изменения из неё при read committed видимы все**
1. завершите транзакцию во второй сессии
1. начать новые но уже repeatable read транзации
set transaction isolation level repeatable read;
1. в первой сессии добавить новую запись 
insert into persons(first_name, second_name) values('sveta', 'svetova');
1. сделать выборку во второй сессии
select * from persons
1. **видите ли вы новую запись и если да то почему?**
**не вижу, потому что запрещено грязное чтение**
1. завершить первую транзакцию
commit;
1. сделать выборку во второй сессии
select * from persons
1. **видите ли вы новую запись и если да то почему?**
**не вижу, потому что при repeatable read эта выборка уже была сделана и не может измениться извне в течение транзакции**
1. завершить вторую транзакцию
1. сделать выборку во второй сессии
select * from persons
1. **видите ли вы новую запись и если да то почему?**
**вижу, потому что запись подтверждена и выборка делается первый раз в транзакции, показывая все подтвержденные записи**