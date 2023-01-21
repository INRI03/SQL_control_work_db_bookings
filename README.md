
--1. В каких городах больше одного аэропорта?
```sql
select city, count
from
	(select city, airport_name, count(airport_name) over (partition by city)
	from airports) t
where count > 1
group by 1, 2
```

--2. В каких аэропортах есть рейсы, выполняемые самолетом с максимальной дальностью перелета?
```sql
select distinct a.airport_name
from flights f 
join airports a on a.airport_code = f.departure_airport
where aircraft_code = 
	(select aircraft_code from aircrafts
	where range = (select max("range") from aircrafts))
```	
	
--3. Вывести 10 рейсов с максимальным временем задержки вылета
```sql		
select flight_id, flight_no, (actual_departure - scheduled_departure) as delay_time
from flights f 
where actual_departure is not null
order by 3 desc
limit 10
```

--4. Были ли брони, по которым не были получены посадочные талоны?
```sql
select count(distinct b.book_ref)
from boarding_passes bp
	right join ticket_flights tf on tf.ticket_no = bp.ticket_no
	join tickets t on t.ticket_no = tf.ticket_no
	join bookings b on b.book_ref = t.book_ref 
where bp.ticket_no is null
```


-- 5. Найдите количество свободных мест для каждого рейса, их % отношение к общему количеству мест в самолете.
--Добавьте столбец с накопительным итогом - суммарное накопление количества вывезенных пассажиров из каждого аэропорта на каждый день.
-- Т.е. в этом столбце должна отражаться накопительная сумма - сколько человек уже вылетело из данного аэропорта на этом или более ранних рейсах в течении дня.
```sql
select t.*, t2.sum from (
	select * from (
		with c1 as
			(select * from
				(select aircraft_code, count(seat_no) over (partition by aircraft_code)
				from seats s) t
				group by 1,2)
		select  f.flight_id,
				f.actual_departure, 
				f.departure_airport, 
				c1."count" - count(tf.ticket_no) over (partition by tf.flight_id) as free_seats,
				round((c1."count" - count(tf.ticket_no) over (partition by tf.flight_id))::numeric / c1."count"::numeric * 100, 2) as percent_free_seats
		from ticket_flights tf 
			join flights f on f.flight_id = tf.flight_id
			join aircrafts a on a.aircraft_code = f.aircraft_code 
			join c1 on c1.aircraft_code = a.aircraft_code
		where f.status = 'Departed' or f.status = 'Arrived' and c1.aircraft_code = f.aircraft_code)	t
		group by 1,2,3,4,5)
		t
	join
		(select f.flight_id,
		   		f.actual_departure, 
		   		f.departure_airport,
		   		sum(count(tf.ticket_no)) over (partition by f.departure_airport, date_trunc('day', f.actual_departure) order by f.actual_departure)
		from ticket_flights tf 
			join flights f on f.flight_id = tf.flight_id
		where f.status = 'Departed' or f.status = 'Arrived'
		group by 1, 2)
		t2
			on t2.flight_id = t.flight_id
order by 3,2
```

--6. Найдите процентное соотношение перелетов по типам самолетов от общего количества.
```sql
select t.code, round(count::numeric / (select count(*) from ticket_flights tf)::numeric * 100, 1) as "percent"
	from
	(select f.aircraft_code as code, count(*) over (partition by f.aircraft_code)
from ticket_flights tf
join flights f on f.flight_id = tf.flight_id) t
group by 1, 2
```

--7. Были ли города, в которые можно  добраться бизнес - классом дешевле, чем эконом-классом в рамках перелета?
```sql
with c1 as 
		(select tf.amount, f.flight_id, f.departure_airport, f.arrival_airport 
			from ticket_flights tf 
			join flights f on tf.flight_id = f.flight_id
			where tf.fare_conditions = 'Economy'
		group by 2,1),
	c2 as
		(select tf.amount, f.flight_id, f.departure_airport, f.arrival_airport
			from ticket_flights tf 
			join flights f on tf.flight_id = f.flight_id
			where tf.fare_conditions = 'Business'
		group by 2,1),
	c3 as 
		(select airport_code, city
			from airports)
select distinct c3.city
	from c1
	join c2 on c2.flight_id = c1.flight_id
	join c3 on c3.airport_code = c2.arrival_airport
	where c2.amount < c1.amount
```		
	
--8. Между какими городами нет прямых рейсов?
```sql	
create view view_1 as
	(select distinct a.city as dep_city, a2.city as arr_city
		from flights f
	join airports a on a.airport_code = f.departure_airport
	join airports a2 on a2.airport_code = f.arrival_airport)

create view view_2 as
	(select city as dep_city
		from airports)

create view view_3 as
	(select city as arr_city
		from airports)

select dep_city, arr_city
	from view_2, view_3
except
select *
	from view_1
where dep_city != arr_city
```

--9. Вычислите расстояние между аэропортами, связанными прямыми рейсами, сравните с допустимой максимальной дальностью перелетов  в самолетах, обслуживающих эти рейсы 
```sql
with c1 as
	(select distinct a1.airport_name as dep_airport,
			a2.airport_name as arr_airport,
			a."range",
			round (acos(sind(a1.latitude) * sind(a2.latitude) + cosd(a1.latitude) * cosd(a2.latitude) * cosd(a1.longitude - a2.longitude))::numeric, 2) * 6371 as distance
	from flights f 
	join airports a1 on a1.airport_code = f.departure_airport
	join airports a2 on a2.airport_code = f.arrival_airport
	join aircrafts a on a.aircraft_code = f.aircraft_code)
select c1.*,
		(case when c1."range" > c1.distance then 'ok'
		else 'break'
		end) as distance_test
from c1
```
