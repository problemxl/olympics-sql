
# Overview

This project demonstrates SQL ability using the Olympic dataset from Kaggle. This dataset was augmented with some generated data regarding athletes sponsors and mentors to help expand the dataset for the purposes of this demonstration.

# SQL

The SQL for this project was written for the PostgreSQL flavor.
## Aggregates
### List the total, average, median for each Olympic game



```sql
select slug_game,  
	count(total_medals) as total_medals,  
	avg(total_medals):: numeric(10, 2) as avg_medals,  
	percentile_cont(0.5) within group (order by total_medals):: numeric(10, 2) as median_medals  
from (select slug_game, country_name, count(medals.id) as total_medals  
		from olympics.medals  
		group by slug_game, country_name) as medal_counts  
group by slug_game;
```

### List the total, average, median for each country
```sql
select country_name,  
	count(total_medals) as total_medals,  
	avg(total_medals):: numeric(10, 2) as avg_medals,  
	percentile_cont(0.5) within group (order by total_medals):: numeric(10, 2) as median_medals  
from (select slug_game, country_name, count(medals.id) as total_medals  
		from olympics.medals  
		group by slug_game, country_name) as medal_counts  
group by country_name;
```


## Left Join
### List all Athletes and include one injury they may have had
```sql
select athlete_full_name, i.name as injury_name  
from olympics.athletes  
	left join olympics.athlete_injuries ai on athletes.id = ai.athlete_id  
	inner join olympics.injuries i on i.id = ai.injury_id;
```
![[1__PSoNtTmWbmomychydTdsA.webp]]
## Right Join
### List all Athletes and include one sponsor name that they may have had

```sql
select athlete_full_name, sponsors.name as sponsor_name  
from olympics.athletes  
	right join olympics.athlete_sponsors on athletes.id = athlete_sponsors.athlete_id  
	inner join olympics.sponsors on sponsors.id = athlete_sponsors.sponsor_id;
```

## Left Outer Join
### List all athletes who never had an injury

```sql
select athlete_full_name  
from olympics.athletes  
	left outer join olympics.athlete_injuries ai on athletes.id = ai.athlete_id;
```
![[1__PSoNtTmWbmomychydTdsA.webp]]
## Right Outer Join

### List all athletes that don’t have a sponsor

```sql
select athlete_full_name  
from olympics.athlete_sponsors  
	right outer join olympics.athletes on athletes.id = athlete_sponsors.athlete_id  
where sponsor_id is null;
```
## Inner Join
### List all athletes who had a sponsor and an injury  
```sql
select athlete_full_name  
from olympics.athletes  
	inner join olympics.athlete_injuries ai on athletes.id = ai.athlete_id  
	inner join olympics.athlete_sponsors asp on athletes.id = asp.athlete_id;
```

## Self Join
### List all athletes and mentors by name
```sql
select a1.athlete_full_name, a2.athlete_full_name  
from olympics.athletes a1  
	inner join olympics.mentors m on a1.id = m.athlete_id  
	inner join olympics.athletes a2 on m.mentor_id = a2.id;
```

## Common Table Expression

### List all athletes who received a sponsor amount greater than the median amount
```sql
with sponsor_amounts as (select athlete_id, sum(amount) as total_amount  
						 from olympics.athlete_sponsors  
						 group by athlete_id),  
	 median_amount as (select percentile_cont(0.5) within group (order by total_amount) as median_amount  
					  from sponsor_amounts),  
	 athletes_above_median as (select *  
							   from sponsor_amounts,  
								    median_amount  
							   where sponsor_amounts.total_amount > median_amount.median_amount)  
select *  
from athletes_above_median  
limit 100;
```


## Case statements
### List all athletes with their sponsor amount and a category based on the amount  
```sql
select 
	athlete_full_name,
	amount,  
	case  
		when amount < 10000 then 'Low'  
		when 10000 <= amount and amount<= 35000 then 'Medium'  
		else 'High'  
	end as sponsor_category  
from  
	( select
		athletes.id,
		athletes.athlete_full_name, sum(amount) as amount  
	from olympics.athletes  
	inner join olympics.athlete_sponsors asp on athletes.id = asp.athlete_id  
group by athletes.id, athletes.athlete_full_name) as sponsor_amounts;
```

## Rank
### Rank top 10 countries by total medals

```sql
select rank() over (order by count(medals.id) desc) as rank,
	   country_name,
	   count(medals.id) as total_medals  
from 
	olympics.medals  
group by country_name  
order by total_medals desc  
limit 10;
```

### Rank countries by the percent of gold medals relative to the countries total medal count

```sql
select country_name,  
	count(medals.id) as total_medals,  
	count(case when medal_type = 'GOLD' THEN 1 END) AS total_gold_medals,  
	(count(case when medal_type = 'GOLD' THEN 1 END)::numeric(10, 2) /  
	count(medals.id)::numeric(10, 2))::numeric(10, 2) * 100 as percent_gold_medals,  
	rank() over (order by (count(case when medal_type = 'GOLD' THEN 1 END) /  
								count(medals.id))::numeric(10, 2) desc) as rank  
from olympics.medals  
group by country_name  
order by percent_gold_medals desc;
```
## Lag/Partitions
### Canada's medal count and running total at each game

```sql
select game_name,  
	game_year,  
	country_name,  
	count(medals.id) as total_medals,  
	sum(count(medals.id)) over (partition by country_name order by game_year) as running_total  
from olympics.medals  
	inner join olympics.hosts h on h.id = medals.host_id  
where country_name = 'Canada'  
group by game_name, game_year, country_name  
order by game_year;
```

### 3 year rolling sponsor amount for all Canadian medal winners by year 

```sql
select athlete_full_name,  
	game_year,  
	sum(amount) as total_amount,  
	sum(sum(amount))  
	over (partition by athlete_full_name order by game_year rows between 2 preceding and current row) as rolling_total  
from olympics.athletes  
	inner join olympics.athlete_sponsors asp on athletes.id = asp.athlete_id  
	inner join olympics.medals m on athletes.id = m.athlete_id  
	inner join olympics.hosts h on h.id = m.host_id  
where m.country_name = 'Canada'  
group by athlete_full_name, game_year  
order by athlete_full_name, game_year;
```
