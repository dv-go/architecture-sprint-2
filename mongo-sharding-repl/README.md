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

# 3. Инициализация shard1 и трех реплик в shard1

docker exec -it shard1 mongosh --port 27018

> rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1:27018" },
    { _id: 1, host: "shard1_replica1:27018" },
    { _id: 2, host: "shard1_replica2:27018" },
    { _id: 3, host: "shard1_replica3:27018" }
  ]
});
> exit();

# 4. Инициализация shard2 и трех реплик в shard1

docker exec -it shard2 mongosh --port 27020

> rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2:27020" },
    { _id: 1, host: "shard2_replica1:27020" },
    { _id: 2, host: "shard2_replica2:27020" },
    { _id: 3, host: "shard2_replica3:27020" }
  ]
});
> exit();

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

# 6. Проверка данных на shard1

docker exec -it shard1 mongosh --port 27018

> use somedb;
> db.helloDoc.countDocuments();
> exit();

# Получится результат — 492 документа.

# 7. Проверка данных на реплике shard1_replica1 шарда shard1

docker exec -it shard1_replica1 mongosh --port 27018

> use somedb;
> db.helloDoc.countDocuments();
> exit();

# Получится результат — 492 документа - такой же как в основной базе shard1

# 8. Проверка данных на shard2

docker exec -it shard2 mongosh --port 27020

> use somedb;
> db.helloDoc.countDocuments();
> exit();

# Получится результат — 508 документов.

# 9. Проверка данных на реплике shard2_replica1 шарда shard2

docker exec -it shard2_replica1 mongosh --port 27020

> use somedb;
> db.helloDoc.countDocuments();
> exit();