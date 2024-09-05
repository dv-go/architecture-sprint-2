# 1. Запуск MongoDB и приложения
docker compose up -d

# 2. Инициализация конфигурационного сервера
docker exec -it configSrv mongosh --port 27019

> rs.initiate(
  {
    _id : "config_server",
    configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27019" }
    ]
  }
);
> exit();

# 3. Инициализация shard1
docker exec -it shard1 mongosh --port 27018

> rs.initiate(
  {
    _id : "shard1",
    members: [
      { _id : 0, host : "shard1:27018" }
    ]
  }
);
> exit();

# 4. Инициализация shard2
docker exec -it shard2 mongosh --port 27020

> rs.initiate(
  {
    _id : "shard2",
    members: [
      { _id : 1, host : "shard2:27020" }
    ]
  }
);
> exit();

# Удаление старых данных из томов Docker, если они мешают инициализации шардов

docker-compose down
docker volume rm mongo-sharding_shard1-data
docker volume rm mongo-sharding_shard2-data
docker volume rm mongo-sharding_config-data
docker-compose up -d


# 5. Инициализация роутера и наполнение тестовыми данными
docker exec -it mongos_router mongosh --port 27017

> sh.addShard("shard1/shard1:27018");
> sh.addShard("shard2/shard2:27020");
> sh.enableSharding("somedb");
> sh.shardCollection("somedb.helloDoc", { "name" : "hashed" });
> use somedb;
> for (var i = 0; i < 1000; i++) db.helloDoc.insert({ age: i, name: "ly" + i });
> db.helloDoc.countDocuments();
> exit();

# Получится результат — 1000 документов.

# 5. Проверка данных на shard1
docker exec -it shard1 mongosh --port 27018

> use somedb;
> db.helloDoc.countDocuments();
> exit();

# Получится результат — 492 документа.

# Проверка данных на shard2
docker exec -it shard2 mongosh --port 27020

> use somedb;
> db.helloDoc.countDocuments();
> exit();

# Получится результат — 508 документов.