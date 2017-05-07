---
layout:     post
title:      "Greenplum扩展数据节点及表数据重分布"
subtitle:   "gpdb expand"
date:       2017-05-07
author:     "Garry fan"
header-img: "img/post-bg-2015.jpg"
tags:
    - Greenplum
    - gpexpand
---
## Greenplum扩展数据节点及表数据重分布

Greenplum支持数据segment的动态扩展(包括增加新的host节点和segment实例，以及在已有节点上扩展segment实例)。Segment的动态扩展和数据重分布都是通过gpexpand的工具来完成的，本文以增加新的host节点为例，来说明如何使用gpexpand。

### 环境说明

已经安装并正常运行的greenplum集群：
* mdw: master节点，仅运行master实例
* sdw1: segment节点，上面运行了2个segment实例

待加入集群的host:
* sdw2: 新host待加入集群

### 数据准备
为了更好的演示表数据重分布，我们在GP创建一个测试数据库testdb以及testdis表，并插入1000条数据
```
gpadmin@mdw $ psql -c CREATE DATABASE testdb
gpadmin@mdw $ psql -f create_data.sql
```
create_data.sql
```SQL
CREATE TABLE testdis (id int, name varchar(20)) DISTRIBUTED BY (id);

CREATE OR REPLACE FUNCTION creatData()
  RETURNS boolean AS
$BODY$
DECLARE
  i int;
BEGIN
  i :=1;
  FOR i IN 1..10000 LOOP
    INSERT INTO testdis(id,name) VALUES(i,'CEIC');
  END LOOP;

  IF i > 0 THEN
    RETURN true;
  ELSE
    RETURN false;
  END IF;
END;
$BODY$
LANGUAGE 'plpgsql' VOLATILE;

ALTER FUNCTION creatData() OWNER TO gpadmin;
SELECT * FROM creatData();
```

### 添加数据节点

#### 新节点安装greenplum

安装依赖的rpm包并设置好时间同步

```
root@mdw # ssh sdw2
root@sdw2 # yum install libevent perl sigar uuid apr apr-util ntpdate
root@sdw2 # ntpdate mdw && hwclock -w
root@sdw2 # exit
```

> 注意：

> 1、在sdw2上按照greenplum的官方文档执行os的参数优化

> 2、在sdw2上的/etc/hosts文件里配置好mdw和sdw1对应的IP

安装greenplum

```
root@mdw # source /usr/local/greenplum-db/greenplum_path.sh
root@mdw # echo sdw2 > hostfile
root@mdw # gpseginstall -f hostfile -u gpadmin -p gpadmin
```

#### 新节点加入集群

生成gpexpand配置文件，expand命令交互时，加入的主机host填写sdw2，其他的默认即可。
完成后，会在当前目录下生成一个gpexpand_inputfile_YYYYMMDD_XXXXXX的文件

```
root@mdw # su - gpadmin
gpadmin@mdw $ export PGDATABASE=testdb
gpadmin@mdw $ gpexpand
```

添加节点到集群

```
gpadmin@mdw $ gpexpand -i gpexpand_inputfile_20170507_000151 -D testdb
```

> 注意: 节点扩展后，如果之前集群已经开启了gpperfmon，那么master节点也会控制新节点启动gpsmon服务

### 表数据重分布
在进行表数据重分布之前，我们先查看一下当前表数据的分布情况
```SQL
SELECT gp_segment_id, count(*) FROM testdis GROUP BY 1;

 gp_segment_id | count 
---------------+-------
             1 |  4999
             0 |  5001
(2 rows)

```

#### 数据重分布

```
gpadmin@mdw $ gpexpand -d 00:10:00
```

> 注意:当前的测试场景，测试数据量很少，所以duration设置的时间为10分钟，具体情况要根据数据量来指定duration的长度，详细信息请参考gpexpand的帮助文档

查看一下重分布后的数据分布情况
```SQL
SELECT gp_segment_id, count(*) FROM testdis;

 gp_segment_id | count 
---------------+-------
             1 |  2499
             0 |  2500
             3 |  2500
             2 |  2501
(4 rows)

```

### 节点添加失败的处理

当我们在执行添加节点命令失败时，需要回滚操作，具体步骤如下

```
gpadmin@mdw $ gpstart -m
gpadmin@mdw $ gpexpand -r -D testdb
```

> 注意: expand失败后，只能通过master only的模式启动，执行回滚rollback完成后，才能通过正常模式启动greenplum

