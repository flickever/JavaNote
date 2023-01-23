# Flink调优

**调优要点**

做过什么优化？解决过什么问题？遇到哪些问题？

1. 说明业务场景
2. 说明遇到什么问题，往往需要结合工具报警系统
3. 问题排查过程
4. 解决手段



## 1、资源调优

资源并不是越多越好，选择与任务合适的资源即可。

例如有内存足够大的机器，如果使用map join去缓存大表，效率反而不一定高，原因有两个：

1. 每个机器都要下载一份数据，占用网络IO。
2. 申请资源耗时极长。



### 1.1 内存设置

```
bin/flink run \
-t yarn-per-job \
-d \
-p 5 \ 指定并行度
-Dyarn.application.queue=test \                  指定 yarn 队列
-Djobmanager.memory.process.size=2048mb \        JM2~4G 足够
-Dtaskmanager.memory.process.size=6144mb \       单个 TM2~8G 足够
-Dtaskmanager.numberOfTaskSlots=2 \              与容器核数 1core：1slot 或 1core：2slot
-c com.atguigu.app.dwd.LogBaseApp \
/opt/module/gmall-flink/gmall-realtime-1.0-SNAPSHOT-jar-with-dependencies.jar
```

on yarn模式需要配合container配置，尤其是`taskmanager内存` 和 `slot数`的设置

Flink 是实时流处理，关键在于资源情况能不能抗住高峰时期每秒的数据量，通常用QPS/TPS 来描述数据情况

- **QPS**：每秒请求数，就是说服务器在一秒的时间内处理了多少个请求
- **TPS**：每秒事务数



### 1.2 并行度

#### 1.2.1 最优并行度

先进行压测。任务并行度给 10 以下，测试单个并行度的处理上限。然后总 QPS/单并行度的处理能力 = 并行度；

例如测出单并行度最多20m/s，总QPS 100m/s的话，应设置5并行度；

不能只从 QPS 去得出并行度，因为有些字段少、逻辑简单的任务，单并行度一秒处理几万条数据。而有些数据字段多，处理逻辑复杂，单并行度一秒只能处理 1000 条数据。

最好根据高峰期的 QPS 压测，并行度*1.2  ~ 1.5 倍，富余一些资源。



#### 1.2.2 Source 端并行度

数据源端是 Kafka，Source 的并行度设置为 Kafka 对应 Topic 的分区数。

如果已经等于 Kafka 的分区数，消费速度仍跟不上数据生产速度，考虑下 Kafka 要扩大分区，同时调大并行度等于分区数。

Flink 的一个并行度可以处理一至多个分区的数据，如果并行度多于 Kafka 的分区数，那么就会造成有的并行度空闲，浪费资源。

**一般是作为最后手段使用**，因为kafka分区增加不可逆。



#### 1.2.3 Transform 端并行度

**Keyby 之前的算子**

一般不会做太重的操作，都是比如 map、filter、flatmap 等处理较快的算子，并行度可以和 source 保持一致。

**Keyby 之后的算子**

如果并发较大，建议设置并行度为 2 的整数次幂，例如：128、256、512；

小并发任务的并行度不一定需要设置成 2 的整数次幂；

大并发任务如果没有 KeyBy，并行度也无需设置为 2 的整数次幂；



#### 1.2.4 Sink 端并行度

**需要考虑下游Sink端的配置**

**如果 Sink 端是 Kafka，可以设为 Kafka 对应 Topic 的分区数**

另外：可以考虑批量写入方式



### 1.3 RocksDB 大状态调优

RocksDB 是基于 LSM Tree 实现的（类似 HBase），写数据都是先缓存到内存中，所以 RocksDB 的写请求效率比较高。RocksDB 使用内存结合磁盘的方式来存储数据，每次获取数据时，先从内存中 blockcache 中查找，如果内存中没有再去磁盘中查询。优化后差不多单并行度 TPS 5000 record/s，性能瓶颈主要在于 RocksDB 对磁盘的读请求，所以当处理性能不够时，仅需要横向扩展并行度即可提高整个 Job 的吞吐量。



