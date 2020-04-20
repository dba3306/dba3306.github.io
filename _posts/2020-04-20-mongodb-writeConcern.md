---
title: mongodb writeConcern初探
categories: mongodb
description: 介绍writeConcern的概念以及注意事项
tags: mongodb                                                          
---

### 什么是writeConcern

writeConcern 决定一个写操作落到多少个节点上才算成功。取值包括：

- 0 ：发起写操作，不关心是否成功。
- 1~集群最大节点数 ：写操作需要复制到指定节点数才算成功
- majority：写操作需要复制到多数节点才算成功

发起写操作的程序将阻塞直到满足上诉条件。

> writeConcern类似MySQL的半同步复制`--rpl-semi-sync-master-wait-for-slave-count`

这里还要提到另一个概念：Journal 日志，跟MySQL的redo log类似，也是为了做crash safe。为了保证每次写数据都落盘，可以指定`j: true` 

注：3.2之前只要primary节点落盘就确认ok，3.2之后需要w:N都确认才ok
### writeConcern功能验证

#### w: "majority"

```json
rs0:PRIMARY> db.test.insert( {count: 1}, {writeConcern: {w: "majority"}})
WriteResult({ "nInserted" : 1 })
```

这是推荐的写法，mongodb会自己判断只要写入多数节点即返回成功

#### w: 节点数

```json
rs0:PRIMARY> db.test.insert( {count: 1}, {writeConcern: {w: 3 }})
WriteResult({ "nInserted" : 1 })
```

可以看到正常情况下（secondary节点没延迟）也很快返回成功了。

#### w＞节点数

```json
rs0:PRIMARY> db.test.insert( {count: 1}, {writeConcern: {w: 4}})
WriteResult({
        "nInserted" : 1,
        "writeConcernError" : {
                "code" : 100,
                "codeName" : "UnsatisfiableWriteConcern",
                "errmsg" : "Not enough data-bearing nodes"
        }
})
```

这里虽然报错了，但实际还是写入成功了。

#### secondary节点有延迟的情况下，观察现象

首先我们模拟延迟场景，在primary节点执行，模拟secondary 2节点延迟10秒

```json
conf=rs.conf() 
conf.members[2].slaveDelay = 10
conf.members[2].priority = 0 
rs.reconfig(conf)
```

观察复制延迟下的写入，以及timeout参数

`db.test.insert( {count: 1}, {writeConcern: {w: 3}})`

```json
rs0:PRIMARY> db.test.find()
{ "_id" : ObjectId("5e9d7bf6d4eb493af10599c9"), "count" : 1 }
{ "_id" : ObjectId("5e9d7cf4d4eb493af10599ca"), "count" : 1 }
{ "_id" : ObjectId("5e9d7e2ad4eb493af10599cb"), "count" : 1 }
rs0:PRIMARY> db.test.insert( {count: 1}, {writeConcern: {w: 3}})
# 这里会等待10s
WriteResult({ "nInserted" : 1 })
rs0:PRIMARY> db.test.find()
{ "_id" : ObjectId("5e9d7bf6d4eb493af10599c9"), "count" : 1 }
{ "_id" : ObjectId("5e9d7cf4d4eb493af10599ca"), "count" : 1 }
{ "_id" : ObjectId("5e9d7e2ad4eb493af10599cb"), "count" : 1 }
{ "_id" : ObjectId("5e9d7f9ad4eb493af10599cc"), "count" : 1 }
```
`db.test.insert( {count: 1}, {writeConcern: {w: 3, wtimeout:3000 }})`

```json
rs0:PRIMARY> db.test.find()
{ "_id" : ObjectId("5e9d7bf6d4eb493af10599c9"), "count" : 1 }
{ "_id" : ObjectId("5e9d7cf4d4eb493af10599ca"), "count" : 1 }
{ "_id" : ObjectId("5e9d7e2ad4eb493af10599cb"), "count" : 1 }
{ "_id" : ObjectId("5e9d7f9ad4eb493af10599cc"), "count" : 1 }
rs0:PRIMARY> db.test.insert( {count: 1}, {writeConcern: {w: 3, wtimeout:3000 }})
# 这里等3s后报错，实际命令敲下的一瞬间已写入到primary和没延迟的secondary，延迟节点需要等10s后落盘。
WriteResult({
        "nInserted" : 1,
        "writeConcernError" : {
                "code" : 64,
                "codeName" : "WriteConcernFailed",
                "errmsg" : "waiting for replication timed out",
                "errInfo" : {
                        "wtimeout" : true
                }
        }
})
rs0:PRIMARY> db.test.find()
{ "_id" : ObjectId("5e9d7bf6d4eb493af10599c9"), "count" : 1 }
{ "_id" : ObjectId("5e9d7cf4d4eb493af10599ca"), "count" : 1 }
{ "_id" : ObjectId("5e9d7e2ad4eb493af10599cb"), "count" : 1 }
{ "_id" : ObjectId("5e9d7f9ad4eb493af10599cc"), "count" : 1 }
{ "_id" : ObjectId("5e9d800ed4eb493af10599cd"), "count" : 1 }
```

### 总结

- 虽然多于半数的writeConcern都是安全的，但通常只会设置majority，因为这是等待写入延迟时间最短的选择;
- 不要设置writeConcern等于总节点数，因为一旦有一个节点故障，所有写操作都将失败;
- writeConcern虽然会增加写操作延迟时间，但并不会显著增加集群压力，因此无论是否等待，写操作最终都会复制到所有节点上。设置 writeConcern 只是让写操作等待复制后再返回而已;
- 应对重要数据应用{w:“majority”}，普通数据可以应用{w:1}以确保最佳性能。

