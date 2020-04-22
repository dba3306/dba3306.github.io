### 用dbdeployer 部署3个单节点
`dbdeployer deploy multiple 5.7 -n 3`

### 查看节点状态
```
dbdeployer sandboxes --full-info
.------------------.----------.---------.-----------.----------------------.--------.-------.--------.
|       name       |   type   | version |   host    |         port         | flavor | nodes | locked |
+------------------+----------+---------+-----------+----------------------+--------+-------+--------+
| multi_msb_5_7_29 | multiple | 5.7.29  | 127.0.0.1 | [24630 24631 24632 ] | mysql  |     3 |        |
'------------------'----------'---------'-----------'----------------------'--------'-------'--------'
```

```shell
dbdeployer global status
# Running "status_all" on multi_msb_5_7_29
MULTIPLE  /Users/qianyin/sandboxes/multi_msb_5_7_29
node1 : node1 on  -  port       24630 (24630)
node2 : node2 on  -  port       24631 (24631)
node3 : node3 on  -  port       24632 (24632)
```

### 下载MySQL-Shell

wget https://dev.mysql.com/get/Downloads/MySQL-Shell/mysql-shell-8.0.19-macos10.15-x86-64bit.tar.gz

解压后进入mysql shell 目录
bin目录下就3个文件

```bash
-rwxr-xr-x@ 1 qianyin  staff    633256 12 17 09:42 mysql-secret-store-keychain*
-rwxr-xr-x@ 1 qianyin  staff   7957420 12 17 09:42 mysql-secret-store-login-path*
-rwxr-xr-x@ 1 qianyin  staff  31067132 12 17 09:42 mysqlsh*
```

我们运行mysqlsh
可以先打\help 看下帮助提示
可以看到支持的模块如下：

```bash
 - dba    Used for InnoDB cluster administration.
 - mysql  Support for connecting to MySQL servers using the classic MySQL
          protocol.
 - mysqlx Used to work with X Protocol sessions using the MySQL X DevAPI.
 - os     Gives access to functions which allow to interact with the operating
          system.
 - shell  Gives access to general purpose functions and properties.
 - sys    Gives access to system specific parameters.
 - util   Global object that groups miscellaneous tools like upgrade checker
          and JSON import.
```

部署集群我们需要用到dba功能，再查看dba的帮助信息

`\? dba`

### 正式配置集群

#### 检查并配置3个数据库实例

```shell
mysql-js> \connect root@localhost:24630
 MySQL  localhost:24630 ssl  JS > dba.configureLocalInstance()
Configuring local MySQL instance listening at port 24630 for use in an InnoDB cluster...

This instance reports its own address as node-1:24630

ERROR: User 'root' can only connect from 'localhost'. New account(s) with proper source address specification to allow remote connection from all instances must be created to manage the cluster.

1) Create remotely usable account for 'root' with same grants and password
2) Create a new admin account for InnoDB cluster with minimal required grants
3) Ignore and continue
4) Cancel

Please select an option [1]: 
Please provide a source address filter for the account (e.g: 192.168.% or % etc) or leave empty and press Enter to cancel.
Account Host: %

NOTE: Some configuration options need to be fixed:
+----------------------------------+---------------+----------------+------------------------------------------------+
| Variable                         | Current Value | Required Value | Note                                           |
+----------------------------------+---------------+----------------+------------------------------------------------+
| binlog_checksum                  | CRC32         | NONE           | Update the server variable and the config file |
| enforce_gtid_consistency         | OFF           | ON             | Update the config file and restart the server  |
| gtid_mode                        | OFF           | ON             | Update the config file and restart the server  |
| log_slave_updates                | OFF           | ON             | Update the config file and restart the server  |
| master_info_repository           | FILE          | TABLE          | Update the config file and restart the server  |
| relay_log_info_repository        | FILE          | TABLE          | Update the config file and restart the server  |
| transaction_write_set_extraction | OFF           | XXHASH64       | Update the config file and restart the server  |
+----------------------------------+---------------+----------------+------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server: an option file is required.

Detecting the configuration file...
Default file not found at the standard locations.
Please specify the path to the MySQL configuration file: /Users/qianyin/sandboxes/multi_msb_5_7_29/node1/my.sandbox.cnf
Do you want to perform the required configuration changes? [y/n]: y

Cluster admin user 'root'@'%' created.
Configuring instance...
The instance 'node-1:24630' was configured to be used in an InnoDB cluster.
NOTE: MySQL server needs to be restarted for configuration changes to take effect.
```

依次修改另外2个节点

查看my.cnf 会发现在文件末尾添加了如下配置：

```
binlog_checksum = NONE
enforce_gtid_consistency = ON
gtid_mode = ON
log_slave_updates = ON
master_info_repository = TABLE
relay_log_info_repository = TABLE
transaction_write_set_extraction = XXHASH64
```

配置完后，检查下

```shell
MySQL  localhost:24630 ssl  JS > dba.checkInstanceConfiguration('root@localhost:24631')
Validating local MySQL instance listening at port 24631 for use in an InnoDB cluster...

This instance reports its own address as node-2:24631

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...
Instance configuration is compatible with InnoDB cluster

The instance 'node-2:24631' is valid to be used in an InnoDB cluster.

{
    "status": "ok"
}

```

3个节点都显示"status": "ok" 即可。

#### 创建集群

```shell
 MySQL  localhost:24630 ssl  JS > var cluster = dba.createCluster('testCluster',{'localAddress':'localhost:25630'})
A new InnoDB cluster will be created on instance 'localhost:24630'.

Disabling super_read_only mode on instance 'node-1:24630'.
Validating instance configuration at localhost:24630...

This instance reports its own address as node-1:24630

Instance configuration is suitable.
WARNING: Instance 'node-1:24630' cannot persist Group Replication configuration since MySQL version 5.7.29 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the <Dba>.configureLocalInstance() command locally to persist the changes.
Creating InnoDB cluster 'testCluster' on 'node-1:24630'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.



Adding Instances to an InnoDB Cluster
```