以下几个调优参数

**设置本地 RocksDB 多目录**

在 flink-conf.yaml 中配置：

```ymal
state.backend.rocksdb.localdir:
/data1/flink/rocksdb,/data2/flink/rocksdb,/data3/flink/rocksdb
```

注意：

- 不要配置单块磁盘的多个目录，务必将目录配置到多块不同的磁盘上，让多块磁盘来分担压力。
- 当设置多个 RocksDB 本地磁盘目录时，Flink 会随机选择要使用的目录，所以就可能存在三个并行度共用同一目录的情况。
- 如果服务器磁盘数较多，一般不会出现该情况，但是如果任务重启后吞吐量较低，可以检查是否发生了多个并行度共用同一块磁盘的情况。

**state.backend.incremental：开启增量检查点，默认 false，改为 true。**

**state.backend.rocksdb.predefined-options：**SPINNING_DISK_OPTIMIZED_HIGH_MEM 设置为机械硬盘+内存模式，有条件上
SSD，指定为 FLASH_SSD_OPTIMIZED

**state.backend.rocksdb.block.cache-size:** 整 个 RocksDB 共 享 一 个 blockcache，读数据时内存的 cache 大小，该参数越大读数据时缓存命中率越高，默认大小为 8 MB，建议设置到 64 ~ 256 MB。

**state.backend.rocksdb.thread.num**: 用于后台 flush 和合并 sst 文件的线程数，默认为 1，建议调大，机械硬盘用户可以改为 4 等更大的值

**state.backend.rocksdb.writebuffer.size:** RocksDB 中，每个 State 使用一个Column Family，每个 Column Family 使用独占的 write buffer，建议调大，例如：32M

**state.backend.rocksdb.writebuffer.count**: 每 个 Column Family 对 应 的writebuffer 数目，默认值是 2，对于机械磁盘来说，如果内存⾜够大，可以调大到 5左右

**state.backend.rocksdb.writebuffer.number-to-merge**: 将数据从 writebuffer中 flush 到磁盘时，需要合并的 writebuffer 数量，默认值为 1，可以调成 3。

**state.backend.local-recovery:** 设置本地恢复，当 Flink 任务失败时，可以基于本地的状态信息进行恢复任务，可能不需要从 hdfs 拉取数据



### 1.4 Checkpoint 设置

Checkpoint 时间间隔可以设置为分钟级别，例如 1 分钟、3 分钟，对于状态很大的任务每次 Checkpoint 访问 HDFS 比较耗时，可以设置为 5~10 分钟一次Checkpoint，并且调大两次 Checkpoint 之间的暂停间隔，例如设置两次 Checkpoint 之间至少暂停 4 或 8 分钟。

如果 Checkpoint 语义配置为 EXACTLY_ONCE，那么在 Checkpoint 过程中还会存在 barrier 对齐的过程，可以通过 Flink Web UI 的 Checkpoint 选项卡来查看Checkpoint 过程中各阶段的耗时情况，从而确定到底是哪个阶段导致 Checkpoint 时间过长然后针对性的解决问题。

```java
// 使⽤ RocksDBStateBackend 做为状态后端，并开启增量 Checkpoint
RocksDBStateBackend rocksDBStateBackend = new RocksDBStateBackend("hdfs://hadoop102:8020/flink/checkpoints", true);
env.setStateBackend(rocksDBStateBackend);

// 开启 Checkpoint，间隔为 3 分钟
env.enableCheckpointing(TimeUnit.MINUTES.toMillis(3));
// 配置 Checkpoint
CheckpointConfig checkpointConf = env.getCheckpointConfig();
checkpointConf.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
// 最小间隔 4 分钟
checkpointConf.setMinPauseBetweenCheckpoints(TimeUnit.MINUTES.toMillis(4))
// 超时时间 10 分钟
checkpointConf.setCheckpointTimeout(TimeUnit.MINUTES.toMillis(10));
// 保存 checkpoint
checkpointConf.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```





### 1.5 ParameterTool

