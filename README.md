## Решение заданий по направлению "Performance engineer" ##

### [1] ускорить простой запроc, добиться времени выполнения < 10ms
``` sql
select name from t1 where id = 50000;
```
> Проблема

Поиск нужного значения происходит последовательно, что замедляет выполнение запроса

![](/1/without_idx_t1_id.png)

> Решение

Создать индекс по id. Поиск нужного значения происходит с помощью индекса
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

Создать индекс по id в таблице t1 и индекс по day в таблице t2.</br>
Поиск происходит с помощью индекса. К тому же небольшое изменение текста запроса улучшило его план выполнения.<br>
Возможно нужно было создать индекс по name c помощью расширения pg_trgm<br>
... using gin(name gin_trgm_ops), но особенного улучшения не заметил - решил отказаться.
``` sql
create index idx_t1_id on t1(id);
create index idx_t2_day on t2(day);
select t2.day from t2 left join t1 on t2.t_id = t1.id and t1.name like 'a%' order by t2.day desc limit 1;
```
![](/2/with_idx_t1_id_and_idx_t2_day.png)
