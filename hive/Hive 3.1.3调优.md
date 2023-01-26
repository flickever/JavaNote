# Hive 3.1.3调优

## 1. Yarn&MR调参

### 1.1 Yarn调参

**yarn.nodemanager.resource.memory-mb**

该参数的含义是，一个NodeManager节点分配给Container使用的内存。该参数的配置，取决于NodeManager所在节点的总内存容量和该节点运行的其他服务的数量。

考虑上述因素，此处可将该参数设置为64G

```xml
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>65536</value>
</property>
```



**yarn.nodemanager.resource.cpu-vcores**

该参数的含义是，一个NodeManager节点分配给Container使用的CPU核数。该参数的配置，同样取决于NodeManager所在节点的总CPU核数和该节点运行的其他服务。

考虑上述因素，此处可将该参数设置为16

核数一般是 内存/4，如这里就是64 / 4 = 16

```xml
<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>16</value>
</property>
```



**yarn.scheduler.maximum-allocation-mb**

该参数的含义是，单个Container能够使用的最大内存

```xml
<property>
    <name>yarn.scheduler.maximum-allocation-mb</name>
    <value>16384</value>
</property>
```



**yarn.scheduler.minimum-allocation-mb**

该参数的含义是，单个Container能够使用的最小内存

```xml
<property>
    <name>yarn.scheduler.minimum-allocation-mb</name>
    <value>512</value>
</property>
```





### 1.2 MapReduce资源配置

**mapreduce.map.memory.mb**

该参数的含义是，单个Map Task申请的container容器内存大小，其默认值为1024。该值不能超出yarn.scheduler.maximum-allocation-mb和yarn.scheduler.minimum-allocation-mb规定的范围。

该参数需要根据不同的计算任务单独进行配置，在hive中，可直接使用如下方式为每个SQL语句单独进行配置

```hive
set  mapreduce.map.memory.mb=2048;
```



**mapreduce.map.cpu.vcores**

该参数的含义是，单个Map Task申请的container容器cpu核数，其默认值为1。该值一般无需调整



**mapreduce.reduce.memory.mb**

该参数的含义是，单个Reduce Task申请的container容器内存大小，其默认值为1024。该值同样不能超出yarn.scheduler.maximum-allocation-mb和yarn.scheduler.minimum-allocation-mb规定的范围。

该参数需要根据不同的计算任务单独进行配置，在hive中，可直接使用如下方式为每个SQL语句单独进行配置：

```hive
set  mapreduce.reduce.memory.mb=2048;
```



**mapreduce.reduce.cpu.vcores**   

该参数的含义是，单个Reduce Task申请的container容器cpu核数，其默认值为1。该值一般无需调整



## 2. Explain查看执行计划

Explain呈现的执行计划，由一系列Stage组成，这一系列Stage具有依赖关系，每个Stage对应一个MapReduce Job，或者一个文件系统操作等。

若某个Stage对应的一个MapReduce Job，其Map端和Reduce端的计算逻辑分别由Map Operator Tree和Reduce Operator Tree进行描述，Operator Tree由一系列的Operator组成，一个Operator代表在Map或Reduce阶段的一个单一的逻辑操作，例如TableScan Operator，Select Operator，Join Operator等

常见的Operator及其作用如下：

- TableScan：表扫描操作，通常map端第一个操作肯定是表扫描操作
- Select Operator：选取操作
- Group By Operator：分组聚合操作
- Reduce Output Operator：输出到 reduce 操作
- Filter Operator：过滤操作
- Join Operator：join 操作
- File Output Operator：文件输出操作
- Fetch Operator 客户端获取数据操作



### 2.1 基本语法

```hive
EXPLAIN [FORMATTED | EXTENDED | DEPENDENCY] query-sql
```

- FORMATTED：将执行计划以JSON字符串的形式输出
- EXTENDED：输出执行计划中的额外信息，通常是读写的文件名等信息
- DEPENDENCY：输出执行计划读取的表及分区



### 2.2 插件



## 3. 分组聚合优化

### 3.1 说明

