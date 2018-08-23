---
layout: post
title:  "Oracle sharding database的一些概念 转整理"
date:   2018-03-30 12:00:00 +0800
categories: 文档整理
tags:  文档整理 sharding oracle 数据分片
author: cm
---

* content
{:toc}





### 1 Table family：
有相关关联关系的一组表，如客户表（customers），订单表（order），订单明细表（LineItems）。这些表之间往往有外键约束关系，可以通过如下2种方式建立table family：

1.1 通过CONSTRAINT [FK_name] FOREIGN KEY (FK_column) REFERENCES \[R_table_name]([R_table_column]) **——这种关系可以有级联关系。**

```
SQL> CREATE SHARDED TABLE Customers
  2  (
  3  CustId VARCHAR2(60) NOT NULL,
  4  FirstName VARCHAR2(60),
  5  LastName VARCHAR2(60),
  6  Class VARCHAR2(10),
  7  Geo VARCHAR2(8),
  8  CustProfile VARCHAR2(4000),
  9  Passwd RAW(60),
 10  CONSTRAINT pk_customers PRIMARY KEY (CustId),
 11  CONSTRAINT json_customers CHECK (CustProfile IS JSON)
 12  ) TABLESPACE SET TSP_SET_1
 13  PARTITION BY CONSISTENT HASH (CustId) PARTITIONS AUTO;
 
Table created.
 
SQL> 
SQL> CREATE SHARDED TABLE Orders
  2  (
  3  OrderId INTEGER NOT NULL,
  4  CustId VARCHAR2(60) NOT NULL,
  5  OrderDate TIMESTAMP NOT NULL,
  6  SumTotal NUMBER(19,4),
  7  Status CHAR(4),
  8  constraint pk_orders primary key (CustId, OrderId),
  9  constraint fk_orders_parent foreign key (CustId)
 10  references Customers on delete cascade
 11  ) partition by reference (fk_orders_parent);
 
Table created.
 
SQL> 
SQL> CREATE SEQUENCE Orders_Seq;
 
Sequence created.
 
SQL> CREATE SHARDED TABLE LineItems
  2  (
  3  OrderId INTEGER NOT NULL,
  4  CustId VARCHAR2(60) NOT NULL,
  5  ProductId INTEGER NOT NULL,
  6  Price NUMBER(19,4),
  7  Qty NUMBER,
  8  constraint pk_items primary key (CustId, OrderId, ProductId),
  9  constraint fk_items_parent foreign key (CustId, OrderId)
 10  references Orders on delete cascade
 11  ) partition by reference (fk_items_parent);
 
Table created.
``` 

可以看到上面根表（root table）是customer表，主键是CustId，partition是根据CONSISTENT HASH，对CustId进行分区；
下一级的表是order表，主键是CustId+OrderId，外键是CustId且references Customers表，partition是参考外键；
再下一级表是LineItems表，主键是CustId+OrderId+ProductId，外键是CustId+OrderId，即上一层表达主键，partition是参考外键

1.2 同关键字PARENT来显式的说明父子关系。**——这种关系只有父子一层关系，不能级联。**

```
SQL>  CREATE SHARDED TABLE Customers
 2  ( CustNo NUMBER NOT NULL
 3  , Name VARCHAR2(50)
 4  , Address VARCHAR2(250)
 5  , region VARCHAR2(20)
 6  , class VARCHAR2(3)
 7  , signup DATE
 8  )
 9  PARTITION BY CONSISTENT HASH (CustNo)
 10 TABLESPACE SET ts1
 11 PARTITIONS AUTO
 12 ;
 
SQL> CREATE SHARDED TABLE Orders          
 2  ( OrderNo NUMBER                     
 3  , CustNo NUMBER                      
 4  , OrderDate DATE                     
 5  )                                    
 6  PARENT Customers                     
 7  PARTITION BY CONSISTENT HASH (CustNo)
 8  TABLESPACE SET ts1                   
 9  PARTITIONS AUTO                      
 10 ; 
 
SQL> CREATE SHARDED TABLE LineItems
 2  ( LineNo NUMBER
 3  , OrderNo NUMBER
 4  , CustNo NUMBER
 5  , StockNo NUMBER
 6  , Quantity NUMBER
 7  )
 8  PARENT Customers
 9  PARTITION BY CONSISTENT HASH (CustNo)
 10 TABLESPACE SET ts1
 11 PARTITIONS AUTO
 12 ;
```

