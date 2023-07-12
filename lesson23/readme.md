# Занятие 23 (триггеры)

* Скрипт и развернутое описание задачи – в ЛК (файл hw_triggers.sql) или по ссылке: https://disk.yandex.ru/d/l70AvknAepIJXQ
* В БД создана структура, описывающая товары (таблица goods) и продажи (таблица sales).
* Есть запрос для генерации отчета – сумма продаж по каждому товару.
* БД была денормализована, создана таблица (витрина), структура которой повторяет структуру отчета.
* Создать триггер на таблице продаж, для поддержки данных в витрине в актуальном состоянии (вычисляющий при каждой продаже сумму и записывающий её в витрину)

	SET search_path = pract_functions, publ

-- триггерная функция на пересчет по изменению таблицы goods

	CREATE OR REPLACE FUNCTION pract_functions.goods_sum()
	RETURNS trigger
	AS
	$TRIG_FUNC$
	begin
		IF TG_OP <> 'INSERT' then
			delete from good_sum_mart
				where good_name in(select good_name from old_table);
		end if;
		IF TG_OP <> 'DELETE' then
			insert into good_sum_mart
				select n.good_name, coalesce(sum(s.sales_qty) * n.good_price, 0.0) as sales_sum 
				from sales s right outer join new_table n on s.good_id = n.goods_id
				group by n.good_name, n.good_price;
		end if;
		RETURN NEW;
	END;
	$TRIG_FUNC$
	  LANGUAGE plpgsql
	  VOLATILE
	  SET search_path = pract_functions, public;

-- удаление

	DROP TRIGGER IF EXISTS goods_sum_upd ON goods;
	DROP TRIGGER IF EXISTS goods_sum_ins ON goods;
	DROP TRIGGER IF EXISTS goods_sum_del ON goods;

-- создание триггеров

	CREATE TRIGGER goods_sum_ins
	AFTER insert ON goods
	REFERENCING NEW TABLE AS new_table
	FOR EACH STATEMENT
	EXECUTE PROCEDURE goods_sum();

	CREATE TRIGGER goods_sum_upd
	AFTER update ON goods
	REFERENCING OLD TABLE AS old_table NEW TABLE AS new_table
	FOR EACH STATEMENT
	EXECUTE PROCEDURE goods_sum();

	CREATE TRIGGER goods_sum_del
	AFTER delete ON goods
	REFERENCING OLD TABLE AS old_table
	FOR EACH STATEMENT
	EXECUTE PROCEDURE goods_sum();

-- триггерная функция на пересчет по изменению таблицы sales

	CREATE OR REPLACE FUNCTION pract_functions.sales_sum()
	RETURNS trigger
	AS
	$TRIG_FUNC$
	begin
		IF TG_OP <> 'INSERT' then
			delete from good_sum_mart
				where good_name in (select g.good_name 
					from goods g join (select distinct o.good_id from old_table o) a
						on g.goods_id = a.good_id);
		end if;
		IF TG_OP <> 'DELETE' then
			insert into good_sum_mart (good_name, sum_sale) 
				select g.good_name, coalesce(sum(s.sales_qty) * g.good_price, 0.0) as sum_sale 
					from sales s right outer join goods g on s.good_id = g.goods_id
					where g.goods_id in(select n.good_id from new_table n) 
					group by g.good_name, g.good_price;
		else
			insert into good_sum_mart (good_name, sum_sale) 
				select g.good_name, coalesce(sum(s.sales_qty) * g.good_price, 0.0) as sum_sale 
					from sales s right outer join goods g on s.good_id = g.goods_id
					where g.goods_id in(select o.good_id from old_table o) 
					group by g.good_name, g.good_price;
		end if;
		return NEW;
	END;
	$TRIG_FUNC$
	  LANGUAGE plpgsql
	  VOLATILE
	  SET search_path = pract_functions, public;
	 
	 
-- удаление

	DROP TRIGGER IF EXISTS sales_sum_upd ON sales;
	DROP TRIGGER IF EXISTS sales_sum_ins ON sales;
	DROP TRIGGER IF EXISTS sales_sum_del on sales;

-- создание триггеров

	CREATE TRIGGER sales_sum_ins
	AFTER insert ON sales
	REFERENCING NEW TABLE AS new_table
	FOR EACH STATEMENT
	EXECUTE PROCEDURE sales_sum();

	CREATE TRIGGER sales_sum_upd
	AFTER update ON sales
	REFERENCING OLD TABLE AS old_table NEW TABLE AS new_table
	FOR EACH STATEMENT
	EXECUTE PROCEDURE sales_sum();

	CREATE TRIGGER sales_sum_del
	AFTER delete ON sales
	REFERENCING OLD TABLE AS old_table
	FOR EACH STATEMENT
	EXECUTE PROCEDURE sales_sum();
	