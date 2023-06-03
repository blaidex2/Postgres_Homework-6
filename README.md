### Домашнее задание №6

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/bdf4d454-bb26-471e-80d3-377a693ca0be)

 
1.	Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
>sudo -u postgres psql
show deadlock_timeout;
alter system set deadlock_timeout to 200;
show deadlock_timeout;

--покажет неизмененное кол-во миллисекунд, нужно перезагрузить постгре
>sudo systemctl restart postgresql
sudo -u postgres psql
show deadlock_timeout;

--покажет измененное кол-во миллисекунд

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/d432d767-214e-43c5-a41b-71b62900da29)

Создадим таблицу и наполним ее данными:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/8be1a371-a892-462b-8912-0aba9d105f28)

Для просмотра блокировок:

>select locktype, relation::regclass, mode, transactionid as tid, virtualtransaction as vtid, pid, granted from pg_locks;

В новой сессии:
 
 ![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/5766a653-db96-4167-be31-fc56984f9434)


В 1 сессии получаем (цена не изменилась, смотрим на блокировки):
 
 ![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/58e5bdfe-30c1-4f32-9e13-fc409b8d7367)


>В таблице "films" теперь стоит блокировка RowExclusiveLock.
Также появился идентификатор транзакции transactionid на котором удерживается блокировка ExclusiveLock.
Такой идентификатор появляется у каждой транзакции, потенциально меняющей состояние базы данных.

Выполнив коммит во второй сессии, получаем следующее состояние блокировок:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/11ca9eaa-8bc5-4fb2-8eec-e25b6c1e124f)

2.	Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

В первой сессии будем смотреть состояние блокировок, используя запрос:
>select lock.locktype, lock.relation::regclass, lock.mode, lock.transactionid as tid, lock.virtualtransaction as vtid, lock.pid, lock.granted
from pg_catalog.pg_locks lock
left join pg_catalog.pg_database db on db.oid = lock.database
where not lock.pid = pg_backend_pid()
order by lock.pid;

Во второй сессии:
![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/58dc2f88-2e2f-461d-8df1-cdfbcc67e4ad)
 
В первой сессии видим:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/3cbe9569-0843-48a0-a5b0-d870eac4cc00)

В третьей сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/460da08d-4a87-47cc-b1e8-c6be278555e7)

 
В первой сессии видим:
 
![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/be867e36-42dd-4b93-b2dc-ddb1f0d9c744)


В четвертой сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/8841fa2b-b4d3-4782-b369-0b745c0cb702)


В первой сессии видим:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/376bf4d4-e966-4900-8cad-f8fb78dac7fa)


>1223 (вторая сессия) блокирует строку таблицы, 1228 (третья сессия) ожидает 1223, а 1264 (четвертая сессия) ожидает 1228.
Каждый сеанс держит эксклюзивные (ExclusiveLock) блокировки на номера своих транзакций (transactionid) и виртуальной транзакции (virtualxid)
Вторая сессия (1223) захватила эксклюзивную блокировку строки (RowExclusiveLock) для строки с id = 1.
Третья (1228) и четвертая (1264) сессии также повесили RowExclusiveLock на строку. 
Оставшаяся блокировка ShareLock вызванна тем, что мы пытаемся обновить строку, на которой уже есть RowExclusiveLock.


Выполним коммит во второй сессии, получаем:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/2142a9ab-87cc-4794-b7d2-ae8b0b575561)

Выполним коммит в третьей сессии, получаем:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/468e4eba-05d5-4a9c-9e0e-ad4a0475aa78)

 
Выполним коммит в четвертой сессии, получаем:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/4ffbd74d-3d30-4bee-a51c-6373f6a7dd04)



3.	Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Блокировок в базе нет:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/632f8d22-d34f-46d0-88d1-9e70802e6442)


В первой сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/5046dda1-892a-4d6b-ae27-49855352d079)

 
Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/025a39e0-71d2-4a7d-b45e-49db51d1623e)

 
Во второй сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/bdd7446a-ac26-425e-ae6e-b61823e093cb)

Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/ae615a82-4b1f-4d72-9b72-e62ea93f4e05)

 
В третьей сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/ac8dc342-5542-4947-b103-de98f118ca11)

 
Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/6da922a4-2ec6-4531-94c9-468f3ee1acba)

 
Снова в первой сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/85f1be05-8a32-4a7a-915b-72c1d8aeb149)

 
Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/28f92ada-ecb0-4d77-b9e3-705d044cd6c7)

 
Во второй сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/7e70bcc8-6b62-476e-b6a2-60b63b35ffc0)


Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/24c99c2e-e2a3-4432-aeca-6fe0c7568c0e)


В третьей сессии:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/cc7f1722-49b3-41a0-8dbb-4521864300f7)

 
Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/397752ea-851e-46cd-90de-bd7a5bdf50cd)

 
Выполняем «rollback» в 3 сессии.
Во 2 делаем коммит и смотрим блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/fea49a31-73cf-424e-9cfd-88973403592a)

 
В 1 сессии делаем коммит и смотрим результат:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/38310a3b-31a7-4ec5-aa20-d88026d43ba6)

 
Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/b53d0022-0b78-42b9-9d61-250b05808b28)

 
Смотрим журнал:
tail -n 20 /var/log/postgresql/postgresql-15-main.log


 
> Читая лог, можем разобраться , что произошло ранее.

4.	Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
________________________________________
Задание со звездочкой*
Попробуйте воспроизвести такую ситуацию.

Блокировок в базе нет:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/4a806feb-56c9-47d8-a984-6c5649ff8dff)

 
Создадим таблицу с текстовым полем и заполним случайно сгенерированными данными в размере 1млн строк:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/3ddd27d7-4c90-4505-a081-31a8a5ad5d1e)

 
Начнем в обеих сессиях транзакцию, выполнив команду «begin;».
В 1 сессии выполним заполнение поля «rw_num» номером строки, отсортировав записи в порядке возрастания:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/1c8528fd-0853-495d-9c21-4bee17e9c3e4)


Во второй сессии выполним заполнение поля «rw_num» номером строки, отсортировав записи в порядке убывания:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/64975bf3-eb0c-410d-af87-4c27f50d5d27)


Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/4f7ec077-d106-42b2-a09b-8be2af2e6b0d)

 
Вторая сессия отваливается:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/f27c1b07-f1cf-4d2e-82c4-3890a7badd89)

 
Блокировки:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/54117ce4-a66a-4bc2-a990-527e693f376f)

 
В логе:

![image](https://github.com/blaidex2/Postgres_Homework-6/assets/130083589/b7dd509b-8095-4711-9bac-d0c6bb0e5080)

 




