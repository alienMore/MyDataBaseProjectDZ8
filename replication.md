<h1 align="center">ДЗ 8</h1>
<h1 align="center">Replication в postgreSQL</h1>

---
```
Создал сеть postgres_network для обмена между контейнерами
docker network create postgres_network
docker network inspect postgres_network

[
    {
        "Name": "postgres_network",
        "Id": "3fe175aace370304e151f8b53cf5c626f1bffbc030f5bf0640f3b814bf4a794f",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Containers": {
            "a2c3753d620e2a5ca41cd35a36932b5c945648637535b82688fad8ff91b66c01": {
                "Name": "PostgresMaster",
                "EndpointID": "a0d5986ef8fd76edbe630fc96cf209519230d6f3c8a2a9f18e14c17058236863",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "d31f97e36f2a2d26991a3ad56c07080d522643a0f44ee90b1f85d8a7f0c71e4c": {
                "Name": "PostgresSlave",
                "EndpointID": "7d3223fe22fd6b3c3d8a0a5db6f7f8c3c732b6982021949106d1cab8e67cebb5",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {}
    }
]
```

1. ### Потоковая репликация(физическая репликация) - streaming исспользуя слот репликации ###

- **Slave**

postgresql.conf
```
важные настройки
(wal_level = replica
max_wal_size = 1GB
max_slot_wal_keep_size = 1000  #Нужно чтобы успеть запустить Slave и не потерять WAL-файлы мастера (чтобы Slave, поспевал за Master)
max_replication_slots = 10
recovery_min_apply_delay = 5min #задает отстование от мастера на 5 min, есть токлько на slave)
```
```
listen_addresses = '*'
max_connections = 100                           # (change requires restart)
shared_buffers = 128MB                          # min 128kB
dynamic_shared_memory_type = posix              # the default is the first option
wal_level = replica                             # minimal, replica, or logical
max_wal_size = 1GB
min_wal_size = 80MB
max_wal_senders = 8                             # max number of walsender processes
max_slot_wal_keep_size = 1000                   # in megabytes; -1 disables
max_replication_slots = 10                      # max number of replication slots
recovery_min_apply_delay = 5min                 # minimum delay for applying changes during recovery
log_timezone = 'Etc/UTC'
datestyle = 'iso, mdy'
timezone = 'Etc/UTC'
lc_messages = 'en_US.utf8'                      # locale for system error message
lc_monetary = 'en_US.utf8'                      # locale for monetary formatting
lc_numeric = 'en_US.utf8'                       # locale for number formatting
lc_time = 'en_US.utf8'                          # locale for time formatting
default_text_search_config = 'pg_catalog.english'
```
pg_hba.conf
```
добавил строку
(host    replication     replicator      172.18.0.0/16           md5)
```
```
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust
host    replication     replicator      172.18.0.0/16           md5
host all all all md5
```
standby.signal (нулевого размера)

postgresql.auto.conf
```
listen_addresses = '*'
primary_conninfo = 'user=replicator password=replicator channel_binding=prefer host=172.18.0.2 port=5432 sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'slot1'

Файл наполняется автоматически в результате выполнения команды:
- если слот еще не создан, создаст автоматически
pg_basebackup -h <ip мастер сервера> -D /var/lib/postgresql/data_basebackup -U replicator -P -v -R -X stream -C -S slot1

- если слот создан ранее на мастере командой ( SELECT pg_create_physical_replication_slot('slot1'); удалить SELECT pg_drop_replication_slot('slot1'); )
pg_basebackup -h <ip мастер сервера> -D /var/lib/postgresql/data_basebackup -U replicator --slot slot1 -X stream

Полный бэкап мастера будет в каталоге /var/lib/postgresql/data_basebackup, затем останавливаем базу слейва, удаляем все из рабочего каталога базы.
Содержимое каталога /var/lib/postgresql/data_basebackup скопировать в рабочий каталог базы слейва и выполнить настройки конфигов из пунка настройки слейва
```



- **Master**

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'replicator';
ALTER SYSTEM SET listen_addresses TO '*';
```
```
Добавил в pg_hba.conf
host    replication     replicator      172.18.0.0/16           md5
```
```
Включение синхронной репликации
Синхронная репликация дает возможность зафиксировать транзакцию в master и slave одновременно Подтверждает успешность транзакции, когда все изменения в транзакции были перенесены на все сервера.
Чтобы включить синхронную репликацию:
для параметра synchronous_commit также должно быть установлено значение on (по умолчанию)
И необходимо установить для параметра synchronous_standby_names непустое значение
ALTER SYSTEM SET synchronous_standby_names TO  '*';
```

2. ### Логическая репликация ###

- **Master**

```sql
И на мастере и на слейве создадим таблицу
CREATE TABLE data_from_elasticsearch
(date_error_appeared timestamp,
warning_type text NOT NULL,
host text NOT NULL,
date_inser_in_table timestamp DEFAULT CURRENT_TIMESTAMP);
--На Master Создаем публикацию и добавляем в эту публикацию таблицу data_from_elasticsearch
CREATE PUBLICATION my_publication;
ALTER PUBLICATION my_publication ADD TABLE data_from_elasticsearch;
```

```
в postgresql.conf
...
wal_level = logical
...
```

- **Slave**

```sql
--Добавляем подписчика
CREATE SUBSCRIPTION my_subscription CONNECTION 'host=172.18.0.2 port=5432 password=secret user=postgres dbname=stage' PUBLICATION my_publication;
```

| Database   | ver |
| -----      | --- |
| PostgreSQL | 13.3|
