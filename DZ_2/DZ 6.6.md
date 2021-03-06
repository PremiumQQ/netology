# Домашнее задание к занятию "6.6. Troubleshooting"


## Обязательная задача 1

Перед выполнением задания ознакомьтесь с документацией по администрированию MongoDB.

Пользователь (разработчик) написал в канал поддержки, что у него уже 3 минуты происходит CRUD операция в MongoDB и её нужно прервать.

Вы как инженер поддержки решили произвести данную операцию:

- напишите список операций, которые вы будете производить для остановки запроса пользователя
```
1) Определю id запроса:
db.currentOp({ "active" : true, "secs_running" : { "$gt" : 180 }})
2) Удалю его по полученному id:
db.killOp(???)
```
- предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB
```
1. Использовать метод maxTimeMS() для установки предела исполнения по времени операций 
2. Выполнить explain и посмотреть что не так с запросом. Попробовать оптимизировать, если это возможно: 
денормализовать данные, добавить/удалить индексы, настроить шардинг и т.д.
```


## Обязательная задача 2

Перед выполнением задания познакомьтесь с документацией по Redis latency troobleshooting.

Вы запустили инстанс Redis для использования совместно с сервисом, который использует механизм TTL. Причем отношение количества записанных key-value значений к количеству истёкших значений есть величина постоянная и увеличивается пропорционально количеству реплик сервиса.

При масштабировании сервиса до N реплик вы увидели, что:

- сначала рост отношения записанных значений к истекшим
- Redis блокирует операции записи

Как вы думаете, в чем может быть проблема?

```
Я думаю проблема кроется в этом куске заголовка:
Latency generated by expires
Redis evict expired keys in two ways:

One lazy way expires a key when it is requested by a command, but it is found to be already expired.
One active way expires a few keys every 100 milliseconds.
The active expiring is designed to be adaptive. An expire cycle is started every 100 milliseconds (10 times per second), and will do the following:

Sample ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP keys, evicting all the keys already expired.
If the more than 25% of the keys were found expired, repeat.
Given that ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP is set to 20 by default, and the process is performed ten times per second, usually just 200 keys per second are actively expired. This is enough to clean the DB fast enough even when already expired keys are not accessed for a long time, so that the lazy algorithm does not help. At the same time expiring just 200 keys per second has no effects in the latency a Redis instance.

However the algorithm is adaptive and will loop if it finds more than 25% of keys already expired in the set of sampled keys. But given that we run the algorithm ten times per second, this means that the unlucky event of more than 25% of the keys in our random sample are expiring at least in the same second.

Basically this means that if the database has many many keys expiring in the same second, and these make up at least 25% of the current population of keys with an expire set, Redis can block in order to get the percentage of keys already expired below 25%.

Т.е. на русский язык, память занята истекшими ключами, но они ещё не удалены. Redis заблокировался, чтобы убрать из DB удаленные ключи и снизить их до 25%. 
Redis - однопоточное приложение и пока он не выполнит выше описанное действие, все операции будут блокироваться. 
```

## Обязательная задача 3

Перед выполнением задания познакомьтесь с документацией по Common Mysql errors.

Вы подняли базу данных MySQL для использования в гис-системе. При росте количества записей, в таблицах базы, пользователи начали жаловаться на ошибки вида:

```yaml
InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '
```
Как вы думаете, почему это начало происходить и как локализовать проблему?
```
Может быть проблема в сети, из-за того что база разрослась. Из-за слишком объемного запроса, размер сообщения/запроса превышает размер буфера и клиент отваливается по таймауту.
Нужно увеличить размер пакета.
```

Какие пути решения данной проблемы вы можете предложить?
```
Решить проблемы с сетью, шардирование таблиц, добавление системных ресурсов.
```

## Обязательная задача 4

Перед выполнением задания ознакомтесь со статьей Common PostgreSQL errors из блога Percona.

Вы решили перевести гис-систему из задачи 3 на PostgreSQL, так как прочитали в документации, что эта СУБД работает с большим объемом данных лучше, чем MySQL.

После запуска пользователи начали жаловаться, что СУБД время от времени становится недоступной. В dmesg вы видите, что:

postmaster invoked oom-killer

Как вы думаете, что происходит?
```
Происходит из-за недостатка ресурсов (памяти).
```

Как бы вы решили данную проблему?

```
Надо проверить настройки памяти согласно оборудованию. Возможно надо выделить больше памяти на процесс Postgres.
Если это возможно, добавить ресурсов (оперативной памяти).

Настроить выделение памяти в конфиге (sysctl.conf):
vm_overcommit_memory
0:  установка переменной в 0, при этом ядро будет решать, делать ли избыточную фиксацию или нет. 
1:  установка переменной в 1 означает, что ядро всегда будет перегружаться. 
2: установка переменной в 2 означает, что ядро не должно выделять больше памяти, чем overcommit_ratio
В нашем случае используем 2.
overcommit_ratio (если я правильно понял, по умолчанию использует 60% выделенной оперативной памяти, используется значение 1)

Так же можно настроить конфиг postgres (postgres.conf):
max_connections: количество одновременных сеансов
work_mem : максимальный объем памяти, который будет использоваться для промежуточных результатов, таких как хеш-таблицы, и для сортировки
shared_buffers объем памяти, выделенный для «закрепленного» буферного пространства.
effective_cache_size объем памяти, предположительно используемый буферами LRU операционной системы.
random_page_cost : оценка относительной стоимости дисков ищет.
```