---
title: mongodb复制集环境搭建
categories: mongodb
description: 本文介绍了如何快速在mac上部署3节点的mongodb复制集
tags: mongodb
---
> 目标：在mac上部署3个节点的复制集，为后续测试准备环境

### 创建目录：

`mkdir -p /data/mongodb/db{1,2,3}`

### 编辑配置文件：

`cat /data/mongodb/db1/mongod.conf `

```
systemLog:
    destination: file
    path: /data/mongodb/db1/mongod.log
    logAppend: true
storage:
    dbPath: /data/mongodb/db1
net:
    bindIp: 0.0.0.0
    port: 28017
replication:
    replSetName: rs0
processManagement:
    fork: true
```

> 其他节点只需修改path和port即可

### 启动mongod：

`mongod -f /data/db1/mongod.conf`

### 配置复制集：

`mongo --port 28017`

> 方法一

```
> rs.initiate()
> rs.add("HOSTNAME:28018") 
> rs.add("HOSTNAME:28019")
```

> 方法二

`mongo --port 28017`

```
> rs.initiate({
    _id: "rs0",
    members: [
    {_id: 0,host: "localhost:28017" },
    {_id: 1,host: "localhost:28018" },
    {_id: 2,host: "localhost:28019" }
    ]
})
```

### 验证

> MongoDB 主节点进行写入

`mongo localhost:28017`

`> db.test.insert({ a:1 })`

`> db.test.insert({ a:2 });`

> MongoDB 从节点进行读

`mongo localhost:28018`

```
> rs.slaveOk()
> db.test.find()
```

 ![image-20200420160908919](/downloads/image-20200420160908919.png)