Hive中未经优化的分组聚合，是通过一个MapReduce Job实现的。Map端负责读取数据，并按照分组字段分区，通过Shuffle，将数据发往Reduce端，各组数据在Reduce端完成最终的聚合运算。

Hive对分组聚合的优化主要围绕着减少Shuffle数据量进行，具体做法是map-side聚合。所谓map-side聚合，就是在map端维护一个hash table，利用其完成部分的聚合，然后将部分聚合的结果，按照分组字段分区，发送至reduce端，完成最终的聚合。map-side聚合能有效减少shuffle的数据量，提高分组聚合运算的效率。



### 3.2 相关参数

默认就是开启的

```hive
--启用map-side聚合
set hive.map.aggr=true;

--用于检测源表数据是否适合进行map-side聚合。检测的方法是：先对若干条数据进行map-side聚合，若聚合后的条数和聚合前的条数比值小于该值，则认为该表适合进行map-side聚合；否则，认为该表数据不适合进行map-side聚合，后续数据便不再进行map-side聚合。
set hive.map.aggr.hash.min.reduction=0.5;

--用于检测源表是否适合map-side聚合的条数。
set hive.groupby.mapaggr.checkinterval=100000;

--map-side聚合所用的hash table，占用map task堆内存的最大比例，若超出该值，则会对hash table进行一次flush。
set hive.map.aggr.hash.force.flush.memory.threshold=0.9;
```

注意：

Map端取前10w行数据试聚合，判断可能不准，可能会将适合聚合的数据判断为不适合，改善方法为调整参数`set hive.map.aggr.hash.min.reduction=1;`

由于有flush操作，map端输出的行数可能会比预想的数据量要大点。效果不好的话，可以调整flush阈值或map端内存。



## 4. Join优化

Hive拥有多种join算法，包括Common Join，Map Join，Bucket Map Join，Sort Merge Buckt Map Join等



### 4.1 Common Join

Common Join是Hive中最稳定的join算法，其通过一个MapReduce Job完成一个join操作。Map端负责读取join操作所需表的数据，并按照关联字段进行分区，通过Shuffle，将其发送到Reduce端，相同key的数据在Reduce端完成最终的Join操作。

![image-20230124152635756](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/hive%20common%20join.png)

需要**注意**的是，sql语句中的join操作和执行计划中的Common Join任务并非一对一的关系，一个sql语句中的**相邻**的且**关联字段相同**的多个join操作可以合并为一个Common Join任务。

例如：

```hive
hive (default)> 
select 
    a.val, 
    b.val, 
    c.val 
from a 
join b on (a.key = b.key1) 
join c on (c.key = b.key1)
```

上述sql语句中两个join操作的关联字段均为b表的key1字段，则该语句中的两个join操作可由一个Common Join任务实现，也就是可通过一个Map Reduce任务实现。 

```hive
hive (default)> 
select 
    a.val, 
    b.val, 
    c.val 
from a 
join b on (a.key = b.key1) 
join c on (c.key = b.key2)
```

上述sql语句中的两个join操作关联字段各不相同，则该语句的两个join操作需要各自通过一个Common Join任务实现，也就是通过两个Map Reduce任务实现。



### 4.2 Map Join

Map Join算法可以通过两个只有map阶段的Job完成一个join操作。其适用场景为大表join小表。若某join操作满足要求，则第一个Job会读取小表数据，将其制作为hash table，并上传至Hadoop分布式缓存（本质上是上传至HDFS）。第二个Job会先从分布式缓存中读取小表数据，并缓存在Map Task的内存中，然后扫描大表数据，这样在map端即可完成关联操作

![image-20230124155651016](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/image-20230124155651016.png)

**Hint触发（过时）**

新版本需要开启参数

```hive
hive (default)> 
select /*+ mapjoin(ta) */
    ta.id,
    tb.id
from table_a ta
join table_b tb
on ta.id=tb.id;
```



**自动触发**

Hive在编译SQL语句阶段，起初所有的join操作均采用Common Join算法实现。

