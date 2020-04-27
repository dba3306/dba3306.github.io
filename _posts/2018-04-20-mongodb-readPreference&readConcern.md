---
title: mongodb readPreference 和 readConcern初探
categories: mongodb
description: 介绍readPreference 和 readConcern
tags: mongodb                                                          
---

### 综述

在读取数据的过程中我们需要关注以下两个问题: 

- 从哪里读?
- 什么样的数据可以读?

第一个问题是是由 readPreference 来解决 第二个问题则是由 readConcern 来解决

### 什么是 readPreference

readPreference 决定使用哪一个节点来满足 正在发起的读请求。可选值包括:

- primary:只选择主节点;
- primaryPreferred:优先选择主节点，如果不可用则选择从节点;
- secondary:只选择从节点;
- secondaryPreferred:优先选择从节点， 如果从节点不可用则选择主节点;
- nearest:选择最近的节点;

### readPreference 场景举例

- **primary/primaryPreferred** 用户下订单后马上将用户转到订单详情页,因为此时从节点可能还没复制到新订单;
- **secondary/secondaryPreferred** 用户查询自己下过的订单。查询历史订单对 时效性通常没有太高要求;
- **secondary** 生成报表。报表对时效性要求不高，但资源需求大，可以在从节点 单独处理，避免对线上用户造成影响;
- **nearest**将用户上传的图片分发到全世界，让各地用户能够就近读取。每个地区 的应用选择最近的节点读取数据。

### readPreference 与 Tag

readPreference 只能控制使用一类节点。Tag 则可以将节点选择控制 到一个或几个节点。考虑以下场景:

- 一个5个节点的复制集;
- 3个节点硬件较好，专用于服务线上客户;
- 2个节点硬件较差，专用于生成报表

可以使用 Tag 来达到这样的控制目的:

- 为3个较好的节点打上{purpose:"online"};
- 为2个较差的节点打上{purpose:"analyse"};
- 在线应用读取时指定online，报表读取时指定analyse。

### readPreference 配置

**通过 MongoDB 的连接串参数:**

-  `mongodb://host1:27107,host2:27107,host3:27017/?replicaSet=rs&readPre ference=secondary`

**通过 MongoDB 驱动程序 API:**

- `MongoCollection.withReadPreference(ReadPreferencereadPref)`

**Mongo Shell:** 

- `db.collection.find({}).readPref(“secondary”)`

### readPreference注意事项

- 指定readPreference时也应注意高可用问题。例如将readPreference指定primary，则发生故障转移不存在 primary 期间将没有节点可读。如果业务允许，则应选择 primaryPreferred;
- 使用Tag时也会遇到同样的问题，如果只有一个节点拥有一个特定Tag，则在这个节点失效时将无节点可读。这在有时候是期望的结果，有时候不是。例如:
  - 如果报表使用的节点失效，即使不生成报表，通常也不希望将报表负载转移到其他节点上，此时只有一个节点有报表 Tag 是合理的选择;
  - 如果线上节点失效，通常希望有替代节点，所以应该保持多个节点有同样的Tag;
- Tag有时需要与优先级、选举权综合考虑。例如做报表的节点通常不会希望它成为主节点，则优先级应为 0。

---

### 什么是 readConcern

在 readPreference 选择了指定的节点后，readConcern 决定这个节点上的数据哪些 是可读的，类似于关系数据库的隔离级别。可选值包括:

- available:读取所有可用的数据;
- local:读取所有可用且属于当前分片的数据;
- majority:读取在大多数节点上提交完成的数据;
- linearizable:可线性化读取文档;
- snapshot:读取最近快照中的数据;

### readConcern: local 和 available

在复制集中 local 和 available 是没有区别的。两者的区别主要体现在分片集上。考虑以下场景: 

- 一个chunkx正在从shard1向shard2迁移;
- 整个迁移过程中chunkx中的部分数据会在shard1和shard2中同时存在，但源分片shard1仍然是 chunk x 的负责方:
  - 所有对chunkx的读写操作仍然进入shard1;
  - config中记录的信息chunkx仍然属于shard1;
- 此时如果读shard2，则会体现出local和available的区别:
  - local:只取应该由shard2负责的数据(不包括x);
  - available:shard2上有什么就读什么(包括x);

**注意事项:**

- 虽然看上去总是应该选择local，但毕竟对结果集进行过滤会造成额外消耗。在一些无关紧要的场景(例如统计)下，也可以考虑 available;
- MongoDB<=3.6不支持对从节点使用{readConcern:"local"};
- 从主节点读取数据时默认readConcern是local，从从节点读取数据时默认 readConcern 是 available(向前兼容原因)。

### readConcern: majority 与脏读

如果在一次写操作到达大多数节点前读取了这个写操作，然后因为系统故障该操作回滚了，则发生了脏读问题;

使用 {readConcern: “majority”} 可以有效避免脏读

### readConcern: 如何实现安全的读写分离

考虑如下场景:

- 向主节点写入一条数据;
- 立即从从节点读取这条数据。
- 如何保证自己能够读到刚刚写入的数据?

下述方式有可能读不到刚写入的订单

```json
db.orders.insert({ oid: 101, sku: ”kite", q: 1}) db.orders.find({oid:101}).readPref("secondary")
```

**使用 writeConcern + readConcern majority 来解决**

```json
db.orders.insert({ oid: 101, sku: "kiteboar", q: 1}, {writeConcern:{w: "majority”}}) db.orders.find({oid:101}).readPref("secondary").readConcern("majority")
```