在 Flink 中可以通过使用 ParameterTool 类读取配置，它可以读取环境变量、运行参数、配置文件。**防止修改配置后重新打包运行**；

ParameterTool 是可序列化的，所以你可以将它当作参数进行传递给算子的自定义函数类。

#### 1.5.1 读取运行参数

```java
ParameterTool parameterTool = ParameterTool.fromArgs(args);
String myJobname = parameterTool.get("jobname"); //参数名对应
env.execute(myJobname);
```



#### 1.5.2 读取系统属性

ParameterTool 还⽀持通过 ParameterTool.fromSystemProperties() 方法读取系统属性。做个打印

```java
ParameterTool parameterTool = ParameterTool.fromSystemProperties();
System.out.println(parameterTool.toMap().toString());
```



#### 1.5.3 读取配置文件

可 以 使 用ParameterTool.fromPropertiesFile("/application.properties") 读 取properties 配置文件。

可以将所有要配置的地方（比如并行度和一些 Kafka、MySQL 等配置）都写成可配置的，然后其对应的 key 和 value 值都写在配置文件中，最后通过ParameterTool 去读取配置文件获取对应的值。



#### 1.5.4 注册全局参数

在 ExecutionConfig 中可以将 ParameterTool 注册为全作业参数的参数，这样就可以被 JobManager 的 web 端以及用户⾃定义函数中以配置值的形式访问。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(ParameterTool.fromArgs(args));
```

可以不用将 ParameterTool 当作参数传递给算子的自定义函数，直接在用户⾃定义的Rich 函数中直接获取到参数值了。

```java
env.addSource(new RichSourceFunction() {
    
    @Override
    public void run(SourceContext sourceContext) throws Exception {
        while (true) {
            ParameterTool parameterTool =(ParameterTool) getRuntimeContext().getExecutionConfig().getGlobalJobParameters();
        }
    }
    
    @Override
    public void cancel() {
    }
})
```



### 1.6 压测方式

压测的方式很简单，先在 kafka 中积压数据，之后开启 Flink 任务，出现反压，就是处理瓶颈。相当于水库先积水，一下子泄洪。数据可以是自己造的模拟数据，也可以是生产中的部分数据。



## 2、反压处理

反压（BackPressure）通常产生于这样的场景：短时间的负载高峰导致系统接收数据的速率远高于它处理数据的速率。

场景：垃圾回收停顿、大促、秒杀活动导致流量陡增。反压如果不能得到正确的处理，可能会导致资源耗尽甚至系统崩溃。

反压机制是指系统能够自己检测到被阻塞的 Operator，然后自适应地降低源头或上游数据的发送速率，从而维持整个系统的稳定。

Flink 任务一般运行在多个节点上，数据从上游算子发送到下游算子需要网络传输，若系统在反压时想要降低数据源头或上游算子数据的
发送速率，那么肯定也需要网络传输。所以下面先来了解一下 Flink 的网络流控（Flink 对网络数据流量的控制）机制。



### 2.1 反压现象及定位

**排查的时候，先把 operator chain 禁用，方便定位**

Flink 的反压太过于天然了，导致无法简单地通过监控 BufferPool 的使用情况来判断反压状态。Flink 通过对运行中的任务进行采样来确定其反压，如果一个 Task 因为反压导致处理速度降低了，那么它肯定会卡在向 LocalBufferPool 申请内存块上。那么该 Task 的 stack trace 应该是这样：

```
java.lang.Object.wait(Native Method)
o.a.f.[...].LocalBufferPool.requestBuffer(LocalBufferPool.java:163)
o.a.f.[...].LocalBufferPool.requestBufferBlocking(LocalBufferPool.java:133) [...]
```

监控对正常的任务运行有一定影响，因此只有当Web页面切换到Job的**BackPressure**页面时，JobManager才会对该Job触发反压监控。

或者在Metrics界面，查看每个算子task的`inPoolUsage`和`outPoolUsage`情况。

默认情况下，JobManager会触发100次stacktrace采样，每次间隔50ms来确定反压。Web界面看到的比率表示在内部方法调用中有多少stacktrace被卡在LocalBufferPool.requestBufferBlocking()，例如:0.01表示在100个采样中只有1个被卡在LocalBufferPool.requestBufferBlocking()。

采样得到的比例与反压状态的对应关系如下:

- OK: 0 <= 比例 <= 0.10
- LOW: 0.10 < 比例 <= 0.5
- HIGH: 0.5 < 比例 <= 1

Task 的状态为 OK 表示没有反压，HIGH 表示这个 Task 被反压。

**以上是Flink Web界面定位方法，是利用了监控工具，flink自带的普罗米修斯来监控。**



### 2.2 反压的原因及处理

下面列出从最基本到比较复杂的一些反压潜在原因

**注意：**反压可能是暂时的，可能是由于负载高峰、CheckPoint 或作业重启引起的数据积压而导致反压。如果反压是暂时的，应该忽略它。另外，请记住，断断续续的反压会影响我们分析和解决问题。

#### 2.2.1 系统资源

检查涉及服务器基本资源的使用情况，如 CPU、网络或磁盘 I/O，目前 Flink 任务使用最主要的还是内存和 CPU 资源，本地磁盘、依赖的外部存储资源以及网卡资源一般都不会是瓶颈。如果某些资源被充分利用或大量使用，可以借助分析工具，分析性能瓶颈（JVM Profiler+ FlameGraph 生成火焰图）

如何生成火焰图：http://www.54tianzhisheng.cn/2020/10/05/flink-jvm-profiler/

如何读懂火焰图：https://zhuanlan.zhihu.com/p/29952444

- 针对特定的资源调优 Flink
- 通过增加并行度或增加集群中的服务器数量来横向扩展
- 减少瓶颈算子上游的并行度，从而减少瓶颈算子接收的数据量（不建议，可能造成整个Job 数据延迟增大）



#### 2.2.2 垃圾收集（GC）

长 时 间 GC 暂 停 会 导 致 性 能 问 题 。 可 以 通 过 打 印 调 试 GC 日 志 （ 通 过-XX:+PrintGCDetails）或使用某些内存或 GC 分析器（GCViewer 工具）来验证是否处于这种情况。

- 在 Flink 提交脚本中,设置 JVM 参数，打印 GC 日志：

```shell
bin/flink run \
-t yarn-per-job \
-d \
-p 5 \ 指定并行度
-Dyarn.application.queue=test \ 指定 yarn 队列
-Djobmanager.memory.process.size=1024mb \ 指定 JM 的总进程大小
-Dtaskmanager.memory.process.size=1024mb \ 指定每个 TM 的总进程大小
-Dtaskmanager.numberOfTaskSlots=2 \ 指定每个 TM 的 slot 数
-Denv.java.opts="-XX:+PrintGCDetails -XX:+PrintGCDateStamps"
-c com.atguigu.app.dwd.LogBaseApp \
/opt/module/gmall-flink/gmall-realtime-1.0-SNAPSHOT-jar-with-dependencies.jar
```

- ###### 下载 GC 日志的方式：

因为是 on yarn 模式，运行的节点一个一个找比较麻烦。可以打开 WebUI，选择JobManager 或者 TaskManager，点击 Stdout，即可看到 GC 日志，点击下载按钮即可将 GC 日志通过 HTTP 的方式下载下来。

- 分析 GC 日志：

通过 GC 日志分析出单个 Flink Taskmanager 堆总大小、年轻代、老年代分配的内存空间、Full GC 后老年代剩余大小等，相关指标定义可以去 Github 具体查看。

GCViewer 地址：https://github.com/chewiebug/GCViewer

最重要的指标是 Full GC 后，老年代剩余大小这个指标，按照《Java 性能优化权威指南》这本书 Java 堆大小计算法则，设 Full GC 后老年代剩余大小空间为 M，那么堆的大小建议 3 ~ 4 倍 M，新生代为 1 ~ 1.5 倍 M，老年代应为 2 ~ 3 倍 M。



#### 2.2.3 CPU/线程瓶颈

有时，一个或几个线程导致 CPU 瓶颈，而整个机器的 CPU 使用率仍然相对较低，则可能无法看到 CPU 瓶颈。例如，48 核的服务器上，单个 CPU 瓶颈的线程仅占用 2％的CPU 使用率，就算单个线程发生了 CPU 瓶颈，我们也看不出来。可以考虑使用 2.2.1 提到的分析工具，它们可以显示每个线程的 CPU 使用情况来识别热线程。



#### 2.2.4 线程竞争

与上⾯的 CPU/线程瓶颈问题类似，subtask 可能会因为共享资源上高负载线程的竞争而成为瓶颈。同样，可以考虑使用 2.2.1 提到的分析工具，考虑在用户代码中查找同步开销、锁竞争，尽管避免在用户代码中添加同步。



#### 2.2.5 负载不平衡

如果瓶颈是由数据倾斜引起的，可以尝试通过将数据分区的 key 进行加盐或通过实现本地预聚合来减轻数据倾斜的影响。（关于数据倾斜的详细解决方案，会在下一章节详细讨论）



#### 2.2.6 外部依赖

如果发现我们的 Source 端数据读取性能比较低或者 Sink 端写入性能较差，需要检查第三方组件是否遇到瓶颈。例如，Kafka 集群是否需要扩容，Kafka 连接器是否并行度较低，HBase 的 rowkey 是否遇到热点问题。关于第三方组件的性能问题，需要结合具体的组件来分析。



**综上是三种解决方案：**

1. 系统资源
2. 数据倾斜
3. 旁路缓存 + 异步IO



## 3、数据倾斜

### 3.1 判断是否存在数据倾斜

从Web UI界面，查看每个SubTask处理了多大数据量，如果不均匀就是数据倾斜



### 3.2 数据倾斜的解决

#### 3.2.1 keyBy 之前发生数据倾斜

如果 keyBy 之前就存在数据倾斜，上游算子的某些实例可能处理的数据较多，某些实例可能处理的数据较少，产生该情况可能是因为数据源的数据本身就不均匀，例如由于某些原因 Kafka 的 topic 中某些 partition 的数据量较大，某些 partition 的数据量较少。对于不存在 keyBy 的 Flink 任务也会出现该情况。

这种情况，需要让 Flink 任务强制进行 shuffle。使用 shuffle、**rebalance（最优）** 或 rescale算子即可将数据均匀分配，从而解决数据倾斜的问题。

| 分区策略                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| dataStream.partitionCustom(partitioner, “someKey”); dataStream.partitionCustom(partitioner, 0); | 根据指定的字段进行分区，指定字段值相同的数据发送到同一个 Operator 实例处理 |
| dataStream.shuffle();                                        | 将数据随机地分配到下游 Operator 实例                         |
| dataStream.rebalance();                                      | 使用轮循的策略将数据发送到下游 Operator 实例                 |
| dataStream.rescale();                                        | 基于 rebalance 优化的策略，依然使用轮循策略，但仅仅是 TaskManager 内的轮循，只会在 TaskManager 本地进行 shuffle 操作，减少了网络传输 |
| dataStream.broadcast();                                      | 将数据广播到下游所有的 Operator 实例                         |



#### 3.2.2 keyBy 后的聚合操作存在数据倾斜

**使用 LocalKeyBy 的思想**

在 keyBy 上游算子数据发送之前，首先在上游算子的本地对数据进行聚合后再发送到下游，使下游接收到的数据量大大减少，从而使得 keyBy 之后的聚合操作不再是任务的瓶颈。类似 MapReduce 中 Combiner 的思想，但是这要求聚合操作必须是多条数据或者一批数据才能聚合，单条数据没有办法通过聚合来减少数据量。从 Flink LocalKeyBy 实现原理来讲，必然会存在一个积攒批次的过程，在上游算子中必须攒够一定的数据量，对这些数据聚合后再发送到下游。

**注意**：Flink 是实时流处理，如果 keyby 之后的聚合操作存在数据倾斜，且没有开窗口的情况下，简单的认为使用两阶段聚合，是不能解决问题的。因为这个时候 Flink 是来一条处理一条，且向下游发送一条结果，对于原来 keyby 的维度（第二阶段聚合）来讲，数据量并没有减少，且结果重复计算（**非 FlinkSQL，未使用回撤流**）

**实现方式**：以计算 PV 为例，keyby 之前，使用 flatMap 实现 LocalKeyby

```java
class LocalKeyByFlatMap extends RichFlatMapFunction<String, Tuple2<String, Long>> implements CheckpointedFunction {