之后在物理优化阶段，Hive会根据每个Common Join任务所需表的大小判断该Common Join任务是否能够转换为Map Join任务，若满足要求，便将Common Join任务自动转换为Map Join任务。

但有些Common Join任务所需的表大小，在SQL的编译阶段是未知的（例如对子查询进行join操作），所以这种Common Join任务是否能转换成Map Join任务在编译阶是无法确定的。

针对这种情况，Hive会在编译阶段生成一个条件任务（Conditional Task），其下会包含一个计划列表，计划列表中包含转换后的Map Join任务以及原有的Common Join任务。最终具体采用哪个计划，是在运行时决定的。



**大致思路**如下：

先生成所有Map join任务，之后遍历判断是否可行

![image-20230124171125279](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/map%20join%E5%A4%A7%E8%87%B4%E6%B5%81%E7%A8%8B.png)



**具体判断逻辑**如下

**基于Commom Join Task做优化**

![image-20230124171203859](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/map%20join%E6%B5%81%E7%A8%8B%E5%9B%BE.png)



**子任务Map Join合并**

如果有三个表a、b、c，大小已知，a为大表，join key不同，将会生成两个common join任务

但如果b、c表加起来的大小没有超过map join阈值，那么map端会同时把b、c两个表缓存到map端，在map端完成join。这就是子任务map join合并。



**涉及到的参数**

```hive
--启动Map Join自动转换
set hive.auto.convert.join=true;

--一个Common Join operator转为Map Join operator的判断条件,若该Common Join相关的表中,存在n-1张表的已知大小总和<=该值,则生成一个Map Join计划,此时可能存在多种n-1张表的组合均满足该条件,则hive会为每种满足条件的组合均生成一个Map Join计划,同时还会保留原有的Common Join计划作为后备(back up)计划,实际运行时,优先执行Map Join计划，若不能执行成功，则启动Common Join后备计划。
set hive.mapjoin.smalltable.filesize=250000;

--开启无条件转Map Join
set hive.auto.convert.join.noconditionaltask=true;

--无条件转Map Join时的小表之和阈值,若一个Common Join operator相关的表中，存在n-1张表的大小总和<=该值,此时hive便不会再为每种n-1张表的组合均生成Map Join计划,同时也不会保留Common Join作为后备计划。而是只生成一个最优的Map Join计划。
set hive.auto.convert.join.noconditionaltask.size=10000000;
```



**获取分区信息**

```hive
desc formatted table_name partition(partition_col='partition');
```



### 4.3 Bucket Map Join

Bucket Map Join是对Map Join算法的改进，其打破了Map Join只适用于大表join小表的限制，可用于大表join大表的场景。

Bucket Map Join的核心思想是：若能保证参与join的表均为分桶表，且关联字段为分桶字段，且其中一张表的分桶数量是另外一张表分桶数量的**整数倍**，就能保证参与join的两张表的分桶之间具有明确的关联关系，所以就可以在两表的分桶间进行Map Join操作了。这样一来，第二个Job的Map端就无需再缓存小表的全表数据了，而只需缓存其所需的分桶即可。

![image-20230124161652827](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/bucket%20map%20join.png)



MR引擎（开发团队基本已放弃MR引擎，如果是tez、spark引擎是可以自动转换的）Bucket Map Join不支持自动转换，发须通过用户在SQL语句中提供如下Hint提示，并配置如下相关参数，方可使用。

```hive
--关闭cbo优化，cbo会导致hint信息被忽略
set hive.cbo.enable=false;
--map join hint默认会被忽略(因为已经过时)，需将如下参数设置为false
set hive.ignore.mapjoin.hint=false;
--启用bucket map join优化功能
set hive.optimize.bucketmapjoin = true;
```



**案例**

首先需要依据源表创建两个分桶表，order_detail建议分16个bucket，payment_detail建议分8个bucket,注意**分桶个数**的倍数关系以及**分桶字段**。

