---
title: 用innobackupex实现加密备份
description:  这是应一个朋友的需求帮他写的加密备份脚本
categories:
 - MySQL
tags: pt-tools
---

### 安装innobackupex

```bash
yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
yum install percona-xtrabackup-24
```

### MySQL创建备份用户

```mysql
GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'xbackupopr'@'%' IDENTIFIED BY 'UFfrBrtyv_cxR50C';
```

### 生成encrypt-key（可不执行，直接用脚本中的key）

```
openssl enc -aes-256-cbc -pass pass:Password -P -md sha1 | grep iv | cut -d'=' -f2 
```

### 脚本

```python
#!/usr/bin/python
#coding:UTF-8

import os,time,sys,commands

innobackupex = '/usr/bin/innobackupex'
# 全备函数
def full_backup(host, port, user, password, full_backup_dir,week_of_dir, backup_log):
    os.system("%s --host=%s --port=%s --user=%s --password=%s --encrypt=AES256 --encrypt-key=CCD0C542F643163CBF4734362FC917C7 --extra-lsndir=%s --no-timestamp %s > %s 2>&1" %(innobackupex,host, port, user, password, week_of_dir,full_backup_dir, backup_log))

# 增备函数
def incr_backup(host, port, user, password, incr_backup_dir, incremental_lsn,extra_lsndir,backup_log):
    os.system("%s --incremental %s  --incremental-lsn=%s --host=%s --port=%s --user=%s --password=%s  --extra-lsndir=%s --encrypt=AES256 --encrypt-key=CCD0C542F643163CBF4734362FC917C7 > %s 2>&1" %(innobackupex,incr_backup_dir,incremental_lsn,host,port,user,password,extra_lsndir,backup_log))

# 主函数
if __name__ == '__main__':
    host = "127.0.0.1"
    port = '3306'
    user = 'xbackupopr'
    password = 'UFfrBrtyv_cxR50C'
    backup_dir = '/data/backup/'
    defaults_file = '/data/my3306/my.cnf'
    wday = time.localtime().tm_wday
    week_of_dir = backup_dir + time.strftime("%U", time.localtime())
    full_backup_dir = week_of_dir + '/full'
    backup_log = "/tmp/backup.log"
    # 增备寻找最新的上一次备份的基准目录
    base_backup_dir = commands.getoutput('find ' + week_of_dir + ' -mindepth 1 -maxdepth 1 -type d -printf "%P\n"  | sort -nr | head -1') 
    # 获取最新的lsn,xtrabackup_checkpoints文件在每次全备或者增备后会被覆盖
    if os.path.exists(week_of_dir + '/xtrabackup_checkpoints'):
        incremental_lsn = commands.getoutput('cat ' + week_of_dir + '/xtrabackup_checkpoints| grep to_lsn | cut -d"=" -f2|sed -e "s/^[ ]*//g"')
        print(incremental_lsn)
    else:
        print('xtrabackup_checkpoints not exists.Maybe full_backup progress not running yet.')
    # 探测mysql实例是否存活，如果存活继续下面的程序执行，如果不存活则直接退出程序
    mysql_stat = commands.getoutput('/bin/netstat -anp|grep ' + port + ' |grep -v unix|wc -l')
    if mysql_stat >= 1:
        print "mysql实例存活，可进行备份操作！"
    else:
        print "mysql实例不存在，备份操作终止！"
        sys.exit()
    # 每周生成一个周备份目录，全备和增备目录都放在此目录下面
    if os.path.exists(week_of_dir):
        print "周备份目录已经生成，可进行相应的全备或者增量备份"
    else:
        print "周备份目录未产生，创建周备份目录..."
        os.makedirs(week_of_dir)
    # 判断是否周日，如果是周日，直接进行全备，如果不是周日，先检查全备是否存在，不存在则进行全备，存在则进行增备
    print "备份开始"
    if wday == 6:
        full_backup(host, port, user, password, full_backup_dir, week_of_dir,backup_log)
    else:
        if os.path.exists(full_backup_dir):
            incr_backup(host, port, user, password, week_of_dir,incremental_lsn,week_of_dir,backup_log)
        else:
            full_backup(host, port, user, password, full_backup_dir,week_of_dir, backup_log)
    print "备份结束，判断备份是否成功"
    try:
        with open("/tmp/backup.log") as f:
            f.seek(-14, 2)
            backup_results = f.readline().strip()
            if backup_results == "completed OK!":
                print "备份成功"
            else:
                print "备份失败"            
    except Error:
        sys.exit()

    # 备份保留4周
    os.system("find %s -mindepth 1 -maxdepth 1 -type d -ctime +28 -exec rm -rf  {} \;" %backup_dir) 
```

### 备份解密

```bash
innobackupex  --decrypt=AES256 --encrypt-key=CCD0C542F643163CBF4734362FC917C7  /data/backup/42/full/
```