    //Checkpoint 时为了保证 Exactly Once，将 buffer 中的数据保存到该 ListState 中
    private ListState<Tuple2<String, Long>> localPvStatListState;

    //本地 buffer，存放 local 端缓存的 app 的 pv 信息
    private HashMap<String, Long> localPvStat;

    //缓存的数据量大小，即：缓存多少数据再向下游发送
    private int batchSize;

    //计数器，获取当前批次接收的数据量
    private AtomicInteger currentSize;

    LocalKeyByFlatMap(int batchSize){
        this.batchSize = batchSize;
    }

    @Override
    public void flatMap(String in, Collector collector) throws Exception {
        //  将新来的数据添加到 buffer 中
        Long pv = localPvStat.getOrDefault(in, 0L);
        localPvStat.put(in, pv + 1);

        // 如果到达设定的批次，则将 buffer 中的数据发送到下游
        if(currentSize.incrementAndGet() >= batchSize){
            // 遍历 Buffer 中数据，发送到下游
            for(Map.Entry<String, Long> appIdPv: localPvStat.entrySet()) {
                collector.collect(Tuple2.of(appIdPv.getKey(), appIdPv.getValue()));
            }
            // Buffer 清空，计数器清零
            localPvStat.clear();
            currentSize.set(0);
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext functionSnapshotContext) {
        // 将 buffer 中的数据保存到状态中，来保证 Exactly Once
        localPvStatListState.clear();
        for(Map.Entry<String, Long> appIdPv: localPvStat.entrySet()) {
            localPvStatListState.add(Tuple2.of(appIdPv.getKey(), appIdPv.getValue()));
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) {
        // 从状态中恢复 buffer 中的数据
        localPvStatListState = context.getOperatorStateStore().getListState(
                new ListStateDescriptor<>("localPvStat",
                        TypeInformation.of(new TypeHint<Tuple2<String, Long>>() {
                        })));
        localPvStat = new HashMap();
        if(context.isRestored()) {
            // 从状态中恢复数据到 localPvStat 中
            for(Tuple2<String, Long> appIdPv: localPvStatListState.get()){
                localPvStat.put(appIdPv.f0, appIdPv.f1);
            }
            //  从状态恢复时，默认认为 buffer 中数据量达到了 batchSize，需要向下游发送数据了
            currentSize = new AtomicInteger(batchSize);
        } else {
            currentSize = new AtomicInteger(0);
        }
    }
}
```



#### 3.2.3 keyBy 后的窗口聚合操作存在数据倾斜

因为使用了窗口，变成了有界数据的处理（3.2.2 已分析过），窗口默认是触发时才会输出一条结果发往下游，所以可以使用两阶段聚合的方式。

实现思路：

**第一阶段聚合**：key 拼接随机数前缀或后缀，进行 keyby、开窗、聚合

注意：聚合完不再是 WindowedStream，要获取 WindowEnd 作为窗口标记作为第二阶段分组依据，避免不同窗口的结果聚合到一起）

**第二阶段聚合**：去掉随机数前缀或后缀，按照原来的 key 及 windowEnd 作 keyby、聚合



## 4、KafkaSource 调优

### 4.1 动态发现分区

FlinkKafkaConsumer 初始化时，每个 subtask 会订阅一批 partition，但是当Flink 任务运行过程中，如果被订阅的 topic 创建了新的 partition，FlinkKafkaConsumer如何实现动态发现新创建的 partition 并消费呢？

在使用 FlinkKafkaConsumer 时，可以开启 partition 的动态发现。通过 Properties指定参数开启（单位是毫秒）：

```
FlinkKafkaConsumerBase.KEY_PARTITION_DISCOVERY_INTERVAL_MILLIS
```

该参数表示间隔多久检测一次是否有新创建的 partition。默认值是 Long 的最小值，表示不开启，大于 0 表示开启。开启时会启动一个线程根据传入的 interval 定期获取 Kafka最新的元数据，新 partition 对应的那一个 subtask 会自动发现并从 earliest 位置开始消费，新创建的 partition 对其他 subtask 并不会产生影响。

```java
properties.setProperty(FlinkKafkaConsumerBase.KEY_PARTITION_DISCOVERY_INTERVAL_MILLIS, 30 * 1000 + "");
```



### 4.2 从 Kafka 数据源生成 watermark

Kafka 单分区内有序，多分区间无序。在这种情况下，可以使用 Flink 中可识别 Kafka分区的 watermark 生成机制。使用此特性，将在 Kafka 消费端内部针对每个 Kafka 分区生成 watermark，并且不同分区 watermark 的合并方式与在数据流 shuffle 时的合并方式相同

在单分区内有序的情况下，使用时间戳单调递增按分区生成的 watermark 将生成完美的全局 watermark。

```java
FlinkKafkaConsumer<String> kafkaSourceFunction = new FlinkKafkaConsumer<>(
    "flinktest",
    new SimpleStringSchema(),
    properties
);

kafkaSourceFunction.assignTimestampsAndWatermarks(
    WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofMinutes(2))
);

env.addSource(kafkaSourceFunction);
```



### 4.3 设置空闲等待

如 果 数据 源中 的某 一个分 区 / 分片 在 一段 时间内 未 发送 事件 数据 ，则意 味 着WatermarkGenerator 也不会获得任何新数据去生成 watermark。我们称这类数据源为空闲输入或空闲源。在这种情况下，当某些其他分区仍然发送事件数据的时候就会出现问题。比如 Kafka 的 Topic 中，由于某些原因，造成个别 Partition 一直没有新的数据。

由于下游算子 watermark 的计算方式是取所有不同的上游并行数据源 watermark的最小值，则其 watermark 将不会发生变化，导致窗口、定时器等不会被触发。

为了解决这个问题，你可以使用 WatermarkStrategy 来检测空闲输入并将其标记为空闲状态。

**Kafka已经默认加了这个参数**

```java
kafkaSourceFunction.assignTimestampsAndWatermarks(
    WatermarkStrategy
        .forBoundedOutOfOrderness(Duration.ofMinutes(2))
    	// 如果有并行度5分钟没有watermark，下游就不会考虑这个并行度
        .withIdleness(Duration.ofMinutes(5))
);
```



### 4.4 offset 消费策略

FlinkKafkaConsumer可以调用以下API，注意与”auto.offset.reset”区分开：

1. setStartFromGroupOffsets()：默认消费策略，默认读取上次保存的offset信息，如果是应用第一次启动，读取不到上次的offset信息，则会根据这个参数auto.offset.reset的值来进行消费数据。建议使用这个。
2. setStartFromEarliest()：从最早的数据开始进行消费，忽略存储的offset信息
3. setStartFromLatest()：从最新的数据进行消费，忽略存储的offset信息
4. setStartFromSpecificOffsets(Map)：从指定位置进行消费 （Key -> topic partition, value -> long值）
5. setStartFromTimestamp(long)：从topic中指定的时间点开始消费，指定时间点之前的数据忽略
6. 当checkpoint机制开启的时候，KafkaConsumer会定期把kafka的offset信息还有其他operator的状态信息一块保存起来。当job失败重启的时候，Flink会从最近一次的checkpoint中进行恢复数据，重新从保存的offset消费kafka中的数据（也就是说，上面几种策略，只有第一次启动的时候起作用）。
7. 为了能够使用支持容错的kafkaConsumer，需要开启checkpoint，例如如果checkpoint没开启，设置消费策略**EXACTLY_ONCE**没有用
