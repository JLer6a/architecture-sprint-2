# pymongo-api

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```
### В консоли конфигурируем configSrv командами:
docker exec -it configSrv mongosh --port 27017
#### Параметр
rs.initiate(
{
        _id : "configReplSet",
        configsvr: true,
        members: [
        { _id : 0, host : "configSrv:27017" }
        ]
    }
);

### В консоли конфигурируем shard1 командами:
docker exec -it shard1 mongosh --port 27018
#### Параметр
rs.initiate(
    {
        _id : "shard1ReplSet",
        members: [
        { _id : 0, host : "shard1:27018" }
        ]
    }
);

### В консоли конфигурируем shard2 командами:
docker exec -it shard2 mongosh --port 27019
#### Параметр
rs.initiate(
    {
        _id : "shard2ReplSet",
        members: [
        { _id : 0, host : "shard2:27019" }
        ]
    }
);

### В консоли конфигурируем mongos_router командами:
docker exec -it mongos_router mongosh --port 27020
#### Параметр
sh.addShard("shard1ReplSet/shard1:27018");
sh.addShard("shard2ReplSet/shard2:27019");

#### Создаем и заполняем данными

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

#### Проверка (Ответ: 1000)
db.helloDoc.countDocuments()


### Заходим для проверки на шард shard1
docker exec -it shard1 mongosh --port 27018
#### Параметр
use somedb;
db.helloDoc.countDocuments();
#### Ответ: 492


### Заходим для проверки на шард shard2
docker exec -it shard2 mongosh --port 27019
#### Параметр
use somedb;
db.helloDoc.countDocuments();
#### Ответ: 508


### Объеденяем редисы в класстер
docker exec -it redis_1 redis-cli --cluster create 173.17.0.2:6379 173.17.0.3:6379 173.17.0.4:6379 173.17.0.5:6379 173.17.0.11:6379 173.17.0.12:6379 --cluster-replicas 1
#### отвечаем
yes

### Проверяем кластер на мастера и слейв
docker exec -it redis_1 redis-cli cluster nodes


### Проверка на работу. Вызываем приложение
docker exec -it pymongo_api python

#### ВСТАВЛЯЕМ КОД НИЖЕ

import time
import redis
from redis.cluster import RedisCluster

    # Узлы кластера Redis
startup_nodes = [
{"host": "redis_1", "port": "6379"},
{"host": "redis_2", "port": "6379"},
{"host": "redis_3", "port": "6379"},
{"host": "redis_4", "port": "6379"},
{"host": "redis_5", "port": "6379"},
{"host": "redis_6", "port": "6379"}
]

    # Инициализация клиента Redis с поддержкой кластера
r = RedisCluster(startup_nodes=startup_nodes, decode_responses=True, skip_full_coverage_check=True)

    # Тест записи и чтения в Redis
start_time = time.time()
r.set('key', 'value')  # Запись в Redis
write_time = time.time() - start_time
print(f"Time to write to Redis: {write_time:.6f} seconds")

start_time = time.time()
value = r.get('key')  # Чтение из Redis
read_time = time.time() - start_time
print(f"Time to read from Redis: {read_time:.6f} seconds")


#### КАК РЕЗУЛЬТАТ Время чтения при первом обращении:
Time to read from Redis: 0.000402 seconds

#### Время чтения в последующее разы:
Time to write to Redis: 0.000126 seconds
Time to read from Redis: 0.000100 seconds
Time to read from Redis: 0.000136 seconds

### ПОЗВОЛЯЕТ УДАЛИТЬ ВСЕМ VOLUME, котороые могу мешаться и вызывать ошибки
docker-compose down --volumes

##### Описание инфраструктуры

Docker Compose поднимает следующие сервисы:

configSrv – сервер конфигурации для шардинга.

shard1 – первый шард MongoDB.

shard2 – второй шард MongoDB.

mongos_router – роутер MongoDB, через который приложение взаимодействует с БД.

pymongo_api – API-сервис, который подключается к mongos_router.

redis_N - добавил Redis для кеширования