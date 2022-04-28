# 关于 GreatSQL
---

GreatSQL开源数据库专注于提升MGR可靠性及性能，支持InnoDB并行查询等特性，是适用于金融级应用的国内自主MySQL版本；可以作为MySQL或Percona Server的可选替换，用于线上生产环境；且完全免费并兼容MySQL或Percona Server。

## 版本特性
---
GreatSQL除了提升MGR性能及可靠性，还引入InnoDB事务锁优化及并行查询优化等特性，以及众多BUG修复。
选用GreatSQl主要有以下几点优势：

- 提升MGR模式下的大事务并发性能及稳定性
- 改进MGR的GC及流控算法，以及减少每次发送数据量，避免性能抖动
- 在MGR集群AFTER模式下，解决了节点加入集群时容易出错的问题
- 在MGR集群AFTER模式下，强一致性采用多数派原则，以适应网络分区的场景
- 当MGR节点崩溃时，能更快发现节点异常状态，有效减少切主和异常节点的等待时间
- 优化InnoDB事务锁机制，在高并发场景中有效提升事务并发性能至少10%以上
- 实现InnoDB并行查询机制，极大提升聚合查询效率，TPC-H测试中，最高可提升40多倍，平均提升15倍。特别适用于周期性数据汇总报表之类的SAP、财务统计等业务
- 修复了MGR模式下可能导致数据丢失、性能抖动、节点加入恢复极慢等多个缺陷或BUG

## 注意事项
---
运行GreatSQL可能需要依赖jemalloc库（推荐5.2.1+版本），因此请先安装上
```
yum -y install jemalloc jemalloc-devel
```
也可以把自行安装的lib库so文件路径加到系统配置文件中，例如：
```
[root@greatdb]# cat /etc/ld.so.conf
/usr/local/lib64/
```
而后执行下面的操作加载libjemalloc库，并确认是否已存在
```
[root@greatdb]# ldconfig

[root@greatdb]# ldconfig -p | grep libjemalloc
        libjemalloc.so.2 (libc6,x86-64) => /usr/lib64/libjemalloc.so.2
        libjemalloc.so (libc6,x86-64) => /usr/lib64/libjemalloc.so
```

## 安装GreatSQL
执行下面的命令安装GreatSQL
```
#首先，查找GreatSQL
$ yum search GreatSQL
...
greatsql-client.x86_64 : GreatSQL - Client
greatsql-devel.x86_64 : GreatSQL - Development header files and libraries
greatsql-server.x86_64 : GreatSQL: Open source database that can be used to replace MySQL or Percona Server.
greatsql-shared.x86_64 : GreatSQL - Shared libraries

#然后安装
$ yum install -y greatsql-client greatsql-devel greatsql-server greatsql-shared
```

安装完成后，GreatSQL会自行完成初始化，可以再检查是否已加入系统服务或已启动：
```
$ systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
...
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 1137698 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 1137732 (mysqld)
   Status: "Server is operational"
    Tasks: 39 (limit: 149064)
   Memory: 336.7M
   CGroup: /system.slice/mysqld.service
           └─1137732 /usr/sbin/mysqld
...
```

## my.cnf参考

RPM方式安装后的GreatSQL默认配置不是太合理，建议参考下面这份my.cnf文档：

- [my.cnf for GreatSQL 8.0.25](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/my.cnf-example-greatsql-8.0.25-15)

调整文档中关于`datadir`目录配置等相关选项，默认 `datadir=/var/lib/mysql` 通常都会改掉，例如替换成 `datadir=/data/GreatSQL`，修改完后保存退出，
替换原来的 `/etc/my.cnf`，然后重启GreatSQL，会重新进行初始化。
```
# 新建 /data/GreatSQL 空目录，并修改目录所有者
$ mkdir -p /data/GreatSQL
$ chown -R mysql:mysql /data/GreatSQL

# 重启mysqld服务，即自行完成重新初始化
$ systemctl restart mysqld
```

## 登入GreatSQL
首次登入GreatSQL前，需要先找到初始化时随机生成的root密码：
```
$ grep root /data/GreatSQL/error.log
[Note] [MY-010454] [Server] A temporary password is generated for root@localhost: dt_)MtExl594
```

