# Flink SQL优化

阿里云文档：[高性能Flink SQL优化技巧 (aliyun.com)](https://help.aliyun.com/document_detail/98981.html)

## 1、Group Aggregate 优化

### 1.1 开启 MiniBatch

MiniBatch 是微批处理，原理是缓存一定的数据后再触发处理，以减少对 State 的访问，从而提升吞吐并减少数据的输出量。MiniBatch 主要依靠在每个 Task 上注册的 Timer 线程来触发微批，需要消耗一定的线程调度性能。

```java
// 获取 tableEnv 的配置对象
Configuration configuration = tEnv.getConfig().getConfiguration();
// 设置参数：
// 开启 miniBatch
configuration.setString("table.exec.mini-batch.enabled", "true");
// 批量输出的间隔时间
configuration.setString("table.exec.mini-batch.allow-latency", "5 s");
// 防止 OOM 设置每个批次最多缓存数据的条数，可以设为 2 万条
configuration.setString("table.exec.mini-batch.size", "20000");
```

FlinkSQL 参数配置列表：https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/config.html

微批处理通过增加延迟换取高吞吐，如果有超低延迟的要求，不建议开启微批处理。通常对于聚合的场景，微批处理可以显著的提升系统性能，建议开启。

注意事项：

1）目前，key-value 配置项仅被 Blink planner 支持。

2）1.12 之前的版本有 bug，开启 miniBatch，不会清理过期状态，也就是说如果设置状态的 TTL，无法清理过期状态。1.12 版本才修复这个问题。

参考 ISSUE：https://issues.apache.org/jira/browse/FLINK-17096



### 1.2 开启 LocalGlobal

**解决常见数据热点问题**

LocalGlobal 优 化 将 原 先 的 Aggregate 分 成 Local+Global 两 阶 段 聚 合 ， 即MapReduce 模型中的 Combine+Reduce 处理模式。第一阶段在上游节点本地攒一批数据进行聚合（localAgg），并输出这次微批的增量值（Accumulator）。第二阶段再将收到的 Accumulator 合并（Merge），得到最终的结果（GlobalAgg）。

![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/4421659951/p33642.png)

由上图可知：

LocalGlobal 本质上能够靠 LocalAgg 的聚合筛除部分倾斜数据，从而降低 GlobalAgg的热点，提升性能。

- 未开启 LocalGlobal 优化，由于流中的数据倾斜，Key 为红色的聚合算子实例需要处理更多的记录，这就导致了热点问题。
- 开启 LocalGlobal 优化后，先进行本地聚合，再进行全局聚合。可大大减少 GlobalAgg的热点，提高性能



**LocalGlobal 开启方式**：

1）LocalGlobal 优化需要先开启 MiniBatch，依赖于 MiniBatch 的参数。

2）table.optimizer.agg-phase-strategy: 聚合策略。默认 AUTO，支持参数 AUTO、TWO_PHASE(使用 LocalGlobal 两阶段聚合)、ONE_PHASE(仅使用 Global 一阶段聚合)。

```java
// 获取 tableEnv 的配置对象
Configuration configuration = tEnv.getConfig().getConfiguration();
// 设置参数：
// 开启 miniBatch
configuration.setString("table.exec.mini-batch.enabled", "true");
// 批量输出的间隔时间
configuration.setString("table.exec.mini-batch.allow-latency", "5 s");
// 防止 OOM 设置每个批次最多缓存数据的条数，可以设为 2 万条
configuration.setString("table.exec.mini-batch.size", "20000");
// 开启 LocalGlobal
configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE");
```

**判断是否生效**

观察最终生成的拓扑图的节点名字中是否包含GlobalGroupAggregate或LocalGroupAggregate。

**适用场景**
LocalGlobal 适用于提升如 SUM、COUNT、MAX、MIN 和 AVG 等普通聚合的性能，以及解决这些场景下的数据热点问题。

**注意事项**：

1）需要先开启 MiniBatch

2）开启 LocalGlobal 需要 UDAF 实现 Merge 方法。



### 1.3 开启 Split Distinct

**解决 COUNT DISTINCT 热点问题**

LocalGlobal 优化针对普通聚合（例如 SUM、COUNT、MAX、MIN 和 AVG）有较好的效果，对于 COUNT DISTINCT 收效不明显，因为 COUNT DISTINCT 在 Local 聚合时，对于 DISTINCT KEY 的去重率不高，导致在 Global 节点仍然存在热点。

之前，为了解决 COUNT DISTINCT 的热点问题，通常需要手动改写为两层聚合（增加按 Distinct Key 取模的打散层）。

从 Flink1.9.0 版本开始，提供了 COUNT DISTINCT 自动打散功能(即PartialFinal优化)，不需要手动重写。

PartialFinal和LocalGlobal的原理对比参见下图:

![](https://help-static-aliyun-doc.aliyuncs.com/assets/img/zh-CN/4421659951/p33643.png)

**统计一天的 UV**

```sql
SELECT day, COUNT(DISTINCT user_id)
FROM T
GROUP BY day
```

**与下面方式同效果**

```sql
SELECT day, SUM(cnt)
FROM (
	SELECT day, COUNT(DISTINCT user_id) as cnt
	FROM T
    GROUP BY day, MOD(HASH_CODE(user_id), 1024)
)
GROUP BY day
```

第一层聚合: 将 Distinct Key 打散求 COUNT DISTINCT。

第二层聚合: 对打散去重后的数据进行 SUM 汇总。

**开启方式**

默认不开启，使用参数显式开启：

- table.optimizer.distinct-agg.split.enabled: true，默认 false。
- table.optimizer.distinct-agg.split.bucket-num: Split Distinct 优化在第一层聚合中，被打散的 bucket 数目。默认 1024。

```java
// 获取 tableEnv 的配置对象
Configuration configuration = tEnv.getConfig().getConfiguration();
// 设置参数：
// 开启 Split Distinct
configuration.setString("table.optimizer.distinct-agg.split.enabled", "true");
// 第一层打散的 bucket 数目
configuration.setString("table.optimizer.distinct-agg.split.bucket-num", "1024");
```

**判断是否生效**

观察最终生成的拓扑图的节点名中是否包含 Expand 节点，或者原来一层的聚合变成了两层的聚合。

**适用场景**

使用 COUNT DISTINCT，但无法满足聚合节点性能要求。

**注意事项**：

1）目前不能在包含 UDAF 的 Flink SQL 中使用 Split Distinct 优化方法。

2）拆分出来的两个 GROUP 聚合还可参与 LocalGlobal 优化。

3）从 Flink1.9.0 版本开始，提供了 COUNT DISTINCT 自动打散功能，不需要手动重写（不用像上面的例子去手动实现）



### 1.4 改写为AGG WITH FILTER 语法 

**提升大量 COUNTDISTINCT 场景性能**

```sql
SELECT
    day,
    COUNT(DISTINCT user_id) AS total_uv,
    COUNT(DISTINCT CASE WHEN flag IN ('android', 'iphone') THEN user_id ELSE NULL END) AS app_uv,
	COUNT(DISTINCT CASE WHEN flag IN ('wap', 'other') THEN user_id ELSE NULL END) AS web_uv
FROM T
GROUP BY day
```

在这种情况下，建议使用 FILTER 语法, 目前的 Flink SQL 优化器可以识别同一唯一键上的不同 FILTER 参数。如，在上面的示例中，三个 COUNT DISTINCT 都作用在 user_id列上。此时，经过优化器识别后，Flink 可以只使用一个共享状态实例，而不是三个状态实例，可减少状态的大小和对状态的访问。

**将上边的 CASE WHEN 替换成 FILTER 后**，如下所示:

```sql
SELECT
    day,
    COUNT(DISTINCT user_id) AS total_uv,
    COUNT(DISTINCT user_id) FILTER (WHERE flag IN ('android', 'iphone')) AS app_uv,
    COUNT(DISTINCT user_id) FILTER (WHERE flag IN ('wap', 'other')) AS web_uv
FROM T
GROUP BY day
```



## 2、TopN 优化

### 2.1 使用最优算法

当 TopN 的输出是非更新流（例如 Source），TopN 只有一种算法 AppendRank。当TopN 的输出是更新流时（例如经过了 AGG/JOIN 计算），TopN 有 2 种算法，性能从高到低分别是：UpdateFastRank 和 RetractRank。算法名字会显示在拓扑图的节点名字上

**注意**：apache 社区版的 Flink1.12 目前还没有 UnaryUpdateRank，阿里云实时计算版 Flink才有



**UpdateFastRank ：最优算法**

需要具备 2 个条件：

1）输入流有 PK（Primary Key）信息，例如 Group BY AVG。

2）排序字段的更新是单调的，且单调方向与排序方向相反。例如，ORDER BYCOUNT/COUNT_DISTINCT/SUM（正数）DESC。

如果要获取到优化 Plan，则需要在使用 ORDER BY SUM DESC 时，添加 SUM 为正数的过滤条件。

```sql
select sum(col) filter ( WHERE col >= 0)  from table
```



**AppendFast：结果只追加，不更新**

**RetractRank：普通算法，性能差**

不建议在生产环境使用该算法。请检查输入流是否存在 PK 信息，如果存在，则可进行UpdateFastRank 优化。



### 2.2 无排名优化

**解决数据膨胀问题**

```sql
SELECT *
FROM (
    SELECT *,
    ROW_NUMBER() OVER ([PARTITION BY col1[, col2..]] ORDER BY col1 [asc|desc][, col2 [asc|desc]...]) AS rownum
    FROM table_name)
WHERE rownum <= N [AND conditions]
```

**数据膨胀问题：**

根据 TopN 的语法，rownum 字段会作为结果表的主键字段之一写入结果表。但是这可能导致数据膨胀的问题。例如，收到一条原排名 9 的更新数据，更新后排名上升到 1，则从 1 到 9 的数据排名都发生变化了，需要将这些数据作为更新都写入结果表。这样就产生了数据膨胀，导致结果表因为收到了太多的数据而降低更新速度。

**使用方式**

TopN 的输出结果无需要显示 rownum 值，仅需在最终前端显式时进行 1 次排序，极大地减少输入结果表的数据量。只需要在外层查询中将 rownum 字段裁剪掉即可

```sql
// 最外层的字段，不写 rownum
SELECT col1, col2, col3
FROM (
    SELECT col1, col2, col3
    ROW_NUMBER() OVER ([PARTITION BY col1[, col2..]] ORDER BY col1 [asc|desc][, col2 [asc|desc]...]) AS rownum
    FROM table_name)
WHERE rownum <= N [AND conditions]
```

在无 rownum 的场景中，对于结果表主键的定义需要特别小心。如果定义有误，会直接导致 TopN 结果的不正确。 无 rownum 场景中，主键应为 TopN 上游 GROUP BY 节点的 KEY 列表。



### 2.3 增加 TopN 的 Cache 大小

topN 为了提升性能有一个 State Cache 层，Cache 层能提升对 State 的访问效率。

TopN 的 Cache 命中率的计算公式为

```
cache_hit = cache_size*parallelism/top_n/partition_key_num
```

例如，Top100 配置缓存 10000 条，并发 50，当 PatitionBy 的 key 维度较大时，例如10 万级别时，Cache 命中率只有 10000*50/100/100000=5%，命中率会很低，导致大量的请求都会击中 State（磁盘），性能会大幅下降。因此当 PartitionKey 维度特别大时，可以适当加大 TopN 的 CacheS ize，相对应的也建议适当加大 TopN 节点的 Heap Memory。



**使用方式**

```java
// 设置参数：
//默认10000条，调整TopNcahce到20万，那么理论命中率能达到200000*50/100/100000 = 100%
configuration.setString("table.exec.topn.cache-size", "200000");
```

**注意**：目前源码中标记为实验项，官网中未列出该参数



### 2.4 PartitionBy 的字段中要有时间类字段

例如每天的排名，要带上 Day 字段。否则 TopN 的结果到最后会由于 State ttl 有错乱。



### 2.5 优化后的 SQL 示例

```sql
SELECT student_id,
       subject_id,
       stat_date,
       score 
       --不输出rownum字段，能减小结果表的输出量（无排名优化）
FROM (
     SELECT *,
            ROW_NUMBER() OVER (
                PARTITION BY student_id,
                    stat_date --注意要有时间字段，否则state过期会导致数据错乱（分区字段优化）
                ORDER
                    BY score DESC --根据上游sum结果排序。排序字段的更新是单调的，且单调方向与排序方向相反（走最优算法）
                ) AS rownum
     FROM (
              SELECT student_id,
                     subject_id,
                     stat_date,
                     --重点。声明Sum的参数都是正数，所以Sum的结果是单调递增的，因此TopN能使用优化算法，只获取前100个数据（走最优算法）
                     sum(score) filter ( WHERE score >= 0) AS score
              FROM score_test
              WHERE score >= 0
              GROUP BY student_id,
                       subject_id,
                       stat_date
          ) a
     WHERE rownum <= 100
     );
```



## 3、高效去重方案

由于 SQL 上没有直接支持去重的语法，还要灵活的保留第一条或保留最后一条。因此我们使用了 SQL 的 ROW_NUMBER OVER WINDOW 功能来实现去重语法。去重本质上是一种特殊的 TopN。

### 3.1 保留首行的去重策略

**Deduplicate Keep FirstRow**

保留 KEY 下第一条出现的数据，之后出现该 KEY 下的数据会被丢弃掉。因为 STATE 中只存储了 KEY 数据，所以性能较优

```sql
SELECT *
    FROM (
    SELECT *,
    ROW_NUMBER() OVER (PARTITION BY b ORDER BY proctime) as rowNum
    FROM T
)
WHERE rowNum = 1;
```

以上示例是将 T 表按照 b 字段进行去重，并按照系统时间保留第一条数据。Proctime在这里是源表 T 中的一个具有 Processing Time 属性的字段。如果按照系统时间去重，也可以将 Proctime 字段简化 PROCTIME()函数调用，可以省略 Proctime 字段的声明。



### 3.2 保留末行的去重策略

**Deduplicate Keep LastRow**

保留 KEY 下最后一条出现的数据。保留末行的去重策略性能略优于 LAST_VALUE 函数

```sql
SELECT *
FROM (
    SELECT
    	*,
    	ROW_NUMBER() OVER (PARTITION BY b, d ORDER BY rowtime DESC) as rowNum
    FROM T
)
WHERE rowNum = 1;
```

以上示例是将 T 表按照 b 和 d 字段进行去重，并按照业务时间保留最后一条数据。Rowtime 在这里是源表 T 中的一个具有 Event Time 属性的字段。



## 4、高效的内置函数

### 4.1 使用内置函数替换自定义函数

Flink 的内置函数在持续的优化当中，请尽量使用内部函数替换自定义函数。使用内置函数好处：

1）优化数据序列化和反序列化的耗时。

2）新增直接对字节单位进行操作的功能。

支持的系统内置函数：https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/table/functions/systemFunctions.html



### 4.2 LIKE 操作注意事项

- 如果需要进行 StartWith 操作，使用 LIKE 'xxx%'。
- 如果需要进行 EndWith 操作，使用 LIKE '%xxx'。
- 如果需要进行 Contains 操作，使用 LIKE '%xxx%'。
- 如果需要进行 Equals 操作，使用 LIKE 'xxx'，等价于 str = 'xxx'。
- 如果需要匹配 _ 字符，请注意要完成转义 LIKE '%seller/id%' ESCAPE '/'。_在 SQL中属于单字符通配符，能匹配任何字符。如果声明为 LIKE '%seller_id%'，则不单会匹配 seller_id 还会匹配 seller#id、sellerxid 或 seller1id 等，导致结果错误。



### 4.3 慎用正则函数

正则表达式是非常耗时的操作，对比加减乘除通常有百倍的性能开销，而且正则表达式在某些极端情况下可能会进入无限循环，导致作业阻塞。建议使用 LIKE。

正则函数包括：

- REGEXP
- REGEXP_EXTRACT
- REGEXP_REPLACE



## 5、指定时区

本地时区定义了当前会话时区 id。当本地时区的时间戳进行转换时使用。在内部，带有本地时区的时间戳总是以 UTC 时区表示。但是，当转换为不包含时区的数据类型时(例如TIMESTAMP, TIME 或简单的 STRING)，会话时区在转换期间被使用。为了避免时区错乱的问题，可以参数指定时区。

```java
// 获取 tableEnv 的配置对象
Configuration configuration = tEnv.getConfig().getConfiguration();

// 设置参数：
// 指定时区
configuration.setString("table.local-time-zone", "Asia/Shanghai");
```



## 6、设置参数总结

```java
// 获取 tableEnv 的配置对象
Configuration configuration = tEnv.getConfig().getConfiguration();

// 设置参数：
// 开启 miniBatch
configuration.setString("table.exec.mini-batch.enabled", "true");
// 批量输出的间隔时间
configuration.setString("table.exec.mini-batch.allow-latency", "5 s");
// 防止 OOM 设置每个批次最多缓存数据的条数，可以设为 2 万条
configuration.setString("table.exec.mini-batch.size", "20000");
// 开启 LocalGlobal
configuration.setString("table.optimizer.agg-phase-strategy", "TWO_PHASE");
// 开启 Split Distinct
configuration.setString("table.optimizer.distinct-agg.split.enabled", "true");
// 第一层打散的 bucket 数目
configuration.setString("table.optimizer.distinct-agg.split.bucket-num", "1024");
// TopN 的缓存条数
configuration.setString("table.exec.topn.cache-size", "200000");
// 指定时区
configuration.setString("table.local-time-zone", "Asia/Shanghai");
```