**注意：先配置/etc/hosts **

```
127.0.0.1 node-1
127.0.0.1 node-2
127.0.0.1 node-3
```

否则报错：

```shell
 MySQL  127.0.0.1:24630 ssl  JS > cluster.addInstance('root@localhost:24631',{'localAddress':'localhost:25631'})
Cluster.addInstance: Unknown MySQL server host 'node-1' (0) (MySQL Error 2005)
```

#### 添加集群节点

```shell
 MySQL  localhost:24630 ssl  JS > cluster.addInstance('root@localhost:24631',{'localAddress':'localhost:25631'})

NOTE: The target instance 'node-2:24631' has not been pre-provisioned (GTID set is empty). The Shell is unable to decide whether incremental state recovery can correctly provision it.
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'node-2:24631' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.


Please select a recovery method [I]ncremental recovery/[A]bort (default Incremental recovery): 
Validating instance configuration at localhost:24631...

This instance reports its own address as node-2:24631

Instance configuration is suitable.
WARNING: Instance 'node-2:24631' cannot persist Group Replication configuration since MySQL version 5.7.29 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the <Dba>.configureLocalInstance() command locally to persist the changes.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
State recovery already finished for 'node-2:24631'

WARNING: Instance 'node-1:24630' cannot persist configuration since MySQL version 5.7.29 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the <Dba>.configureLocalInstance() command locally to persist the changes.
The instance 'localhost:24631' was successfully added to the cluster.


     MySQL  localhost:24630 ssl  JS > cluster.addInstance('root@localhost:24632',{'localAddress':'localhost:25632'})

NOTE: The target instance 'node-3:24632' has not been pre-provisioned (GTID set is empty). The Shell is unable to decide whether incremental state recovery can correctly provision it.
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'node-3:24632' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.


Please select a recovery method [I]ncremental recovery/[A]bort (default Incremental recovery): 
Validating instance configuration at localhost:24632...

This instance reports its own address as node-3:24632

Instance configuration is suitable.
WARNING: Instance 'node-3:24632' cannot persist Group Replication configuration since MySQL version 5.7.29 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the <Dba>.configureLocalInstance() command locally to persist the changes.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
State recovery already finished for 'node-3:24632'

WARNING: Instance 'node-1:24630' cannot persist configuration since MySQL version 5.7.29 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the <Dba>.configureLocalInstance() command locally to persist the changes.
WARNING: Instance 'node-2:24631' cannot persist configuration since MySQL version 5.7.29 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the <Dba>.configureLocalInstance() command locally to persist the changes.
The instance 'localhost:24632' was successfully added to the cluster.
```

#### 查看cluster 状态

```shell
 MySQL  localhost:24630 ssl  JS > cluster.status()
{
    "clusterName": "testCluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "node-1:24630", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "node-1:24630": {
                "address": "node-1:24630", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "node-2:24631": {
                "address": "node-2:24631", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "node-3:24632": {
                "address": "node-3:24632", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "node-1:24630"
```

#### 安装MySQL router

待补充

### 模拟单节点重启

关闭节点

```shell
./node2/stop
dbdeployer global status
# Running "status_all" on multi_msb_5_7_29
MULTIPLE  /Users/qianyin/sandboxes/multi_msb_5_7_29
node1 : node1 on  -  port       24630 (24630)
node2 : node2 off  -  port      24630 (24631)
node3 : node3 on  -  port       24632 (24632)
```

集群状态

```shell
 MySQL  localhost:6447 ssl  JS > dba.getCluster().status()
{
    "clusterName": "testCluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "node-1:24630", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures. 1 member is not active", 
        "topology": {
            "node-1:24630": {
                "address": "node-1:24630", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "node-2:24631": {
                "address": "node-2:24631", 
                "mode": "n/a", 
                "readReplicas": {}, 
                "role": "HA", 
                "shellConnectError": "MySQL Error 2003 (HY000): Can't connect to MySQL server on 'node-2' (61)", 
                "status": "(MISSING)"
            }, 
            "node-3:24632": {
                "address": "node-3:24632", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "node-1:24630"
}
```

启动节点

```shell
> ./node2/start
.. sandbox server started
> dbdeployer global status
# Running "status_all" on multi_msb_5_7_29
MULTIPLE  /Users/qianyin/sandboxes/multi_msb_5_7_29
node1 : node1 on  -  port       24630 (24630)
node2 : node2 on  -  port       24631 (24631)
node3 : node3 on  -  port       24632 (24632)
```

查看集群状态

```shell
 MySQL  localhost:6447 ssl  JS > dba.getCluster().status()
{
    "clusterName": "testCluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "node-1:24630", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures. 1 member is not active", 
        "topology": {
            "node-1:24630": {
                "address": "node-1:24630", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "node-2:24631": {
                "address": "node-2:24631", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "(MISSING)"
            }, 
            "node-3:24632": {
                "address": "node-3:24632", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "node-1:24630"
```

修复

```shell
cluster.removeInstance('root@localhost:24631',{force: true})  #这一步可能会失败，多尝试几次
cluster.addInstance('root@localhost:24631',{'localAddress':'localhost:25631'})
```

### 常用运维命令

- 组复制状态
  - `select * from performance_schema.global_status where variable_name like '%group%';`
- 组复制成员
  - `select * from performance_schema.replication_group_members;`

### 所有节点重启，脑裂场景参考如下

https://jeremyxu2010.github.io/2019/05/mysql-innodb-cluster%E5%AE%9E%E6%88%98/