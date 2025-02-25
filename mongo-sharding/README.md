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

### ПОЗВОЛЯЕТ УДАЛИТЬ ВСЕМ VOLUME, котороые могу мешаться и вызывать ошибки
docker-compose down --volumes