```hive
--订单表
hive (default)> 
drop table if exists order_detail_bucketed;
create table order_detail_bucketed(
    id           string comment '订单id',
    user_id      string comment '用户id',
    product_id   string comment '商品id',
    province_id  string comment '省份id',
    create_time  string comment '下单时间',
    product_num  int comment '商品件数',
    total_amount decimal(16, 2) comment '下单金额'
)
clustered by (id) into 16 buckets
row format delimited fields terminated by '\t';

--支付表
hive (default)> 
drop table if exists payment_detail_bucketed;
create table payment_detail_bucketed(
    id              string comment '支付id',
    order_detail_id string comment '订单明细id',
    user_id         string comment '用户id',
    payment_time    string comment '支付时间',
    total_amount    decimal(16, 2) comment '支付金额'
)
clustered by (order_detail_id) into 8 buckets
row format delimited fields terminated by '\t';

```

然后向两个分桶表导入数据

```hive
--订单表
hive (default)> 
insert overwrite table order_detail_bucketed
select
    id,
    user_id,
    product_id,
    province_id,
    create_time,
    product_num,
    total_amount   
from order_detail
where dt='2020-06-14';

--分桶表
hive (default)> 
insert overwrite table payment_detail_bucketed
select
    id,
    order_detail_id,
    user_id,
    payment_time,
    total_amount
from payment_detail
where dt='2020-06-14';
```

设置以下参数

```hive
--关闭cbo优化，cbo会导致hint信息被忽略，需将如下参数修改为false
set hive.cbo.enable=false;
--map join hint默认会被忽略(因为已经过时)，需将如下参数修改为false
set hive.ignore.mapjoin.hint=false;
--启用bucket map join优化功能,默认不启用，需将如下参数修改为true
set hive.optimize.bucketmapjoin = true;
```

重写SQL语句

```hive
hive (default)> 
select /*+ mapjoin(pd) */
    *
from order_detail_bucketed od
join payment_detail_bucketed pd on od.id = pd.order_detail_id;
```

需要注意的是，Bucket Map Join的执行计划的基本信息和普通的Map Join无异，若想看到差异，可执行如下语句，查看执行计划的详细信息。详细执行计划中，如在Map Join Operator中看到 “**BucketMapJoin: true**”，则表明使用的Join算法为Bucket Map Join。

```hive
hive (default)> 
explain extended select /*+ mapjoin(pd) */
    *
from order_detail_bucketed od
join payment_detail_bucketed pd on od.id = pd.order_detail_id;
```



### 4.4 Sort Merge Bucket Map Join

Sort Merge Bucket Map Join（简称SMB Map Join）基于Bucket Map Join。SMB Map Join要求，参与join的表均为分桶表，且需保证分桶内的数据是有序的，且分桶字段、排序字段和关联字段为相同字段，且其中一张表的分桶数量是另外一张表分桶数量的整数倍。

SMB Map Join同Bucket Join一样，同样是利用两表各分桶之间的关联关系，在分桶之间进行join操作，不同的是，分桶之间的join操作的实现原理。Bucket Map Join，两个分桶之间的join实现原理为Hash Join算法；而SMB Map Join，两个分桶之间的join实现原理为Sort Merge Join算法。

Hash Join和Sort Merge Join均为关系型数据库中常见的Join实现算法。Hash Join的原理相对简单，就是对参与join的一张表构建hash table，然后扫描另外一张表，然后进行逐行匹配。Sort Merge Join需要在两张按照关联字段排好序的表中进行

![image-20230124164239522](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/SMB%20map%20join.png)

Hive中的SMB Map Join就是对两个分桶的数据按照上述思路进行Join操作。可以看出，SMB Map Join与Bucket Map Join相比，在进行Join操作时，Map端是无需对整个Bucket构建hash table，也无需在Map端缓存整个Bucket数据的，每个Mapper只需按顺序逐个key读取两个分桶的数据进行join即可。



**案例**

Sort Merge Bucket Map Join有两种触发方式，包括Hint提示和自动转换。**Hint提示已过时**，不推荐使用。

前提：

- 两个表是分桶表
- 按照join key分桶且排序
- 桶数量有倍数关系

满足前提就会有SMB join优化，不需要其他条件。也不用担心内存不足。



