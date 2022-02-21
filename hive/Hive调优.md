# Hive调优

### 分区裁剪与列裁剪

列裁剪就是在查询时只读取需要的列，分区裁剪就是只读取需要的分区。当列很多或者数据量很大时，如果 select * 或者不指定分区，全列扫描和全表扫描效率都很低。

Hive 在读数据的时候，可以只读取查询中所需要用到的列，而忽略其他的列。这样做可以节省读取开销：中间表存储开销和数据整合开销。

分区裁剪与列裁剪的设置是默认开启的，只要在where条件里面加上分区过滤即可。

```shell
set hive.optimize.cp = true;      # 列裁剪
set hive.optimize.pruner = true;  # 分区裁剪 
```



### 谓词下推

将 SQL 语句中的 where 谓词逻辑都尽可能提前执行，减少下游处理的数据量。对应逻辑优化器是 PredicatePushDown，配置项为 hive.optimize.ppd，默认为 true。

```shell
set hive.optimize.ppd = true;      #谓词下推，默认是 true
```

或者是手动sql实现，where子句手动前移

```sql
select b.id from bigtable b join (select id from bigtable where id <= 10) o on b.id = o.id;
```



### Group By 优化

#### Map端聚合

默认情况下，Map 阶段同一 Key 数据分发给一个 Reduce，当一个 key 数据过大时就倾斜了；

因此可以开启Map端聚合，在Map阶段先把数据聚合一次，到reduce端再聚合一次。

```shell
# 是否开启Map聚合，默认True
set hive.map.aggr = true;
# 在 Map 端进行聚合操作的条目数目
set hive.groupby.mapaggr.checkinterval = 100000;
# 有数据倾斜的时候进行负载均衡（默认是 false）
set hive.groupby.skewindata = true;
```

**设置负载均衡的时候需要注意**

设置为True时，生成的查询计划会有两个 MR Job；

第一个MR任务，Map 的输出结果会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果是相同的 Group By Key 有可能被分发到不同的 Reduce中，从而达到负载均衡的目的；

第二个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作（虽然能解决数据倾斜，但是不能让运行速度的更快）；

在数据量比较小的时候，开启负载均衡反而可能会使效率变低；



### 矢量计算

vectorization : 矢量计算的技术，在计算类似scan, filter, aggregation的时候，vectorization技术以设置批处理的增量大小为 1024 行单次来达到比单条记录单次获得更高的效率；

矢量计算要求你的数据存储格式必须为ORC格式；

```shell
set hive.vectorized.execution.enabled = true;         # 默认false
set hive.vectorized.execution.reduce.enabled = true;  # 默认true
```



当然，有时候矢量计算也会触发一些奇怪的错误，类似下面的报错,这时候需要关掉矢量计算

```

```

