[WIP] PostgreSQL Handbook
=========================
Handbook по эксплуатации PostgreSQL в production-e. Основано на реальных болях разработчиков.

Прежде, чем вообще использовать БД помните:
*"Behavior is easy, state is hard" (c)*

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

### Сколько делать размер пула коннектов
- [Закон Литтла](https://en.wikipedia.org/wiki/Little%27s_law)
- [about pool sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

`connections = ((core_count * 2) + effective_spindle_count)`

Если все данные кэше, то `effective_spindle_count = 0`. 

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

### Материалы
- [Как устроить хайлоад на ровном месте](https://www.highload.ru/moscow/2018/abstracts/41819)
- [Борьба с нагрузкой в PostgreSQL, помогает ли репликация в этом?](https://www.highload.ru/spb/2019/abstracts/4890)