其中的 **dt_)MtExl594** 就是初始化时随机生成的密码，在登入GreatSQL时输入该密码：
```
$ mysql -uroot -p'dt_)MtExl594'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 8.0.25-15

Copyright (c) 2021-2021 GreatDB Software Co., Ltd
Copyright (c) 2009-2021 Percona LLC and/or its affiliates
Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> \s
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql>
```

首次登入立刻提醒该密码已过期，需要修改，执行类似下面的命令修改即可：
```
mysql> ALTER USER USER() IDENTIFIED BY 'GreatSQL-8025~%';
Query OK, 0 rows affected (0.02 sec)
```
之后就可以用这个新密码再次登入GreatSQL了。

## 创建新用户、测试库&表，及写入数据
修改完root密码后，应尽快创建普通用户，用于数据库的日常使用，减少超级用户root的使用频率，避免误操作意外删除重要数据。
```
#创建一个新用户GreatSQL，只允许从192.168.0.0/16网络连入，密码是 GreatSQL-2022
mysql> CREATE USER GreatSQL@'192.168.0.0/16' IDENTIFIED BY 'GreatSQL-2022';
Query OK, 0 rows affected (0.06 sec)

#创建一个新的用户库，并对GreatSQL用户授予读写权限
mysql> CREATE DATABASE GreatSQL;
Query OK, 1 row affected (0.03 sec)

mysql> GRANT ALL ON GreatSQL.* TO GreatSQL@'192.168.0.0/16';
Query OK, 0 rows affected (0.03 sec)
```

切换到普通用户GreatSQL登入，创建测试表，写入数据：
```
$ mysql -h192.168.1.10 -uGreatSQL -p'GreatSQL-2022'
...
# 切换到GreatSQL数据库下
mysql> use GreatSQL;
Database changed

# 创建新表
mysql> CREATE TABLE t1(id INT PRIMARY KEY);
Query OK, 0 rows affected (0.07 sec)

# 写入测试数据
mysql> INSERT INTO t1 SELECT RAND()*1024;
Query OK, 1 row affected (0.05 sec)
Records: 1  Duplicates: 0  Warnings: 0

# 查询数据
mysql> SELECT * FROM t1;
+-----+
| id  |
+-----+
| 203 |
+-----+
1 row in set (0.00 sec)
```
成功。

## 版本历史
---
### GreatSQL 8.0
- [GreatSQL 更新说明 8.0.25(2021-8-26)](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/relnotes/changes-greatsql-8-0-25-20210820.md)


## 更多使用文档
---
- [GreatSQL MGR FAQ](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/GreatSQL-FAQ.md)
- [在Linux下源码编译安装GreatSQL](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/build-greatsql-with-source.md)
- [利用Ansible安装GreatSQL并构建MGR集群](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/install-greatsql-with-ansible.md)
- [在Docker中部署GreatSQL并构建MGR集群](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/install-greatsql-with-docker.md)
- [MGR优化配置参考](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/mgr-best-options-ref.md)
- [InnoDB并行查询优化参考](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/innodb-parallel-execute.md)
- [利用GreatSQL部署MGR集群](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/using-greatsql-to-build-mgr-and-node-manage.md)
- [MySQL InnoDB Cluster+GreatSQL部署MGR集群](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/mysql-innodb-cluster-with-greatsql.md)
- [利用systemd管理MySQL单机多实例](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/docs/build-multi-instance-with-systemd.md)

## 专栏文章
- [深入浅出MGR专栏文章](https://gitee.com/GreatSQL/GreatSQL-Doc/blob/master/deep-dive-mgr)，深入浅出MGR相关知识点、运维管理实操，配合「实战MGR」视频内容食用更佳。

## 相关资源
- [GreatSQL-Docker](https://gitee.com/GreatSQL/GreatSQL-Docker)，在Docker中运行GreatSQL。
- [GreatSQL-Ansible](https://gitee.com/GreatSQL/GreatSQL-Ansible)，利用ansible一键安装GreatSQL并完成MGR集群部署。

## 问题反馈
---
- [问题反馈 gitee](https://gitee.com/GreatSQL/GreatSQL-Doc/issues)


## 联系我们
---

扫码关注微信公众号

![输入图片说明](https://images.gitee.com/uploads/images/2021/0802/141935_2ea2c196_8779455.jpeg "greatsql社区-wx-qrcode-0.5m.jpg")