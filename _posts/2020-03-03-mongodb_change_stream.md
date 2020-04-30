---
title: mongodb change stream 功能介绍
categories: mongodb
description: change stream 类似于MySQL的触发器，但是又比其更强大
tags: mongodb                                                          

---

### 什么是 Change Stream

Change Stream 是 MongoDB 用于实现变更追踪的解决方案，类似于关系数据库的触发器，但原理不完全相同:

|          | Change Stream        | 触发器       |
| -------- | -------------------- | ------------ |
| 触发方式 | 异步                 | 同步         |
| 触发位置 | 应用回调事件         | 数据库触发器 |
| 触发次数 | 每个订阅事件的客户端 | 1次          |
| 故障恢复 | 从上次断点重新触发   | 事务回滚     |



### Change Stream 的实现原理

Change Stream 是基于 oplog 实现的。它在 oplog 上开启一个 tailable cursor 来追踪所有复制集上的变更操作，最终调用应用中定义的回调函数。

被追踪的变更事件主要包括:

- insert/update/delete:插入、更新、删除;
- drop:集合被删除;
- rename:集合被重命名;
- dropDatabase:数据库被删除;
- invalidate:drop/rename/dropDatabase将导致invalidate被触发， 并关闭 change stream;

### Change Stream 与可重复读

Change Stream 只推送已经在大多数节点上提交的变更操作。即“可重复读”的变更。 这个验证是通过` {readConcern: "majority"} `实现的。因此:

- 未开启majorityreadConcern的集群无法使用ChangeStream;
- 当集群无法满足`{w:"majority"}`时，不会触发ChangeStream(例如PSA架构中的 S 因故障宕机)。

### Change Stream 变更过滤

如果只对某些类型的变更事件感兴趣，可以使用使用聚合管道的过滤步骤过滤事件。
例如:

```js
var cs = db.collection.watch([{
    $match: {
        operationType: {
            $in: ['insert', 'delete']
} }
}])
```

### Change Stream 示例

开启2个窗口:

在窗口1执行监听：

```js
rs0:PRIMARY> db.test.watch([],{maxAwaitTimeMS: 30000}).pretty()
{
        "_id" : {
                "_data" : "825EA7A400000000012B022C0100296E5A1004124DEBF980784545948639CB6F3A779146645F696400645EA7A400604B3753349DDA300004"
        },
        "operationType" : "insert",
        "clusterTime" : Timestamp(1588044800, 1),
        "fullDocument" : {
                "_id" : ObjectId("5ea7a400604b3753349dda30"),
                "x" : 1
        },
        "ns" : {
                "db" : "test",
                "coll" : "test"
        },
        "documentKey" : {
                "_id" : ObjectId("5ea7a400604b3753349dda30")
        }
}
```

窗口2 执行插入：

```js
rs0:PRIMARY> db.test.insert({x:1})
WriteResult({ "nInserted" : 1 })
```

此刻窗口1会同步如下：

```js
{
        "_id" : {
                "_data" : "825EA7A400000000012B022C0100296E5A1004124DEBF980784545948639CB6F3A779146645F696400645EA7A400604B3753349DDA300004"
        },
        "operationType" : "insert",
        "clusterTime" : Timestamp(1588044800, 1),
        "fullDocument" : {
                "_id" : ObjectId("5ea7a400604b3753349dda30"),
                "x" : 1
        },
        "ns" : {
                "db" : "test",
                "coll" : "test"
        },
        "documentKey" : {
                "_id" : ObjectId("5ea7a400604b3753349dda30")
        }
}
```

这个功能跟gh-ost是不是有点神似？^_^

### Change Stream 故障恢复

想要从上次中断的地方继续获取变更流，只需要保留上次变更通知中的 _id 即可。_

例如:
`var cs = db.collection.watch([], {resumeAfter: <_id>}) `即可从上一条通知中断处继续获取后续的变更通知。

### Change Stream 使用场景

- **跨集群的变更复制:**在源集群中订阅ChangeStream，一旦得到任何变更立即写入目标集群。
- **微服务联动:**当一个微服务变更数据库时，其他微服务得到通知并做出相应的变 更。
- 其他任何需要系统联动的场景。

### 注意事项

- ChangeStream依赖于oplog，因此中断时间不可超过oplog回收的最大时间窗;
- 在执行update操作时，如果只更新了部分数据，那么ChangeStream通知的也是增量部分；
- 删除数据时通知的仅是删除数据的_id。