注意上面的order表和LineItems表，都是属于同一个父表，即parent customers表。

另外，也注意上面的CustNo字段，在每个表中都是有的。而上面说的第一种的级联关系的table family，可以不在每个表中都存在CustNo字段。

### 2 Sharded Table和Duplicated table：
上面创建在表，都是sharded table，即表的各个分区，可以分布在不同的shard node上。各个shard node上的分区，是不同的。即整个表的内容，是被切割成片，分配在不同的机器上的。
而duplicated table，是整个表达同样内容，在各个机器上是一样的。duplicate table在各个shard node上，是以read only mv的方式呈现：在shardcat中，存在mast table；在各个shard中，存在read only mv。duplicated table的同步：以物化视图的方式同步。
![](https://www.oracleblog.org/wp-content/uploads/2016/05/shard_and_dup.png)


### 3 chunk：
A chunk contains a single partition from each table of a table family. This guarantees that related data from different sharded tables can be moved together.

chunk的概念和table family密不可分。因为family之间的各个表都是有关系的，我们把某个table family的一组分区称作一个chunk。如
customers表中的1号～100万号客户信息在一个分区中；在order表中，也有1号～100万号的客户的order信息，也在一个分区中；另外LineItems表中的1号～100万号客户的明细信息，也在一个分区中，我们希望这些相关的分区，都是在一个shard node中，避免cross shard join。所以，我们把这些在同一个table family内，相关的分区叫做chunk。在进行re-sharding的时候，是以chunk为单位进行移动。因此可以避免cross shard join。

![](https://www.oracleblog.org/wp-content/uploads/2016/05/chunk.png)

另外，虽然我们设计了chunk来避免cross shard join，但是在做查询的时候，还是有可能会查到非table family进行cross shard join，这在设计之初，就应该避免的。如果有cross shard，还不如用duplicated table。

注：chunk的数量在CREATE SHARDCATALOG的指定，如果不指定，默认值是每个shard 120个chunk

### 4 chunk move：
chunk move的条件：
（1）re-sharding发生，即当shard的数量发生改变的时候，会发生chunk move。
注，re-sharding之后，chunk的数虽然平均，但并不连续。如：
原来是2个shard，1～6号chunk在shard 1，7～12号chunk在shard2。加多一个shard后，1～4号chunk在shard 1，7～10号chunk在shard 2，那么5～6，11～12号chunk在shard 3上。即：总是挪已经存在的shard node上的后面部分chunk。
（2）DBA手工发起：move chunk -chunk 7 -source sh2 -target sh1。将chunk 7从shard node sh2上，挪到shard node sh1上。

chunk move的过程：
在chunk migration的时候，chunk大部分时间是online的，但是期间会有几秒钟的时间chunk中的data处于read-only状态。
chunk migration的过程就是综合利用rman增量备份和TTS的过程：
level 0备份源chunk相关的TS，还原到新shard->开始FAN（等待几秒）->将源chunk相关的TS置于read-only->level 1备份还原->chunk up(更新routing table连新shard)->chunk down(更新routing table断开源shard)->结束FAN（等待几秒）->删除原shard上的老chunk

### 5 shardspace：
create tablespace set的时候，指定shardspace。主要是在Composite sharding架构中使用多个shardspace。

```
ADD SHARDSPACE –SHARDSPACE shspace1, shspace2;

ADD SHARD –CONNECT shard1 –SHARDSPACE shspace1;
ADD SHARD –CONNECT shard2 –SHARDSPACE shspace1;
ADD SHARD –CONNECT shard3 –SHARDSPACE shspace1;
ADD SHARD –CONNECT shard4 –SHARDSPACE shspace2;
ADD SHARD –CONNECT shard5 –SHARDSPACE shspace2;
ADD SHARD –CONNECT shard6 –SHARDSPACE shspace2;
ADD SHARD –CONNECT shard7 –SHARDSPACE shspace2;

CREATE TABLESPACE SET tbs1 IN SHARDSPACE shspace1;
CREATE TABLESPACE SET tbs2 IN SHARDSPACE shspace2;
```

### 6 shardgroup：

```
In system-managed and composite sharding, the logical unit of replication is a group of shards called a shardgroup.
```

也就是说，在逻辑上，将一组相同复制属性的shard称作shard group。如有8台主机，其中4台是shard node的primary，另外4台是shard node的standby。那么，我们可以把4台primary定义成一个shardgroup，叫primary_shardgroup,另外4台standby定义成另外一个shardgroup，叫standby_shardgroup。

注，一个shardgroup通常是在一个data center内。

dataguard可以级联，那么我们可以把primary做为一个shardgroup1，第一级的dataguard叫shardgroup2，第二级的dataguard叫shardgroup2.如下：
![](https://oracleblog.org/wp-content/uploads/2016/11/System-Managed-Sharding-with-Data-Guard-Replication.png)

### 7 shardspace:


```
A shardspace is set of shards that store data that corresponds to a range or list of key values.
```

shardspace的概念伴随着综合性分片的分片方法（composite sharding method）出现的；如果是系统管理分片方法（system-managed sharding method），只有一个默认的shardspace。所以只有在composite sharding下，才有可能出现多个shardspace。

在composite sharding下，数据首先根据list或者range，分成若干个shardspace。然后再根据一致性hash进行分片。

注，每个shardspace包含一个或者多个复制shard for HA/DR。

![](https://oracleblog.org/wp-content/uploads/2016/11/shardspace.png)
我们可以按照服务级别来划分shardspace，如硬件好的，作为shardspace_gold,硬件差一些的，划做shardspace_silver:
![](https://www.oracleblog.org/wp-content/uploads/2016/05/comp.png)
或者也可以根据业务需要，按照地域来分不同的shardgroup，如这里的shardgroup cust_america和shardgroup cust_euro
![add-shardgroup](https://oracleblog.org/wp-content/uploads/2016/11/add-shardgroup.png)

### 8 super sharding key


```
Composite sharding enables two levels of sharding - one by list or range and another by consistent hash. This is accomplished by the application providing two keys: a super sharding key and a sharding key.
```
super sharding key也是随着综合性分片的分片方法出现的。由于综合性分片是2层分片，第一层是根据list或者range进行分片，这个shard key，就叫做super_sharding_key。第二层才是根据一致性hash进行分片（基于sharding key）。

9 Composite sharding
综合性分片，提供了2层的分片方法，第一层按照range 或者list，第二层根据一致性hash。

综合性分片一般适用于：
1 在不同的地理位置，使用不同的shardspace：
![comp_shard_1](https://oracleblog.org/wp-content/uploads/2016/11/comp_shard_1.png)

2 在不同的服务器，或者云上面，使用不同的shardspace：
![comp_shard_2](https://oracleblog.org/wp-content/uploads/2016/11/comp_shard_2.png)

### 9 region



> A region in the context of Oracle Sharding represents a data center or multiple data centers that are in close network proximity.

region,地区的概念。可以指一个数据中心，也可以指一个地域。

### 10 shard director



> A shard director is a specific implementation of a global service manager that acts as a regional listener for clients that connect to an SDB. The director maintains a current topology map of the SDB. Based on the sharding key passed during a connection request, the director routes the connections to the appropriate shard.
 
> To achieve high availability, deploy multiple shard directors. In Oracle Database 12c Release 2, you can deploy up to 5 shard directors in a given region.

shard director提供：
a).实时维护SDB的配置信息和可用性。
b).衡量自身所处的region和别的region间的网络延时
c).接受来自客户端到SDB的连接，你可以把它当作region listener。
d).管理global services
e).实现连接负载均衡。

注：每个region需要至少一个shard director，可以最多有五个shard director。

### 11 sharding 方式：
在gdsctl create shardcatalog时指定，
System-Managed Sharding：partitioning by consistent hash.主要作用是打散数据。
![](https://www.oracleblog.org/wp-content/uploads/2016/05/system-managed.png)

Composite Sharding：create multiple shardgroups for different subsets of data in a table partitioned by consistent hash. 分层，可以用不同的shardspace，不同的tablespace set，使用不同的硬件。高级的customer用更好的硬件资源。
![](https://www.oracleblog.org/wp-content/uploads/2016/05/comp.png)

Using Subpartitions with Sharding：all of the subpartitioning methods provided by Oracle Database are also supported for sharding. 更细粒度的分区。


### 12 如何部署sharding：
简单来说，就是在gdsctl中运行如下命令。关于详细内容，我会另外再写一篇，讲讲sharding的部署和架构中的注意点。
![](https://www.oracleblog.org/wp-content/uploads/2016/05/subpartition.png)

```
CREATE SHARDCATALOG
ADD GSM; START GSM (create and start shard directors)
CREATE SHARD (for each shard)
DEPLOY
```


