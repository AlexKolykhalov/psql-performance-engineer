## Решение заданий по направлению "Performance engineer" ##

### [1] ускорить простой запроc, добиться времени выполнения < 10ms
``` sql
select name from t1 where id = 50000;
```
> Проблема

Поиск нужного значения происходит последовательно, что замедляет выполнение запроса

![](/1/without_idx_t1_id.png)

> Решение

Создать индекс по ```id```. Поиск нужного значения происходит с помощью индекса
``` sql
create index idx_t1_id on t1(id);
select name from t1 where id = 50000;
```
![](/1/with_idx_t1_id.png)

### [2] ускорить запрос "max + left join", добиться времени выполнения < 10ms
``` sql
select max(t2.day) from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%';
```
> Проблема

В этом случае происходит последование сканирование двух таблиц, без использования индексов это не производительно

![](/2/without_idx_t1_id_and_idx_t2_day.png)

> Решение

Создать индекс по ```id``` в таблице ```t1``` и индекс по ```day``` в таблице ```t2```.</br>
Поиск происходит с помощью индекса. К тому же небольшое изменение текста запроса улучшило его план выполнения.<br>
Возможно нужно было создать индекс по ```name``` c помощью расширения ```pg_trgm```<br>
... ```using gin(name gin_trgm_ops)```, но особенного улучшения не заметил - решил отказаться.
``` sql
create index idx_t1_id on t1(id);
create index idx_t2_day on t2(day);
select t2.day from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%' order by t2.day desc limit 1;
```
![](/2/with_idx_t1_id_and_idx_t2_day.png)

### [3] ускорить запрос "anti-join", добиться времени выполнения < 10sec
``` sql
select day from t2 where t_id not in ( select t1.id from t1 );
```
> Проблема

Последование сканирование таблицы ```t1```, дальнейшее засовывание ```Materialize``` - сильно замедляет работу запроса,<br>
плюс последовательная проверка каждого элемента из ```t2``` на условие - тоже не добавляет скорости.

![](/3/before.png)

> Решение

 Перестройка запроса помогла улучшить план выполнения
``` sql
explain select day from t2 left join t1 on t_id = t1.id where t1.id is null;
```
![](/3/after.png)

### [4] ускорить запрос "semi-join", добиться времени выполнения < 10sec
``` sql
select day from t2 where t_id in ( select t1.id from t1 where t2.t_id = t1.id) and day > to_char(date_trunc('day',now()- '1 months'::interval),'yyyymmdd');
```
> Проблема

Поиск подходящего значения для ```day``` происходит медленно, т.к. нет индекса

![](/4/before.png)

> Решение

После создания индекса по ```day``` проверка условия происходит намного быстрее,<br>
плюс - небольшое изменение запроса помогло улучшить план выполнения
``` sql
create index idx_t2_day on t2(day);
SELECT day FROM t2 LEFT JOIN t1 ON t2.t_id = t1.id WHERE day > to_char(date_trunc('day', now() - '1 months'::interval), 'yyyymmdd');
```
![](/4/after.png)

### [5] ускорить работу "savepoint + update", добиться постоянной во времени производительности (число транзакций в секунду)
```
Virtual Machine
CPU: Intel Core i7 870 @ 4x 2.926GHz
RAM: 2759MiB / 6804MiB
```
> Проблема

![](/5/before.png)

> Решение

![](/5/after.png)
