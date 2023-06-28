# Занятие 20 (секционирование)

* Секционировать большую таблицу из демо базы flights

-- проверим размер таблицы

	SELECT pg_size_pretty(pg_total_relation_size('postgres_air.boarding_pass'));
	
	pg_size_pretty|
	--------------+
	2580 MB       |

	select min(bp.boarding_time),max(bp.boarding_time) from postgres_air.boarding_pass bp;
	
	min                          |max                          |
	-----------------------------+-----------------------------+
	2023-05-28 21:55:00.000 +0300|2023-08-15 17:55:00.000 +0300|

-- Создаем таблицу по примеру существующей (первичный ключ пришлось изменить)

	create table postgres_air.boarding_pass_range 
		(like postgres_air.boarding_pass including comments,
			primary key (pass_id,boarding_time))
		partition by range(boarding_time);

-- создаем партиции по месяцам

	create table postgres_air.boarding_pass_range_202305 partition of postgres_air.boarding_pass_range
		for values from ('2023-05-01'::timestamptz) to ('2023-06-01'::timestamptz);
	create table postgres_air.boarding_pass_range_202306 partition of postgres_air.boarding_pass_range
		for values from ('2023-06-01'::timestamptz) to ('2023-07-01'::timestamptz);
	create table postgres_air.boarding_pass_range_202307 partition of postgres_air.boarding_pass_range
		for values from ('2023-07-01'::timestamptz) to ('2023-08-01'::timestamptz);
	create table postgres_air.boarding_pass_range_202308 partition of postgres_air.boarding_pass_range
		for values from ('2023-08-01'::timestamptz) to ('2023-09-01'::timestamptz);

-- переносим данные

	insert into postgres_air.boarding_pass_range select * from postgres_air.boarding_pass;

-- подключаем внешние ключи как у старой таблицы

	ALTER TABLE postgres_air.boarding_pass_range ADD CONSTRAINT booking_leg_id_r_fk 
		FOREIGN KEY (booking_leg_id) REFERENCES postgres_air.booking_leg(booking_leg_id);

	ALTER TABLE postgres_air.boarding_pass_range ADD CONSTRAINT passenger_id_r_fk 
		FOREIGN KEY (passenger_id) REFERENCES postgres_air.passenger(passenger_id);

-- переименуем таблицу, чтобы встала на место старой

	alter table postgres_air.boarding_pass rename to boarding_pass_old;
	alter table postgres_air.boarding_pass_range rename to boarding_pass;

-- ещё нужно восстановить автонумерацию

	alter table postgres_air.boarding_pass alter pass_id set DEFAULT nextval('postgres_air.boarding_pass_pass_id_seq'::regclass);

-- убираем внешние ключи со старой таблицы, на её месте теперь секционированная

	ALTER TABLE postgres_air.boarding_pass_old DROP CONSTRAINT booking_leg_id_fk;
	ALTER TABLE postgres_air.boarding_pass_old DROP CONSTRAINT passenger_id_fk;

-- таблица секционирована

	select * from pg_tables where schemaname = 'postgres_air';
	
	schemaname  |tablename                 |tableowner|tablespace|hasindexes|hasrules|hastriggers|rowsecurity|
	------------+--------------------------+----------+----------+----------+--------+-----------+-----------+
	postgres_air|booking_leg               |postgres  |          |true      |false   |true       |false      |
	postgres_air|frequent_flyer            |postgres  |          |true      |false   |true       |false      |
	postgres_air|passenger                 |postgres  |          |true      |false   |true       |false      |
	postgres_air|phone                     |postgres  |          |true      |false   |true       |false      |
	postgres_air|boarding_pass_old         |postgres  |          |true      |false   |true       |false      |
	postgres_air|aircraft                  |postgres  |          |true      |false   |true       |false      |
	postgres_air|flight                    |postgres  |          |true      |false   |true       |false      |
	postgres_air|airport                   |postgres  |          |true      |false   |true       |false      |
	postgres_air|account                   |postgres  |          |true      |false   |true       |false      |
	postgres_air|booking                   |postgres  |          |true      |false   |true       |false      |
	postgres_air|boarding_pass             |postgres  |          |true      |false   |false      |false      |
	postgres_air|boarding_pass_range_202305|postgres  |          |true      |false   |true       |false      |
	postgres_air|boarding_pass_range_202306|postgres  |          |true      |false   |true       |false      |
	postgres_air|boarding_pass_range_202307|postgres  |          |true      |false   |true       |false      |
	postgres_air|boarding_pass_range_202308|postgres  |          |true      |false   |true       |false      |

-- проверяем логику перенаправления запроса в секцию

	explain select * from postgres_air.boarding_pass bp where bp.boarding_time = ('2023-06-02 09:00:00'::timestamptz);
	
	QUERY PLAN                                                                                        |
	--------------------------------------------------------------------------------------------------+
	Gather  (cost=1000.00..177402.40 rows=1410 width=40)                                              |
	  Workers Planned: 2                                                                              |
	  ->  Parallel Seq Scan on boarding_pass_range_202306 bp  (cost=0.00..176261.40 rows=588 width=40)|
			Filter: (boarding_time = '2023-06-02 09:00:00+03'::timestamp with time zone)              |
