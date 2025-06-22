[WIP] PostgreSQL Handbook
=========================
Handbook по эксплуатации PostgreSQL в production-e. Основано на реальных болях разработчиков.

Прежде, чем вообще использовать БД помните:
*"Behavior is easy, state is hard" (c)*

- [Реально ли нужна база?](#реально-ли-нужна-база)
- [Сделайте так, чтобы администратор к DB был не нужен](#сделайте-так-чтобы-администратор-к-DB-был-не-нужен)
- [Обязательно ли делать запрос в базу?](#обязательно-ли-делать-запрос-в-базу)
- [Capacity planning](#capacity-planning)
- [Сколько делать размер пула коннектов](#сколько-делать-размер-пула-коннектов)
- [Stateless масштабируется проще, чем Stateful](#stateless-масштабируется-проще-чем-stateful)
- [Использование refresh materialized view, может приводит к connect timeout](#использование-refresh-materialized-view-может-приводит-к-connect-timeout)
- [Влияние uuid vs bigint в качестве primary key на performance](#влияние-uuid-vs-bigint-в-качестве-primary-key-на-performance)
- [Влияние synchronous_commit на TPS](#влияние-synchronous_commit-на-tps)
- [Причины idle in transaction и почему это плохо](#причины-idle-in-transaction-и-почему-это-плохо)
- [Не делайте базы коммуналки](#не-делайте-базы-коммуналки)
- [Какие ресурсы нужны под базу](#какие-ресурсы-нужны-под-базу)
- [Используейте минимальный необходимый уровень блокировки](#используейте-минимальный-необходимый-уровень-блокировки)
- [Стремитесь разделять read-only запросы и read-write](#стремитесь-разделять-read-only-запросы-и-read-write)
- [Советы как эффективно использовать JSONB](#советы-как-эффективно-использовать-jsonb)
    - [Отдельная колонка для Primary key](#отдельная-колонка-для-primary-key)
    - [Threshold деградации latency (TOAST Storage)](#threshold-деградации-latency--toast-storage-)
    - [Избегайте слишком вложенные JSON-ы](#избегайте-слишком-вложенные-json-ы)
    - [Вытаскивайте из JSON часто меняющиеся или читаемые поля](#вытаскивайте-из-json-часто-меняющиеся-или-читаемые-поля)
    - [LZ4 компрессия](#lz4-компрессия)
- [Грязные трюки (не использовать на продакшне)](#грязные-трюки-не-использовать-на-продакшне)
    - [Запуск postgresql, если больше нет свободного места](#запуск-postgresql-если-больше-нет-свободного-места)
    - [Потестировать отказоустойчивость](#потестировать-отказоустойчивость)
    - [Non-Durable Settings](#non-durable-settings)
- [Материалы](#материалы)

### Реально ли нужна база?
Используя базу сразу подписываетесь на дополнительную и весьма немалую ответственность.
- так ли нужно самое точное и последнее значение?
- обязательно ли хранить логи/историю/аудит и соблюдать целостность для них?
- почему нельзя передавать state не через посредника (postgres), а напрямую? (JWT, подписанные параметры)
- можно ли использовать message broker, там где используется shared state?
- справочники можно хардкодить
- данные, которые сохраняются, когда-нибудь читаются? при каких условиях их можно будет удалить?

### Сделайте так, чтобы администратор к DB был не нужен
Есть проблемы коммуникации при эксплуатации:
- Разработчик знает намерения кода, как оно должно работать, какие данные хранятся и характер их потребления
- Администратор знает, как код работает по факту и что при этом происходит с инфраструктурой

То как должно работать != как работает по факту. Возникает конфликт. Можно учиться коммуницировать, а можно сделать так, чтобы этой коммуникации вообще не требовалось.
На этапе предоставления базы для разработчиков подумайте:
- все ли метрики и дашборды есть, как понятнее их интерпретировать
- предоставьте алгоритм действий, обучение, как проводить траблшутинг
- упрощайте, иногда лучше не предоставлять что-то сложное клиенту, чем допустить, чтобы он выстрелил этим себе в ногу
- понимает ли разработчик, как устроена инфраструктура и его технические ограничения

Каждый раз после того, как потушили очередной пожар, думайте:
- что не хватило разработчику, чтобы он самостоятельно разобрался в инциденте и решил ее
- где можно улучшить процесс или ограничить функционал, чтобы это не повторялось

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

Не используйте uuid в виде текст, есть [тип uuid](https://stackoverflow.com/a/33837267) на 16 байт.

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

### Стремитесь разделять read-only запросы и read-write

Это история не только про масштабирование чтения, но и про повышение доступности.

У high-available кластера PostgreSQL может быть только 1 мастер (который обрабатывает read-write) и N реплик (которые обрабатывают read-only).

Обеспечить доступность единственного хоста (мастера) сложнее, чем обеспечить доступность набора хостов (реплик). Не все API endpoints приложения выполняют операции записи.
Направляя read-only трафик на реплики, вы повышаете доступность кластера для операций чтения, а значит и общую доступность приложения.

Учитывайте replication lag при проектировании логики приложения - данные на репликах могут отставать от мастера.

### Советы как эффективно использовать JSONB
#### Отдельная колонка для Primary key
|Do|Don't|
|---|---|
|CREATE TABLE qq (jsonb)<br> (id, {…}::jsonb)|CREATE TABLE qq (jsonb)<br> ({id,…}::jsonb)|
#### Threshold деградации latency (TOAST Storage)
По возможности держите размер tuple с JSON <= 2000 bytes. Иначе оно будет ["тоститься"](https://www.postgresql.org/docs/current/storage-toast.html#STORAGE-TOAST-ONDISK)
Это дает значительный penalty по производительности.
#### Избегайте слишком вложенные JSON-ы
Так плохо: `{"obj": {"obj": {"obj": {"obj": {"obj": {"key": 14, "long_str": "a"}}}}}}`
#### Вытаскивайте из JSON часто меняющиеся или читаемые поля
| Do                                                                                                                                                                      |Don't|
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|
| CREATE TABLE accounts<br>(id, number, status, sum, {…}::jsonb)<br><br>SELECT number FROM accounts WHERE id = 123;<br><br>UPDATE accounts SET sum = 10000 WHERE id = 123 |CREATE TABLE customer (jsonb)<br>(id, {number, status, sum…}::jsonb)<br><br>SELECT js->>'number' FROM accounts WHERE id = 123;<br><br>UPDATE accounts SET js = jsonb_set(js, '{number}', '4444', true)<br>WHERE id = 123|

#### LZ4 компрессия
LZ4 компрессия для TOAST данных обеспечивает [лучший баланс между скоростью компрессии/декомпрессии и размером](https://www.tigerdata.com/blog/optimizing-postgresql-performance-compression-pglz-vs-lz4).
Настройка:
```
default_toast_compression = 'lz4'
```
или
```sql
CREATE TABLE lz4_example(id int, lz4_column text COMPRESSION lz4)
```

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

### Что почитать?
- https://www.interdb.jp/pg/
- PostgreSQL 17 изнутри - https://postgrespro.ru/education/books/internals
- Авторитетный и пополняемый контент:
  - https://www.cybertec-postgresql.com/en/blog/
  - https://www.depesz.com/
  - https://www.crunchydata.com/blog

### Материалы
- [Как устроить хайлоад на ровном месте](https://www.highload.ru/moscow/2018/abstracts/41819)
- [Борьба с нагрузкой в PostgreSQL, помогает ли репликация в этом?](https://www.highload.ru/spb/2019/abstracts/4890)
- [Postgres Highload Checklist](https://www.highload.ru/spb/2019/abstracts/4433)
- [Understanding JSONB Perforamance](http://www.sai.msu.su/~megera/postgres/talks/jsonb-pgconfnyc-2021.pdf)

### Мои статьи на тему PG
- 2022-02 - [Вредные советы для postgres](https://medium.com/@Kirill_P/%D1%8F-%D1%81%D0%BE%D0%B1%D0%B8%D1%80%D0%B0%D1%8E%D1%81%D1%8C-%D1%81%D0%BA%D0%BE%D1%80%D0%BE-%D1%83%D0%B2%D0%BE%D0%BB%D0%B8%D1%82%D1%8C%D1%81%D1%8F-%D0%BA%D0%B0%D0%BA-%D0%BC%D0%BD%D0%B5-%D0%BF%D0%BE%D0%B4%D0%B3%D0%B0%D0%B4%D0%B8%D1%82%D1%8C-%D1%81%D0%B2%D0%BE%D0%B8%D0%BC-%D0%BA%D0%BE%D0%BB%D0%BB%D0%B5%D0%B3%D0%B0%D0%BC-%D1%87%D0%B0%D1%81%D1%82%D1%8C-1-854b94223b0)
- 2022-08 - [Про сеть и postgres](https://medium.com/@Kirill_P/network-issues-of-some-postgres-clients-48fcaeb2ce1d)
- 2022-09 - [OOM и postgres](https://medium.com/@Kirill_P/oom-guard-for-containerized-postgres-79fec16b9ef0)
- 2023-05 - [Про грабли Read Replicas](https://medium.com/@Kirill_P/postgresql-read-replicas-pitfalls-a361150d2564)
- 2025-06 - [Про postgres-operator-ы](https://medium.com/@Kirill_P/%D0%BC%D0%BE%D0%B9-wish-list-%D0%BA-k8s-postgres-operator-%D0%B0%D0%BC-7a32bbcfcf49)

