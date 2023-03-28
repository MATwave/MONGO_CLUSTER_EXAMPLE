# Пример кластера MongoDB

Пример кластера MongoDB, состоящий из:
- :green_square: двух шардов (наборы реплик по три узла); 
- :orange_square: серверов конфигурации (три узла реплик);
- :blue_square: двух маршрутизаторов Mongos.

По итогу 11 контейнеров Docker (:blue_square::blue_square::orange_square::orange_square::orange_square::green_square::green_square::green_square::green_square::green_square::green_square:), на которых будет работать кластер MongoDB.

![пример](https://user-images.githubusercontent.com/44034959/228374558-09bf5d69-4bf1-4e3f-8ede-0fba4a0bad56.png)

[подробнее о шардировании в доке монги](https://www.mongodb.com/docs/manual/sharding/)

При необходимости можно увеличить его ёмкость, подключая новые шарды.

## Запуск кластера

```docker
docker compose up -d
```

## Конфигурация кластера

1. настроим серверы конфигурации ``:

```docker
docker exec -it mongocfg1 bash -c 'echo "rs.initiate({_id: \"mongors1conf\", configsvr: true, members: [{_id: 0, host: \"mongocfg1\"}, {_id: 1, host: \"mongocfg2\"}, {_id: 2, host: \"mongocfg3\"}]})" | mongosh'
```

<details> 
<summary>подробнее:</summary>

вот это передали в `rs.initiate()`

```json
{
  _id: "mongors1conf",
  configsvr: true,
  members: [
    {
      _id: 0,
      host: "mongocfg1"
    },
    {
      _id: 1,
      host: "mongocfg2"
    },
    {
      _id: 2,
      host: "mongocfg3"
    }
  ]
}
```

</details> 

<details> 
<summary>проверим статус на первом сервере конфигурации:</summary>

```docker
docker exec -it mongocfg1 bash -c 'echo "rs.status()" | mongosh'
```

</details>

2. соберём набор реплик `первого` шарда:

> ```docker
> docker exec -it mongors1n1 bash -c 'echo "rs.initiate({_id: \"mongors1\", members: [{_id: 0, host: \"mongors1n1\"}, {_id: 1, host: \"mongors1n2\"}, {_id: 2, host: \"mongors1n3\"}]})" | mongosh'
> ```
> 
> Теперь ноды шарда знают друг друга. Один из них стал первичным (primary), а два других — вторичными (secondary)
> 
> <details> 
> <summary>подробнее:</summary>
> 
> вот это передали в `rs.initiate()`
> 
> ```json
> {
>   _id: "mongors1",
>   members: [
>     {
>       _id: 0,
>       host: "mongors1n1"
>     },
>     {
>       _id: 1,
>       host: "mongors1n2"
>     },
>     {
>       _id: 2,
>       host: "mongors1n3"
>     }
>   ]
> }
> ```
> 
> </details> 
> 
> <details> 
> <summary>проверим статус реплики:</summary>
> 
> ```docker
> docker exec -it mongors1n1 bash -c 'echo "rs.status()" | mongosh'
> ```
> 
> </details>
> 
> 2.1. познакомим шард с маршрутизаторами:
> 
> ```docker
> docker exec -it mongos1 bash -c 'echo "sh.addShard(\"mongors1/mongors1n1\")" | mongosh'
> ```

3. соберём набор реплик `второго` шарда по аналогии:

> ```docker
> docker exec -it mongors2n1 bash -c 'echo "rs.initiate({_id: \"mongors2\", members: [{_id: 0, host: \"mongors2n1\"}, {_id: 1, host: \"mongors2n2\"}, {_id: 2, host: \"mongors2n3\"}]})" | mongosh'
> ```
> 
> <details> 
> <summary>подробнее:</summary>
> 
> вот это передали в `rs.initiate()`
> 
> ```json
> {
>   _id: "mongors2",
>   members: [
>     {
>       _id: 0,
>       host: "mongors2n1"
>     },
>     {
>       _id: 1,
>       host: "mongors2n2"
>     },
>     {
>       _id: 2,
>       host: "mongors2n3"
>     }
>   ]
> }
> ```
> 
> </details> 
> 
> 3.1. добавим их в кластер:
> 
> ```docker
> docker exec -it mongos1 bash -c 'echo "sh.addShard(\"mongors2/mongors2n1\")" | mongosh'
> ```

Теперь маршрутизаторы-интерфейсы кластера для клиентов знают о существовании шардов. Можно проверить статус с помощью
команды, запущенной на первом маршрутизаторе:

```docker
docker exec -it mongos1 bash -c 'echo "sh.status()" | mongosh'
```
---
кластер полностью сконфигурирован, но у вас пока нет ни одной базы данных. 

4. Создадим тестовую БД:

> ```docker
> docker exec -it mongors1n1 bash -c 'echo "use someDb" | mongosh'
> ```
> 
> 4.1. включим шардирование тестовой базы:
> 
> ```docker
> docker exec -it mongos1 bash -c 'echo "sh.enableSharding(\"someDb\")" | mongosh'
> ```
---
5. Теперь у вас есть база данных с подключённым шардированием в кластере MongoDB. Пришло время создать тестовую
   коллекцию:

> ```docker
> docker exec -it mongos1 bash -c 'echo "db.createCollection(\"someDb.someCollection\")" | mongosh'
> ```
> 
> 5.1. Эта коллекция ещё не шардируется. Ключ шардирования должен выбираться тщательно, поскольку он потом будет
>    использоваться для распределения документов в кластере. 
> 
> Настроим шардирование по полю someField:
> 
> ```docker
> docker exec -it mongos1 bash -c 'echo "sh.shardCollection(\"someDb.someCollection\", {\"someField\": \"hashed\"})" | mongosh'
> ```