建表（注意要排序）

```hive
--订单表
hive (default)> 
drop table if exists order_detail_sorted_bucketed;
create table order_detail_sorted_bucketed(
    id           string comment '订单id',
    user_id      string comment '用户id',
    product_id   string comment '商品id',
    province_id  string comment '省份id',
    create_time  string comment '下单时间',
    product_num  int comment '商品件数',
    total_amount decimal(16, 2) comment '下单金额'
)
clustered by (id) sorted by(id) into 16 buckets
row format delimited fields terminated by '\t';

--支付表
hive (default)> 
drop table if exists payment_detail_sorted_bucketed;
create table payment_detail_sorted_bucketed(
    id              string comment '支付id',
    order_detail_id string comment '订单明细id',
    user_id         string comment '用户id',
    payment_time    string comment '支付时间',
    total_amount    decimal(16, 2) comment '支付金额'
)
clustered by (order_detail_id) sorted by (order_detail_id) into 8 buckets
row format delimited fields terminated by '\t';
```

下面是自动转换的相关参数：

```hive
--启动Sort Merge Bucket Map Join优化
set hive.optimize.bucketmapjoin.sortedmerge=true;
--使用自动转换SMB Join
set hive.auto.convert.sortmerge.join=true;

hive (default)> 
select
    *
from order_detail_sorted_bucketed od
join payment_detail_sorted_bucketed pd
on od.id = pd.order_detail_id;
```





## 5. 数据倾斜

数据倾斜问题，通常是指参与计算的数据分布不均，即某个key或者某些key的数据量远超其他key，导致在shuffle阶段，大量相同key的数据被发往同一个Reduce，进而导致该Reduce所需的时间远超其他Reduce，成为整个任务的瓶颈。

Hive中的数据倾斜常出现在分组聚合和join操作的场景中，下面分别介绍在上述两种场景下的优化思路。



### 5.1 Group By倾斜

如果group by分组字段的值分布不均，就可能导致大量相同的key进入同一Reduce，从而导致数据倾斜问题

**Map Side聚合**

```hive
--启用map-side聚合
set hive.map.aggr=true;

--用于检测源表数据是否适合进行map-side聚合。检测的方法是：先对若干条数据进行map-side聚合，若聚合后的条数和聚合前的条数比值小于该值，则认为该表适合进行map-side聚合；否则，认为该表数据不适合进行map-side聚合，后续数据便不再进行map-side聚合。
set hive.map.aggr.hash.min.reduction=0.5;

--用于检测源表是否适合map-side聚合的条数。
set hive.groupby.mapaggr.checkinterval=100000;

--map-side聚合所用的hash table，占用map task堆内存的最大比例，若超出该值，则会对hash table进行一次flush。
set hive.map.aggr.hash.force.flush.memory.threshold=0.9;
```



**Skew-GroupBy优化**

启动两个MR任务，第一个MR按照随机数分区，将数据分散发送到Reduce，完成部分聚合，第二个MR按照分组字段分区，完成最终聚合。

```hive
--启用分组聚合数据倾斜优化
set hive.groupby.skewindata=true;
```

**Map端聚合方案更快一些**，但是消耗内存更多，如果内存不足导致map聚合维护的hash table频繁flush，相当于没有解决数据倾斜。



### 5.2 Join倾斜

未经优化的join操作，默认是使用common join算法，也就是通过一个MapReduce Job完成计算。Map端负责读取join操作所需表的数据，并按照关联字段进行分区，通过Shuffle，将其发送到Reduce端，相同key的数据在Reduce端完成最终的Join操作。

如果关联字段的值分布不均，就可能导致大量相同的key进入同一Reduce，从而导致数据倾斜问题。



**Map Join**

使用map join算法，join操作仅在map端就能完成，没有shuffle操作，没有reduce阶段，自然不会产生reduce端的数据倾斜。该方案适用于大表join小表时发生数据倾斜的场景

