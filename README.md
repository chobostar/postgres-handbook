[WIP] PostgreSQL Handbook
=========================
Handbook по эксплуатации PostgreSQL в production-e. Основано на реальных болях разработчиков.

Прежде, чем вообще использовать БД помните:
*"Behavior is easy, state is hard" (c)*

- [Реально ли нужна база?](#реально-ли-нужна-база)
- [Обязательно ли делать запрос в базу?](#обязательно-ли-делать-запрос-в-базу)
- [Capacity planning](#capacity-planning)
- [Сколько делать размер пула коннектов](#сколько-делать-размер-пула-коннектов)
- [Stateless масштабируется проще, чем Stateful](#stateless-масштабируется-проще-чем-stateful)
- [Использование refresh materialized view, может приводит к connect timeout](#использование-refresh-materialized-view-может-приводит-к-connect-timeout)
- [Влияние uuid vs bigint в качестве primary key на performance](#влияние-uuid-vs-bigint-в-качестве-primary-key-на-performance)
- [Влияние synchronous_commit на TPS](#влияние-synchronous_commit-на-tps)
- [Причины idle in transaction и почему это плохо](#-idle-in-transaction----)
- [Не делайте базы коммуналки](#не-делайте-базы-коммуналки)
- [Какие ресурсы нужны под базу](#какие-ресурсы-нужны-под-базу)
- [Грязные трюки (не использовать на продакшне)](#грязные-трюки-не-использовать-на-продакшне)
    - [Запуск postgresql, если больше нет свободного места](#запуск-postgresql-если-больше-нет-свободного-места)
    - [Потестировать отказоустойчивость](#потестировать-отказоустойчивость)
    - [Non-Durable Settings](#non-durable-settings)
- [Материалы](#материалы)

### Реально ли нужна база?
Используя базу сразу подписываетесь на дополнительную и весьма немалую ответственность.
- так ли нужно самое точное и последнее значение?
- обязательно ли хранить логи/историю/аудит и соблюдать целостность для них?
- почему нельзя передавать state не через посредника (postgres), а напрямую?
- можно ли использовать message broker, там где используется shared state?
- справочники можно хардкодить
- данные, которые сохраняются, когда-нибудь читаются? при каких условиях их можно будет удалить?

### Обязательно ли делать запрос в базу?
- часто пишем ненужную информацию в базу (или информацию, которая по своей природе быстро устаревает)
- часто читаем ненужную информацию из базы
- справочники можно кэшировать
- разделять данные на immutable + часто читаемые, и mutable
    - поля для поиска и часто изменяемые в простых типах
    - отдельный json/поля для часто отображаемой информации
    - отдельный json/таблица для редко используемой и не отображаемой информации

### Capacity planning
Пустая строка занимает 24 байта
```
postgres=# select pg_column_size(row());
 pg_column_size 
----------------
             24
(1 row)
```
Считать размеры строк можно так:
```
postgres=# select pg_column_size(row(0::bigint, 't'::boolean, 1::integer));
 pg_column_size 
----------------
             40
(1 row)
```
Тут про [type alignment](https://www.2ndquadrant.com/en/blog/on-rocks-and-sand/)

### Сколько делать размер пула коннектов
- [Закон Литтла](https://en.wikipedia.org/wiki/Little%27s_law)
- [about pool sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

`connections = ((core_count * 2) + effective_spindle_count)`

Если все данные в кэше, то `effective_spindle_count = 0`.

Чтобы тяжелые запросы не забивали весь пул, можно разделять пулы коннектов - для быстрых синхронных задач и медленных асинхронных.

### Stateless масштабируется проще, чем Stateful
Процессор на stateless "дешевле", т.к. поднять такой же сервис рядом можно быстро, а поднять реплику от базы это дорого. Не пытайтесь всю работу отдавать на откуп базе:
- при batch insert, если можно дедупликацию сделать на приложении, то лучше сделать это на приложении
- обязательно ли надо возвращать отсортированные строки?
- вычисление offset-ов внутри базы не бесплатно

### Использование refresh materialized view, может приводит к connect timeout
Актуально для драйверов, которые вычитываю pg_catalog:

1) как работает matview:
https://github.com/postgres/postgres/blob/REL_10_10/src/backend/commands/matview.c#L158-L166
2) здесь pgx:
https://github.com/jackc/pgx/blob/v3.6.0/conn.go#L607-L618

2-ой не может прочитать каталог, если 1-ый держит эксклюзивный лок либо стоит в очереди на взятии лока 

### Влияние uuid vs bigint в качестве primary key на performance
uuid занимает больше места плюс засчет рандомности трогает больше листьев b-tree, что приводит к большим объемам WAL:
https://www.2ndquadrant.com/en/blog/on-the-impact-of-full-page-writes/ 

### Влияние synchronous_commit на TPS
| synchronous_commit | TPS  |
|---|---|
| off | 3937  |
| local | 1984  |
| remote_write  | 1701  |
| on  | 1373  |
| remote_apply  | 1349  |

[pgbench -c4 -j2 -N bench2 on Amazon EC2 VMs (m3.large, Ubuntu, all in same subnet, 1GB shared_buffers)](https://www.enterprisedb.com/blog/cheat-sheet-configuring-streaming-postgres-synchronous-replication)

### Причины idle in transaction и почему это плохо
1. Поход во внешний сервис при открытой транзакции
    - Во-первых, открытая транзакция обходится не бесплатно     
    - Во-вторых, попусту занимается коннект в пуле, пул-воркеров обычно гораздо больше, чем пул коннектов - зависающие транзакции приведут к истощению 
    - В-третьих, закладываемая логика все равно не будет работать честно, т.к. может произойти disconnect, failover - state или message для внешнего сервиса не откатится
2. Вычисления на стороне приложения при открытой транзакции
3. Транзакция ждет ответа пользователя
4. Внутри транзакции происходит несколько round-trip до приложения и обратно

### Не делайте базы коммуналки
1 база == 1 postgres instance

### Какие ресурсы нужны под базу
Исходить например из:
- планируемой нагрузки (`RPS`)
- среднее время транзакции в секундах (`AvgTxTime`)
- соотношение write/read
- объем dataset-а
- среднего объема запросов
- можно ли разделить данные на горячие и холодные?

Сколько потоков нужно -`AvgTxTime * RPS` исходя из этого планируется количество vCPU.

Если весь dataset горячий, то в диск ходить нежелательно - `RAM` > объем dataset.

latency до локального SSD 150-300μs + ping + planning time + execution time - исходя из этого какую часть нагрузки допустимо пускать в диск, и сколько iops примерно нужно.

количество строк на запрос и средний объем запроса - исходить из худшего сценария, когда каждая нужная строка будет находится на отдельной странице.
Т.е. `8kb` * `на количество строк` * `RPS` и прикинуть влезаем ли в лимиты iops + io bandwitch.

### Используейте минимальный необходимый уровень блокировки
- может достаточно `FOR SHARE` ?
- если используются foreign keys может вместо `FOR UPDATE`, использовать `FOR NO KEY UPDATE` ?

[подбирайте соответствующее](https://www.postgresql.org/docs/current/explicit-locking.html#LOCKING-ROWS)


### Грязные трюки (не использовать на продакшне)
#### Запуск postgresql, если больше нет свободного места
```bash
$ iptables -I INPUT -p tcp -m multiport --dport 5432,6432,6532
$ /usr/lib/postgresql/12/bin/pg_controldata /var/lib/postgresql/12/main | grep checkpoint
Latest checkpoint's REDO WAL file:    000000070000000400000014
...
Time of latest checkpoint:            Thu Aug 27 11:40:25 2020
```
удаляем из папки: `/var/lib/postgresql/12/main/pg_wal` все что старше.

Запускаем postgres, удаляем то, что просили удалить, возвращаем iptables.

#### Потестировать отказоустойчивость
При некоторых настройках `vm.overcommit_memory` и `oom_score_adj` может сработать такое:

открываем столько коннектов сколько сможем и выполняем:   
```sql
postgres=# set temp_buffers='1024GB';
SET
postgres=# set work_mem='1024GB';
SET
postgres=# explain analyze select a, max(b), min(c) from generate_series(1,1000000) as a, generate_series(1,100000) as b, generate_series(1,10) as c group by a;
```
должен сработать oom killer

#### Non-Durable Settings
как ускорить postgres, если [данные в нём не нужны](https://www.postgresql.org/docs/current/non-durability.html)
```
fsync = off
synchronous_commit = off
full_page_writes = off
```
плюс [unlogged tables](https://www.postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-UNLOGGED)

### Материалы
- [Как устроить хайлоад на ровном месте](https://www.highload.ru/moscow/2018/abstracts/41819)
- [Борьба с нагрузкой в PostgreSQL, помогает ли репликация в этом?](https://www.highload.ru/spb/2019/abstracts/4890)
- [Postgres Highload Checklist](https://www.highload.ru/spb/2019/abstracts/4433)