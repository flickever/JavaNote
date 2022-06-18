# 02_DWD-DIM 层

| 分层 | 数据描述                                                     | 生成计算工具         | 存储媒介   |
| ---- | ------------------------------------------------------------ | -------------------- | ---------- |
| ODS  | 原始数据，日志和业务数据                                     | 日志服务器，FlinkCDC | Kafka      |
| DWM  | 对于部分数据对象进行进一步加工，比如独立访问、跳出行为。依旧是明细数据 | Flink                | Kafka      |
| DIM  | 维度数据                                                     | Flink                | HBase      |
| DWS  | 根据某个维度主题将多个事实数据轻度聚合，形成主题宽表         | Flink                | Clickhouse |
| ADS  | 把 Clickhouse 中的数据根据可视化需要进行筛选聚合             | Clickhouse           | 可视化展示 |

## 1. log数据

在Spark中，想要把一条流分成多条流只能通过fliter算子过滤三次

但是在Flink中，可以使用侧输出流

日志数据分为三类： 页面日志、启动日志和曝光日志



## 2. db数据

### 2.1 动态分流功能

由于会同步很多张表，最好不要每个表都写一个任务

由于每个表有不同的特点，有些表是维度表，有些表是事实表

在实时计算中一般把维度数据写入存储容器，一般是方便通过主键查询的数据库比如HBase,Redis,MySQL 等。一般把事实数据写入流中，进行进一步处理，最终形成宽表

业务表会有新增现象，如果想让表刚建立就纳入监控，且中间不需要停止任务修改代码，需要一种动态配置方案，把这种配置长期保存起来，一旦配置有变化，实时计算可以自动感知

**三种实现方法**

➢ 一种是用 Zookeeper 存储，通过 Watch 感知数据变化； 

➢ 另一种是用 mysql 数据库存储，周期性的同步； 

➢ 另一种是用 mysql 数据库存储，使用广播流。



这里采用第三个方式

因为表的不同操作类型数据可能会插入不同的表，所以设置主键 来源表 + 操作类型

如果发现业务表有新建维度表，就把在HBase中建立一个同名表

建表语句：

```sql
use gmall-realtime;
CREATE TABLE `table_process` (
`source_table` varchar(200) NOT NULL COMMENT '来源表',
`operate_type` varchar(200) NOT NULL COMMENT '操作类型 insert,update,delete',
`sink_type` varchar(200) DEFAULT NULL COMMENT '输出类型 hbase kafka',
`sink_table` varchar(200) DEFAULT NULL COMMENT '输出表(主题)',
`sink_columns` varchar(2000) DEFAULT NULL COMMENT '输出字段',
`sink_pk` varchar(200) DEFAULT NULL COMMENT '主键字段',
`sink_extend` varchar(200) DEFAULT NULL COMMENT '建表扩展',
PRIMARY KEY (`source_table`,`operate_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```



**两个流的处理办法**

广播流

- 解析配置表信息
- 校验Phoenix中维表是否存在，如果没有，就在Phoenix中建表
- 写入状态，广播出去

主流

- 获取广播流配置数据
- 过滤字段，分流写到Kafka

**广播配置流比主流晚怎么办？**
会导致主流数据写不到Phoenix中
解决办法：在主流的open方法中先读加载一下配置信息，写到Map中，Map读不到的话，就表示没有相应的配置数据


## 3. DWD-DIM

**行为数据：DWD（Kafka）**

1. 过滤脏数据 --> 侧输出流、脏数据率
2. 新老用户校验 --> 前台校验不准
3. 分流 --> 侧输出流，页面、启动、曝光、动作、错误
4. 写入Kafka

**业务数据：DWD（Kafka）-->  DIM（Phoenix）**

1. 过滤数据 -->  去掉删除数据
2. 读取配置表创建广播流
3. 连接主流广播流
   - 广播流数据：解析流、建表、写入状态广播
   - 主流数据：读取状态、过滤字段、分流（添加SinkTable字段）
4. HBase流：自定义Sink
5. Kafka流：自定义序列化Function





## 4. 优化

### 4.1 旁路缓存

由于读取HBase数据有延迟，所以数据放到redis里面缓存一份，首次查询到的数据先缓存到redis中

**为什么不一开始就用redis？**

因为HBase存的数据量大，成本较低

**注意点**

1. 设置redis缓存过期时间
2. HBase数据修改后，及时清掉redis缓存，保证数据一致性

**一般顺序**

本地缓存 > redis缓存 > HBase

**Redis设计**

1. 存什么数据：json

2. 使用什么类型：String（table + id）

3. 为什么不用hash

   - 以下假设建立在数据量比较大的情况下

   - 数据量大，需要用string结构把key打散分布在redis集群里，防止热点key都在一个服务器中
   - 需要针对key设置过期时间，而不是针对hash。每条key过期时间都不一样，防止缓存雪崩



### 4.2 异步IO

要求数据库提供异步请求客户端（主流数据库都提供）

**或者自己实现客户端**，利用线程池的方式

因为要同时查询HBase和Redis，所以自己实现异步IO

**用法**

1. 实现异步方法
2. 响应请求
3. 处理完结果写回流中