```hive
--启动Map Join自动转换
set hive.auto.convert.join=true;

--一个Common Join operator转为Map Join operator的判断条件,若该Common Join相关的表中,存在n-1张表的大小总和<=该值,则生成一个Map Join计划,此时可能存在多种n-1张表的组合均满足该条件,则hive会为每种满足条件的组合均生成一个Map Join计划,同时还会保留原有的Common Join计划作为后备(back up)计划,实际运行时,优先执行Map Join计划，若不能执行成功，则启动Common Join后备计划。
set hive.mapjoin.smalltable.filesize=250000;

--开启无条件转Map Join
set hive.auto.convert.join.noconditionaltask=true;

--无条件转Map Join时的小表之和阈值,若一个Common Join operator相关的表中，存在n-1张表的大小总和<=该值,此时hive便不会再为每种n-1张表的组合均生成Map Join计划,同时也不会保留Common Join作为后备计划。而是只生成一个最优的Map Join计划。
set hive.auto.convert.join.noconditionaltask.size=10000000;
```



**Skew Join**

大表和大表Join倾斜的时候，使用Bucket Join是无法解决数据倾斜的，可以使用skew join

skew join的原理是，为倾斜的大key**单独启动一个map join任务**进行计算，其余key进行正常的common join

判断倾斜Key的过程有Reduce端检测得到

![image-20230125221453674](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/Hive/Skew%20Join%E5%8E%9F%E7%90%86.png)

这种方案对参与join的源表大小没有要求，但是对两表中倾斜的key的数据量有要求，要求一张表中的倾斜key的数据量比较小（方便走mapjoin）

参数：

```hive
--启用skew join优化
set hive.optimize.skewjoin=true;
--触发skew join的阈值，若某个key的行数超过该参数值，则触发
set hive.skewjoin.key=100000;
```



**调整SQL语句**

若参与join的两表均为大表，其中一张表的数据是倾斜的，此时也可通过以下方式对SQL语句进行相应的调整。

即主表加随机数，右表扩容

```hive
hive (default)>
select
    *
from(
    select --打散操作
        concat(id,'_',cast(rand()*2 as int)) id,
        value
    from A
)ta
join(
    select --扩容操作
        concat(id,'_',0) id,
        value
    from B
    union all
    select
        concat(id,'_',1) id,
        value
    from B
)tb
on ta.id=tb.id;
```



## 6. 并行度

### 6.1 Map并行度

**适用场景**

- **查询的表中存在大量小文件**：使用Hive提供的CombineHiveInputFormat，多个小文件合并为一个切片
- **map端有复杂的查询逻辑**：调整map切片大小

```hive
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; -- 默认开启
--一个切片的最大值
set mapreduce.input.fileinputformat.split.maxsize=256000000;
```



### 6.2 Reduce并行度

Reduce端的并行度，也就是Reduce个数。相对来说，更需要关注。Reduce端的并行度，可由用户自己指定，也可由Hive自行根据该MR Job输入的文件大小进行估算。

```hive
--指定Reduce端并行度，默认值为-1，表示用户未指定
set mapreduce.job.reduces;
--Reduce端并行度最大值，默认1009
set hive.exec.reducers.max;
--单个Reduce Task计算的数据量，用于估算Reduce并行度
set hive.exec.reducers.bytes.per.reducer;
```

Reduce端并行度的确定逻辑如下：

若指定参数**mapreduce.job.reduces**的值为一个非负整数，则Reduce并行度为指定值。否则，Hive自行估算Reduce并行度，估算逻辑如下：

假设Job输入的文件大小为**totalInputBytes**（Map端输入文件）

参数**hive.exec.reducers.bytes.per.reducer**的值为bytesPerReducer。

参数**hive.exec.reducers.max**的值为maxReducers。

设置后并行度为：

```
min⁡( ceil(totalInputBytes / bytesPerReducer), maxReducers )
```

Hive自行估算Reduce并行度时，是以整个MR Job输入的文件大小作为依据的。因此，在某些情况下其估计的并行度很可能并不准确，此时就需要用户根据实际情况来指定Reduce并行度了。