Hive关于矢量计算的文档：[Vectorized Query Execution]([Vectorized Query Execution - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/Vectorized+Query+Execution#space-menu-link-content))

点评

个人发现矢量计算容易报错，还是关闭为妙



### 多重模式

从同一个表取数据，做不同的逻辑，可以考虑使用多重模式，只用扫描一次表,一次扫描，多次插入

```sql
insert int t_ptn partition(city=A). select id,name,sex, age from student 
where city= A;
insert int t_ptn partition(city=B). select id,name,sex, age from student 
where city= B;
insert int t_ptn partition(city=c). select id,name,sex, age from student 
where city= c;

-- 修改为：
from student
insert int t_ptn partition(city=A) select id,name,sex, age where city= A
insert int t_ptn partition(city=B) select id,name,sex, age where city= B
```



### in/exists 语句

in/exists 可以考虑用 left semi join 代替

**IN适合于外表大而内表小的情况；EXISTS适合于外表小而内表大的情况。**

**in /exists / left semi join 不会产生笛卡尔积 ！ inner join可能会产生笛卡尔积！**

in：将数据放到内存中一个一个去对比

```sql
select * from A where A.id in (select B.id from B)
```

它查出B表中的所有id字段并缓存起来，之后,检查A表的id是否与B表中的id相等，如果相等则将A表的记录加入结果集中，直到遍历完A表的所有记录；

可以看出,当B表数据较大时不适合使用in(),因为它会B表数据全部遍历一次；

exists：拿到一条数据，去另一个表里面查询

```sql
select * from A where exists (select B.id from B where A.id = B.id)
```

当B表比A表数据大时适合使用exists()，因为它没有那么遍历操作，只需要再执行一次查询就行。



**推荐使用 left semi join替代in / exist**

```sql
select a.id, a.name from a left semi join b on a.id = b.id;
```



**点评**

这里我是不太信的，因为explain后发现，in /exists底层走的就是left semi join

exists在left semi join前，会对b表做group by操作



**CBO 优化**

join 的时候表的顺序的关系：前面的表都会被加载到内存中。后面的表进行磁盘扫描

```sql
select a.*, b.*, c.* from a join b on a.id = b.id join c on a.id = c.id;
```

Hive 自 0.14.0 开始，加入了一项 "Cost based Optimizer" 来对 HQL 执行计划进行优化，这个功能通过 `hive.cbo.enable` 来开启。在 Hive 1.1.0 之后，这个 feature 是默认开启的，它可以 自动优化 HQL 中多个 Join 的顺序，并选择合适的 Join 算法。

CBO，成本优化器，代价最小的执行计划就是最好的执行计划。传统的数据库，成本优化器做出最优化的执行计划是依据统计信息来计算的。

Hive 的成本优化器也一样，Hive 在提供最终执行前，优化每个查询的执行逻辑和物理执行计划。这些优化工作是交给底层来完成的。根据查询成本执行进一步的优化，从而产生潜在的不同决策：如何排序连接，执行哪种类型的连接，并行度等等。

```sql
set hive.cbo.enable=true;
set hive.compute.query.using.stats=true;
set hive.stats.fetch.column.stats=true;
set hive.stats.fetch.partition.stats=true;
```

存在一个有意思的现象，当把谓词下推的设置关掉之后，explain之后会发现，执行计划还是实现了谓词下推的功能

这是因为CBO优化器做为更高层的优化方案，覆写了谓词下推的优化方案



### Map Join

MapJoin 是将 Join 双方比较小的表直接分发到各个 Map 进程的内存中，在 Map 进程中进行 Join 操作，这样就不用进行 Reduce 步骤，从而提高了速度

首先会产生一个Mapreduce任务，将小表抓取到内存中，并以Hashtable形式输出，在下一阶段，当 MapReduce 任务启动时，会将这个哈希表文件上传到 Hadoop 分布式缓存中，该缓存会将这些文件发送到每个 Mapper 的本地磁盘上。因此，所有 Mapper 都可以将此持久化的哈希表文件加载回内存，并像之前一样进行 Join。

mapjoin 执行过程中会有一个普通的 join 来兜底，如果 mapjoin 失效走普通 join 保证任务不失败。

如果不指定 MapJoin或者不符合 MapJoin 的条件，那么 Hive 解析器会将 Join 操作转换成 Common Join，即：在Reduce 阶段完成 Join。容易发生数据倾斜

```shell
# 设置自动选择 MapJoin,默认为True
set hive.auto.convert.join=true; 
# 大表小表的阈值设置（默认 25M 以下认为是小表）：
set hive.mapjoin.smalltable.filesize=25000000;
```

需要注意的点：

当小表作为左连接主表，map join会失效，因为主表分发出去会造成数据出错



### 大表SMB Join 大表

全称：Sort Merge Bucket Join，分桶Join

原理：当第一个表按照Join Key分了n个桶，第二个表按照Join Key分了n的整数倍个桶，那么相同的Key肯定落在一样的文件编号里面，join时候只需要表1特定文件和表2特定文件两两联合，就可以得到结果，无需文件间进行笛卡尔积式的Join

需要开启设置：

```shell
set hive.optimize.bucketmapjoin = true;
set hive.optimize.bucketmapjoin.sortedmerge = true;
set hive.input.format=org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
```



### 严格模式

设置严格模式可以拒绝没有on条件的join语句提交，避免笛卡尔积



### 数据倾斜

#### 单表优化

单表携带Group by，会产生Shuffle

使用参数：

```shell
# 是否在 Map 端进行聚合，默认为 True
set hive.map.aggr = true;
# 在 Map 端进行聚合操作的条目数目
set hive.groupby.mapaggr.checkinterval = 100000;
# 有数据倾斜的时候进行负载均衡（默认是 false）
set hive.groupby.skewindata = true;
```

当选项`hive.groupby.skewindata`设定为true，生成的查询计划会有两个MRJob。

第一个MRJob 中，Map的输出结果集合会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果，这样处理的结果是相同的GroupBy Key有可能被分发到不同的Reduce中，从而达到负载均衡的目的；

第二个MRJob再根据预处理的数据结果按照GroupBy Key分布到Reduce中（这个过程可以保证相同的GroupBy Key被分布到同一个Reduce中），最后完成最终的聚合操作。

增加reducer的数量：

**调整reduce个数方法一**

1. 每个 Reduce 处理的数据量默认是 256MB

```shell
set hive.exec.reducers.bytes.per.reducer = 256000000
```

2. 每个任务最大的 reduce 数，默认为 1009

```shell
set hive.exec.reducers.max = 1009
```

3. 计算 reducer 数的公式

```shell
N=min(参数 2，总输入数据量/参数 1)(参数 2 指的是上面的 1009，参数 1 值得是 256M)
```

**调整reduce个数方法二**

在 hadoop 的 mapred-default.xml 文件中修改设置每个 job 的 Reduce 个数

```shell
set mapreduce.job.reduces = 15;
```



#### Join 数据倾斜优化

使用参数

如果确定是由于 join 出现的数据倾斜，那么请做如下设置

```shell
# join 的键对应的记录条数超过这个值则会进行分拆，值根据具体数据量设置
set hive.skewjoin.key=100000;


# 如果是 join 过程出现倾斜应该设置为 true
set hive.optimize.skewjoin=false;
```

如果开启了，在 Join 过程中 Hive 会将计数超过阈值 hive.skewjoin.key（默认 100000）的倾斜 key 对应的行临时写进文件中，然后再启动另一个 job 做 map join 生成结果。通过hive.skewjoin.mapjoin.map.tasks 参数还可以控制第二个 job 的 mapper 数量，默认10000。

```shell
set hive.skewjoin.mapjoin.map.tasks=10000;
```



Map Join

没有reduce端，就没有数据倾斜



### Hive Job优化

#### Map优化

**复杂文件增加map数量**

当 input 的文件都很大，任务逻辑复杂，map 执行非常慢的时候，可以考虑增加 Map 数，来使得每个 map 处理的数据量减少，从而提高任务的执行效率。

增加 map 的方法为：根据`computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M`调整 maxSize 最大值。让 maxSize 最大值低于 blocksize 就可以增加 map 的个数。



#### **小文件合并**

由于一个小文件就是一个Map Task，所以在 map 执行前合并小文件，可以减少 map 数：CombineHiveInputFormat 具有对小文件进行合并的功能（系统默认的格式）。HiveInputFormat 没有对小文件合并功能

```shell
set hive.input.format = org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```



在 Map-Reduce 的任务结束时合并小文件的设置： 

- 在 map-only 任务结束时合并小文件，默认 true

  ```shell
  set hive.merge.mapfiles = true;
  ```

- 在 map-reduce 任务结束时合并小文件，默认 false

  ```shell
  set hive.merge.mapredfiles = true;
  ```

- 合并文件的大小，默认 256M

  ```shell
  set hive.merge.size.per.task = 268435456;
  ```

- 当输出文件的平均大小小于该值时，启动一个独立的 map-reduce 任务进行文件 merge

  ```shell
  set hive.merge.smallfiles.avgsize = 16777216;
  ```

#### Map端聚合

```shell
set hive.map.aggr=true;相当于 map 端执行 combiner
```

#### 推测执行

```shell
set mapred.map.tasks.speculative.execution = true #默认是 true
```

点评：没感觉出来推测执行有啥用，可以考虑关掉



#### **Reduce** **优化**

**调整reduce个数**

详情请看单表优化环节



**reduce** **个数并不是越多越好**

1. 过多的启动和初始化 reduce 也会消耗时间和资源；
2. 有多少个 reduce，就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题

在设置 reduce 个数的时候也需要考虑这两个原则：处理大数据量利用合适的 reduce 数；使单个 reduce 任务处理数据量大小要合适；



**推测执行**

```shell
mapred.reduce.tasks.speculative.execution （hadoop 里面的）
hive.mapred.reduce.tasks.speculative.execution（hive 里面相同的参数，效果和hadoop 里面的一样两个随便哪个都行）
```



#### **Hive** **任务整体优化**

**Fetch** **抓取**

简单任务不走mapreduce

```shell
 set hive.fetch.task.conversion=more;
```

**本地模式**

有时 Hive 的输入数据量是非常小的，使用本地模式可以单机处理所有的任务

```shell
# 开启本地 mr
set hive.exec.mode.local.auto=true; 
# 设置 local mr 的最大输入数据量，当输入数据量小于这个值时采用 local mr 的方式，默认为 134217728，即 128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;
# 设置 local mr 的最大输入文件个数，当输入文件个数小于这个值时采用 local mr 的方式，默认为 4
set hive.exec.mode.local.auto.input.files.max=10;
```



**并行执行**

某个特定的 job 可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的

开启并行执行，job执行速度可以加快

```shell
# 打开任务并行执行，默认为 false
set hive.exec.parallel=true;
# 同一个 sql 允许最大并行度，默认为 8
set hive.exec.parallel.thread.number=16;
```



**JVM** **重用**

小文件过多时候使用，文件比较大时候一般不用，因为不能前一个JVM实例在处理，后一个任务还在死等.





## Hive三大场景优化

常见的ETL问题：

| HiveSQL执行日志                             | 可能原因                                         |
| ------------------------------------------- | ------------------------------------------------ |
| 1. ETL长时间任务没有开始                    | 查询文件数过大，需要限制查询数据分区             |
| 2. 长时间没有输出Mapper/Reducer数量         | 一般yarn负载较高                                 |
| 3. 长时间mapper=0%，reducer=0%              | task数量较多，创建task耗时较长，等待task资源分配 |
| 4. 长时间mapper=99%，reducer=10%            | map处理数据不均，存在数据倾斜                    |
| 5. 长时间mapper=100%，reducer=99%           | 数据存在倾斜，导致慢查询                         |
| 6. mapper进度回退，reducer进度不变          | map端链接不上或结果文件有问题，设计容错机制重算  |
| 7. 任务执行失败，尝试多次后成功             | 内存溢出或机器故障导致任务失败                   |
| 8. 某个task处理数据量不大，但是执行耗时很长 | 小文件过多带来文件拉取耗时                       |



### 数据量大

使用分区裁剪、列裁剪、谓词下推

开启他们的配置参数

```shell
set hive.optimize.cp = true;       # 列裁剪
set hive.optimize.pruner = true;   # 分区裁剪 
set hive.optimize.ppd = true;      # 谓词下推，默认是 true
```

### 小文件多

Map端合并小文件

```
mapred.max.split.size
mapred.min.split.size.per.node
mapred.min.split.size.per.rack
```

Reduce端合并小文件

```
hive.exec.reducers.bytes.per.reducer
```

写入后合并小文件

```
hive.merge.mapredfiles
hive.merge.smallfiles.avgsize
hive.merge.size.per.task
```

多分区小文件合并

```
distribute by key1,key2
```



### 数据倾斜

1. 热点数据与非热点数据分开计算
2. mapjoin实现小表链接大表
3. 大表热点数据在key上拼接0～n的前缀/后缀，同时小表冗余扩大n倍数据，保证join到对应键值，然后去掉前缀/后缀。
4. a left join b 可以转换为：b left join a 加 union，基于维度对指标进行聚合函数封装。
5. Null值不参与关联，给Null值分配随机值
6. 两阶段聚合（局部 + 全局聚合）

count distinct去重优化配置

```
hive.optimize.countdistinct 默认是true,3.0新配置
去重并计数的作业会分成两个作业来处理这类sql,已达到减缓sql数据倾斜
```



### 万用配置

```shell
万能参数配置，最优的使用机器资源：
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.exec.parallel=true;
set mapred.max.split.size=64000000;
set mapred.min.split.size.per.node=64000000;
set mapred.min.split.size.per.rack=64000000;
set hive.exec.reducers.bytes.per.reducer=256000000;
set hive.exec.reducers.max=2000;
set hive.merge.mapredfiles=true;
set hive.merge.smallfiles.avgsize = 128000000;
set hive.merge.size.per.task=128000000;
```