MR引擎使用的是Map端输入文件大小估算Reduce文件数量，不一定准确，由于MR引擎已过时，社区不会再优化这个问题，往往需要开发人员自己再调整下。

如果是Spark、Tez引擎，会使用Map端输出文件来估算，相对较准确。



## 7. 其他优化

### 7.1 CBO优化

CBO是指Cost based Optimizer，即基于计算成本的优化。

在Hive中，计算成本模型考虑到了：数据的**行数（主要）**、CPU、本地IO、HDFS IO、网络IO等方面。Hive会计算同一SQL语句的不同执行计划的计算成本，并选出成本最低的执行计划。

目前CBO在hive的MR引擎下主要用于join的优化，例如多表join的join顺序。

```hive
--是否启用cbo优化 
set hive.cbo.enable=true;
```

CBO优化对于执行计划中join顺序是有影响的，在不影响结果情况下，基于行数的成本优化器，可能会将小表的join提前，大表join靠后

当把谓词下推的设置关掉之后，explain之后会发现，执行计划还是实现了谓词下推的功能

这是因为CBO优化器做为更高层的优化方案，覆写了谓词下推的优化方案



### 7.2 谓词下推

谓词下推（predicate pushdown）是指，尽量将过滤操作前移，以减少后续计算步骤的数据量。

```hive
--是否启动谓词下推（predicate pushdown）优化
set hive.optimize.ppd = true;
```



### 7.3 矢量化查询

Hive的矢量化查询，可以极大的提高一些典型查询场景（例如scans, filters, aggregates, and joins）下的CPU使用效率。

若执行计划中，出现“Execution mode: vectorized”字样，即表明使用了矢量化计算。

```hive
set hive.vectorized.execution.enabled=true;
```

https://cwiki.apache.org/confluence/display/Hive/Vectorized+Query+Execution#VectorizedQueryExecution-Limitations



### 7.4 Fetch抓取

Fetch抓取是指，Hive中对某些情况的查询可以不必使用MapReduce计算。例如：select * from emp;在这种情况下，Hive可以简单地读取emp对应的存储目录下的文件，然后输出查询结果到控制台。

```hive
--是否在特定场景转换为fetch 任务
--设置为none表示不转换
--设置为minimal表示支持select *，分区字段过滤，Limit等
--设置为more表示支持select 任意字段,包括函数，过滤，和limit等
set hive.fetch.task.conversion=more;
```



### 7.5 本地模式

大多数的Hadoop Job是需要Hadoop提供的完整的可扩展性来处理大数据集的。不过，有时Hive的输入数据量是非常小的。在这种情况下，为查询触发执行任务消耗的时间可能会比实际job的执行时间要多的多。

对于大多数这种情况，Hive可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间可以明显被缩短。

```hive
--开启自动转换为本地模式
set hive.exec.mode.local.auto=true;  

--设置local MapReduce的最大输入数据量，当输入数据量小于这个值时采用local  MapReduce的方式，默认为134217728，即128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;

--设置local MapReduce的最大输入文件个数，当输入文件个数小于这个值时采用local MapReduce的方式，默认为4
set hive.exec.mode.local.auto.input.files.max=10;
```



### 7.6 并行执行

Hive会将一个SQL语句转化成一个或者多个Stage，每个Stage对应一个MR Job。默认情况下，Hive同时只会执行一个Stage。但是某SQL语句可能会包含多个Stage，但这多个Stage可能并非完全互相依赖，也就是说有些Stage是可以并行执行的。此处提到的并行执行就是指这些Stage的并行执行。

```hive
--启用并行执行优化
set hive.exec.parallel=true;       
    
--同一个sql允许最大并行度，默认为8
set hive.exec.parallel.thread.number=8; 
```



### 7.7 严格模式

通过设置某些参数防止危险操作

- **分区表不使用分区过滤**

  ```
  set hive.strict.checks.no.partition.filter = true;
  ```

- **使用order by没有limit过滤**

  ```
  set hive.strict.checks.orderby.no.limit = true;
  ```

- **笛卡尔积**

  ```
  set hive.strict.checks.cartesian.product = true;
  ```

  
