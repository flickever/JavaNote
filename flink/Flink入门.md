# Flink学习笔记

### Source API

1. **获取当前执行程序的上下文**

   ```scala
   val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
   ```

注：

1. flink会自己判断是否为开发环境，如果是开发环境，自动执行以下方法

   ```scala
   val env = StreamExecutionEnvironment.createLocalEnvironment(1)
   ```

2. 如果是远程环境，执行以下方法

   ```scala
   val env = ExecutionEnvironment.createRemoteEnvironment("jobmanage-hostname", 6123,"YOURPATH//wordcount.jar")
   ```



2. **从集合读取数据**

   ```scala
   env.fromCollection(List(...))
   
   env.fromElements("Hello",1,3.14)   // 类型不限
   ```

   并行度大于1的时候，print顺序不一样

   

3. **从文件读数据**

   ```scala
   env.readTextFile(filePath)
   ```

   

4. **从Kafka读取数据**

   pom文件如下

   ```xml
   <dependency>
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-connector-kafka-0.11_2.12</artifactId> 
       <version>1.10.1</version>
   </dependency>
   ```

   代码如下：

   ```scala
   val properties = new Properties()
   properties.setProperty("bootstrap.servers", "localhost:9092") 
   properties.setProperty("group.id", "consumer-group") 
   properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer") properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer") properties.setProperty("auto.offset.reset", "latest") 
   
   val stream3 = env.addSource(new FlinkKafkaConsumer011[String]("sensor", new SimpleStringSchema(), properties))
   
   // "sensor" ： Kafka的Topic
   // SimpleStringSchema ：反序列化类
   // properties ： 配置项
   ```

   要求addSource里面的方法继承SourceFunction

   

5. **自定义Source**

   ```scala
   class MySensorSource extends SourceFunction[SensorReading]{
   
     var running = true
   
     override def run(ctx: SourceFunction.SourceContext[SensorReading]): Unit = {
         val list = List(
           SensorReading("1",123456,12.3),
           SensorReading("2",123457,22.3),
           SensorReading("3",123458,32.3)
         )
   
         list.foreach(
           ctx.collect(_)
         )
         
         Thread.sleep(100)
       
     }
   
     override def cancel(): Unit = running = false
   }
   
   
   env.addSource(new MySensorSource)
   ```





### Transform API

1. **map**

   ```scala
   val streamMap = stream.map { x => x * 2 }
   ```

   

2. **flatMap**：多个List压碎成为一个List

   ```scala
   val streamFlatMap = stream.flatMap{ x => x.split(" ") }
   ```



3. **Filter**

   ```scala
   val streamFilter = stream.filter{ x => x == 1 }
   ```




4. **KeyBy**

   ```scala
   val keyedStream = stream.keyBy(0)
   
   val keyedStream = stream.keyBy("id")
   ```



5. **minBy**

   ```scala
   val streamMin = stream.keyBy("id").min("temperature")
   val streamMin = stream.keyBy("id").min(1)
   
   val streamMinBy = stream.keyBy("id").minBy("temperature")
   val streamMinBy = stream.keyBy("id").minBy(1)
   
   // min和minBy区别
   // min：其他数据不考虑，只更新min判断的字段
   // minBy：取minBy判断的字段所在的整条数据
   ```



6.  **Reduce**

   合并当前的元素和上次聚合的结果，产生一个新的值，返回的流中包含每一次聚合的结果，而不是 只返回最后一次聚合的最终结果

   ```scala
   val streamReduce = stream.keyBy("id")
   						 .reduce( (x, y) => SensorReading(x.id, x.timestamp + 1, y.temperature) )
   ```

   

7. **Split** **和** **Select**

   Split：根据某些特征把一个 DataStream 拆分成两个或者多个 DataStream

   Select：从一个 SplitStream 中获取一个或者多个DataStream

   

   Split不是真正意义上的拆开，只是逻辑划分

   ```scala
   val streamSplit = stream.split(
       sensorData => { if (sensorData.temperature > 30) Seq("high") else Seq("low") } 
   	) 
   
   val high = splitStream.select("high") 
   val low = splitStream.select("low") 
   val all = splitStream.select("high", "low")
   ```



8. **Connect** **和** **CoMap**

   **Connect** ：连接两个保持他们类型的数 据流，两个数据流被 Connect 之后，只是被放在了一个同一个流中，内部依然保持各自的数据和形式不发生任何变化，两个流相互独立

   connect经常被应用在对一个数据流使用另外一个流进行控制处理的场景上。

   **CoMap**：作用于 ConnectedStreams 上，功能与 map 和 flatMap 一样，对 ConnectedStreams 中的每一个 Stream 分别进行 map 和 flatMap处理

   其中流的类型就算不一样，一样可以conncet起来，得到的流内部还是多个独立的流

   多个不同类型的connect流，coMap返回的结果需要是同一类型的流
   
   ```scala
   val warning = high.map( sensorData => (sensorData.id, sensorData.temperature) ) 
   
   val connected = warning.connect(low) 
   
   // 虽然coMap内部类型不一样
   // 但是他们都是元组类型，所以返回的dataStream泛型是Product就行了，Product是元组的父类
   // 实在不行还可以是DataStream[Any]类型
   val coMap = connected.map( 
       warningData => (warningData._1, warningData._2, "warning"), 
       lowData => (lowData.id, "healthy") 
   )
   ```



9. **Union**

   对两个或者两个以上的 DataStream 进行 union 操作，产生一个包含所有 DataStream 元素的新 DataStream。

   1. Union 之前两个流的类型必须是一样，Connect 可以不一样，在之后的 coMap中再去调整成为一样的。 

   2.  Connect 只能操作两个流，Union 可以操作多个

   ```scala
   val unionStream: DataStream[StartUpLog] = appStoreStream.union(otherStream, otherStream2, ...)
   ```





### Sink API

Flink底层不像Rdd那样是个数据集，所以要调用Sink方法来写入数据

1. 简单写入

   ```scala
   stream.writeAsCsv("filePath")    // 写为CSV文件，但是此方法已经弃用，Flink推荐addSink方法
   ```


2. 推荐写入，上述方法已废弃

   ```scala
   stream.addSink(
   	StreamingFileSink.forRowFormat(
       	new Path("File Path")，
           new SimpleStringEncoder[SenorReading]()  // 调用其中的toString方法
       ).build()
   )
   ```

3. 集成Kafka

   pom文件：

   ```xml
   <dependency>
       <groupId>org.apache.flink</groupId> 
       <artifactId>flink-connector-kafka-0.11_2.12</artifactId>
    	<version>1.10.1</version> 
   </dependency>
   ```

   代码：

   ```scala
   stream.addSink(
       new FlinkKafkaProducer011[String](
           "localhost:9092",
           "test",
           new SimpleStringSchema()
       )
   )
   ```

4. 集成Redis

   pom

   ```xml
   <dependency> 
       <groupId>org.apache.bahir</groupId>
       <artifactId>flink-connector-redis_2.11</artifactId> 
       <version>1.0</version>
   </dependency>
   ```

   代码

   1. 定义一个 redis 的 mapper 类，用于定义保存到 redis 时调用的命令：

   ```scala
   class MyRedisMapper extends RedisMapper[SensorReading]{ 
       override def getCommandDescription: RedisCommandDescription = {
           new RedisCommandDescription(RedisCommand.HSET, "sensor_temperature") 
       }
       override def getValueFromData(t: SensorReading): String = t.temperature.toString
       override def getKeyFromData(t: SensorReading): String = t.id 
   }
   
   ```

   2. 主函数调用

   ```scala
   val conf = newFlinkJedisPoolConfig.Builder()
   	.setHost("localhost")
   	.setPort(6379)
   	.build() 
   
   dataStream.addSink( new RedisSink[SensorReading](conf, new MyRedisMapper) )
   ```

5. 集成Elasticsearch

   pom

   ```xml
   <dependency> 
       <groupId>org.apache.flink</groupId>
       <artifactId>flink-connector-elasticsearch6_2.12</artifactId>
       <version>1.10.1</version> 
   </dependency>
   ```

   代码

   ```scala
   val httpHosts = new util.ArrayList[HttpHost]() 
   httpHosts.add(new HttpHost("localhost", 9200)) 
   
   
   val esSinkBuilder = new ElasticsearchSink.Builder[SensorReading]( 
       httpHosts,
       new ElasticsearchSinkFunction[SensorReading] {
           override def process(t: SensorReading, runtimeContext: RuntimeContext, requestIndexer: RequestIndexer): Unit = { 
               println("saving data: " + t) val json = new util.HashMap[String, String]()
               json.put("data", t.toString)    
               val indexRequest = Requests.indexRequest().index("sensor").`type`("readingData").source(json) 
               requestIndexer.add(indexRequest) 
               println("saved successfully") 
           } 
       } 
   ) 
   
   
   dataStream.addSink( esSinkBuilder.build() )
   ```

6. JDBC自定义Sink

   pom

   ```xml
   <dependency> 
       <groupId>mysql</groupId>
       <artifactId>mysql-connector-java</artifactId> 
       <version>5.1.44</version>
   </dependency>
   ```

   代码

   ```scala
   class MyJdbcSink() extends RichSinkFunction[SensorReading]{ 
       var conn: Connection = _ 
       var insertStmt: PreparedStatement = _ 
       var updateStmt: PreparedStatement = _ 	
       
       // open 主要是创建连接 
       override def open(parameters: Configuration): Unit = { 
           super.open(parameters) conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "123456")
           insertStmt = conn.prepareStatement("INSERT INTO temperatures (sensor, temp) VALUES (?, ?)") 
           updateStmt = conn.prepareStatement("UPDATE temperatures SET temp = ? WHERE sensor = ?") }// 调用连接，执行 sql 
       
       override def invoke(value: SensorReading, context: SinkFunction.Context[_]): Unit = { 
           updateStmt.setDouble(1, value.temperature)
           updateStmt.setString(2, value.id) 
           updateStmt.execute() 
           if (updateStmt.getUpdateCount == 0) { 
               insertStmt.setString(1, value.id) 
               insertStmt.setDouble(2, value.temperature)
               insertStmt.execute() 
           } 
       }
       
       override def close(): Unit = { 
           insertStmt.close() 
           updateStmt.close() 
           conn.close() 
       } 
   }
   
   //main方法
   dataStream.addSink(new MyJdbcSink())
   ```





### Window

Window 可以分成两类： 

​		CountWindow：按照指定的数据条数生成一个 Window，与时间无关

​		TimeWindow：按照时间生成 Window



对于 TimeWindow，可以根据窗口实现原理的不同分成三类：滚动窗口（Tumbling Window）、滑动窗口（Sliding Window）和会话窗口（Session Window）

1. 滚动窗口： 按时间切分数据，前后窗口没有重叠
2. 滑动窗口：按时间切分数据，前后窗口可以有重叠，一般有两个参数，第一个参数控制窗口大小，第二个参数控制窗口开始的频率，如果滑动参数小于窗口大小的话，窗口是可以重叠的
3. 会话窗口：没有重叠和固定的开始时间和结束时间，在一定时间没有接收到数据，前一个窗口关闭，后续元素将被分配到新的 session 窗口中去



**窗口分类**：

在timeWinow这个方法里面传一个参数就是滚动窗口,两个是滑动窗口

```scala
val minTempPerWindow = dataStream.map(
    r => (r.id, r.temperature))
 .keyBy(_._1)
// 时间窗口
//.window(TumblingEventTimeWindows.of(Time.seconds(15),offset  // 滚动time窗口，offset:偏移量
//.window(SlidingEventTimeWindows.of(Time.seconds(15),Time.seconds(5)) // 滑动time窗口
//.window(EventTimeSessionWindows.withGap(Time.seconds(15))        // 会话time窗口
//.timeWindow(Time.seconds(15))   						// 滚动time窗口
//.timeWindow(Time.seconds(15), Time.seconds(5)) 		// 滑动time窗口

// 计数窗口
// .countWindow(5)     //滚动计数窗口
// .countWindow(5，2)  //滑动计数窗口
 .reduce(
     (r1, r2) => (r1._1, r1._2.min(r2._2))
 )

```



PS：全局窗口——Global Window：没有定义窗口的结束时间或者结束条件，一般所有数据一开始都放在这个窗口里



**窗口函数**：

1. 增量函数
   - 每条数据一进来就计算，保持一个简单状态
   - 举例：ReduceFunction, AggregateFunction
2. 全窗口函数
   - 先把窗口所有数据收集起来，等到计算的时候会遍历所有数据
   - 例如：ProcessWindowFunction

**其他API**：

```sql
.trigger() —— 触发器 

定义 window 什么时候关闭，触发计算并输出结果 

 .evitor() —— 移除器

定义移除某些数据的逻辑 

.allowedLateness() —— 允许处理迟到的数据 

.sideOutputLateData() —— 将迟到的数据放入侧输出流 

.getSideOutput() —— 获取侧输出流 
```



### WaterMark

Watermark有水位线、水印的意思，Flink中的WaterMark是单调递增的，它是一条数据，数据中只有Timestamp这一个属性，在数据处理时，Flink会在这些数据中插入WaterMark记录

Flink中WaterMark分两种，一种是按时间周期来插入水印，另一种是来一条数据就插入一次水印，一般基于性能考虑，会选择时间周期来插入水印

**乱序数据处理**：当有乱序数据时，Flink会比较数据中时间戳和WaterMark中时间戳的大小，WaterMark时间戳比较大时候就不更新，反之就更新为数据中的时间戳，因此WaterMaek是单调递增的

**迟到数据处理**：1.  设置乱序区间，当设置乱序区间时候，水印中时间戳值 = 当前数据最大时间戳 - 乱序区间时间

​											举例：当前最大时间为10：05，乱序区间2s，水印中时间戳值为 10：03

​                             2. .allowedLateness()方法，窗口会延迟关闭，延迟时间基于方法中设置的时间     

​                             3..getSideOutput()更迟的数据放到侧输出流里面

注意：只有迟到数据属于的所有窗口都关闭了，这条数据才会放到侧输出流里面

**WaterMark传递**：当出现重分区时，多个分区的WaterMark会传递到下游一个分区里，下游分区会把上游分区的所有WaterMark都存起来，然后向下游传递最小的WaterMark值

**WaterMark更新间隔**：默认200ms



代码：

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment 
// 从调用时刻开始给 env 创建的每一个 stream 追加时间特性，这里设置的是EventTime
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

// 每隔 5 秒产生一个 watermark 
env.getConfig.setAutoWatermarkInterval(5000)

// 设置水印时间和乱序区间
dataStream.assignTimestampsAndWatermarks( new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.milliseconds(1000)) { 
    override def extractTimestamp(element: SensorReading): Long = { 
        element.timestamp * 1000 
    } 
} )
```



**自定义Watermark**:

水印有两个接口

- AssignerWithPeriodicWatermarks
- AssignerWithPunctuatedWatermarks

以上两个接口都继承自 TimestampAssigner



**AssignerWithPeriodicWatermarks**：周期性的生成 watermark，系统会周期性的将 watermark 插入到流中(水位线也 是一种特殊的事件!)。默认周期是 200 毫秒。可以使用ExecutionConfig.setAutoWatermarkInterval()方法进行设置

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment 

env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) 
// 每隔 5 秒产生一个 watermark 
env.getConfig.setAutoWatermarkInterval(5000)
```

每隔 5 秒钟，Flink 会调用 AssignerWithPeriodicWatermarks 的 getCurrentWatermark()方法。如果方法返回一个 时间戳大于之前水位的时间戳，新的 watermark 会被插入到流中。这个检查保证了 水位线是单调递增的。如果方法回的时间戳小于等于之前水位的时间戳，则不会 产生新的 watermark。 



自定义水印代码

```scala
val readings: DataStream[SensorReading] =env.addSource(new SensorSource) 	
				.assignTimestampsAndWatermarks(new MyAssigner())
/*
 需要自定义一个类取实现WaterMark接口
 主要重写两个方法: getCurrentWatermark  获取最大时间戳
 	             extractTimestamp     提取数据中时间戳字段
*/
class PeriodicAssigner extends AssignerWithPeriodicWatermarks[SensorReading] { 
    val bound: Long = 60 * 1000 
    // 延时为 1 分钟 
    var maxTs: Long = Long.MinValue 
    // 观察到的最大时间戳 
    override def getCurrentWatermark: Watermark = { 
        new Watermark(maxTs - bound) 
    }
    
    override def extractTimestamp(r: SensorReading, previousTS: Long) = {
        maxTs = maxTs.max(r.timestamp) 
        r.timestamp 
    } 
}
```





**AssignerWithPunctuatedWatermarks**：间断式地生成 watermark。和周期性生成的方式不同，这种方式不是固定时间的， 而是可以根据需要对每条数据进行筛选和处理

自定义水印

```scala
// 只给 sensor_1 的传感器的数据流插入 watermark
class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks[SensorReading] { 
    val bound: Long = 60 * 1000
    override def checkAndGetNextWatermark(r: SensorReading, extractedTS: Long): Watermark = { 
        if (r.id == "sensor_1") { 
            new Watermark(extractedTS - bound) 
        } else { 
            null 
        } 
    }
    
    override def extractTimestamp(r: SensorReading, previousTS: Long): Long = { 
        r.timestamp 
    } 
}
```



**WaterMark迟到数据收集以及侧输出流**

```scala
val lateTag = new OutputTag[(String,Double,Long)]("late")

val output = dataStream.map(
    r => (r.id, r.temperature))
 .keyBy(_._1)
 .timeWindow(Time.seconds(15))
 .allowedLateness(Time.minutes(1))   // 允许窗口延迟关闭一分钟
 .sideOutputLateData(lateTag)        // 将迟到的数据放入侧输出流 
 .reduce(
     (r1, r2) => (r1._1, r1._2.min(r2._2))
 )

output.print("result") 
output.getSideOutput(lateTag).print("late")  // 打印侧输出流
```



**窗口计算逻辑**

当同时存在**窗口**，**水印**，**乱序区间**，**侧输出流**

水印时间 = 数据中时间 - 乱序区间

窗口在水印时间到达窗口右区间关闭计算一次

之后窗口并未关闭

在后面的允许延迟时间每次到达一个时间在原来的窗口时间中的数据，就计算一次，直到允许延迟的时间也到达才关闭窗口

在之后到达的、仍在此窗口内部的数据，放到侧输出流中

**举例**

数据窗口时间15s，乱序区间3s，允许延迟时间1min

10:00 - 10:15为一个窗口，命名`窗口A`

当时间戳为 10：18的数据到达后，水印时间10：15，`窗口A`第一次计算，但窗口没关闭

当时间戳为 10：21的数据到达后，`窗口A`再次计算，因为10：18之后来的仍在`窗口A`中的数据，每来一条计算一次，直到时间戳为11：18的数据到来，`窗口A`完全关闭

之后再来处于`窗口A`区间中的数据，放到侧输出流中

下面使用数据中时间戳举例，只观察`窗口A`的状态变化

```java
10:05   // 减去乱序后时间为10:02,在区间[10:00,10:15)中，不触发计算，水印时间为10:02
10:12   // 减去乱序后时间为10:09,在区间[10:00,10:15)中，不触发计算，水印时间为10:09
10:15   // 减去乱序后时间为10:12,在区间[10:00,10:15)中，不触发计算，水印时间为10:12
10:13   // 减去乱序后时间为10:10,在区间[10:00,10:15)中，不触发计算，水印时间为10:12
10:18   // 减去乱序后时间为10:15,不在区间[10:00,10:15)中，触发计算，水印时间为10:15
10:21   // 减去乱序后时间为10:18,不在区间[10:00,10:15)中，不触发计算，水印时间为10:18
10:09   // 减去乱序后时间为10:06,在区间[10:00,10:15)中，触发计算，水印时间为10:18
10:11   // 减去乱序后时间为10:08,在区间[10:00,10:15)中，触发计算，水印时间为10:18
11:06   // 减去乱序后时间为11:03,不在区间[10:00,10:15)中，不触发计算，水印时间为11:03
11:15   // 减去乱序后时间为11:12,小于 10:15 + 延迟时间 = 11:15,窗口A没有关闭，水印时间为11:12
11:18   // 减去乱序后时间为11:15,大于等于11:15,窗口A关闭，水印时间为11:15
10:08   // 减去乱序后时间为10:05,在区间[10:00,10:15)中,但窗口A关闭，所以计算但放到侧输出流中
10:09   // 减去乱序后时间为10:06,在区间[10:00,10:15)中,但窗口A关闭，所以计算但放到侧输出流中
```



总结：

1. `数据内时间戳` - `乱序时间`   得到的`结果时间`，用来判断窗口是否关闭
2. `结果时间` 第一次大于等于 `窗口A右边区间时间`，开始第一次窗口计算
3. 后面来的数据`结果时间`如果仍在`窗口A`区间内,并且`水印时间` < `窗口A右边区间时间` +  `允许延迟时间` ,每来一条计算一次
4. 直到`水印时间` > `窗口右边区间时间` +  `允许延迟时间`,`窗口A`真正关闭
5. `窗口A`真正关闭后，后来的数据`结果时间`如果仍在`窗口A`区间内,这些数据放大侧输出流里面计算，每来一条计算一次



**窗口区间确定逻辑**

```java
// startTime计算逻辑可以保证， startTime % windowSize = 0  offset：定义窗口的漂移长度
startTime = timestamp - (timestamp - offset + windowSize) % windowSize

endTime   = startTime + windowSize
```





### State

- 由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态 
- 可以认为状态就是一个本地变量，可以被任务的业务逻辑访问  
- Flink 会进行状态管理，包括状态一致性、故障处理以及高效存储和访问，以 便开发人员可以专注于应用程序的逻辑



存在两种状态：

​	算子状态（Operator State） 

​				算子状态的作用范围限定为算子任务 

​	键控状态（Keyed State） 

​				根据输入数据流中定义的键（key）来维护和访问



**算子状态**： 

算子状态的作用范围限定为算子任务。这意味着由同一并行任务所处理的所有 数据都可以访问到相同的状态，状态对于同一任务而言是共享的。

算子状态不能由 相同或不同算子的另一个任务访问。

Flink 为算子状态提供三种基本数据结构： 

​	列表状态（List state） 

​		将状态表示为一组数据的列表。 

​	联合列表状态（Union list state） 

​		也将状态表示为数据的列表。它与常规列表状态的区别在于，在发生故障时，或者从保 存点（savepoint）启动应用程序时如何恢复。 

​	广播状态（Broadcast state） 

​		如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态



**键控状态**：

必须要是在实体类里面定义，在匿名内部类里面定义会被当作算子状态

获取状态需要运行时上下文，而运行时上下文需要在类实例化之后才会生效，所以运行时上下文应该放在open方法内，或者是前面加个`lazy`修饰

键控状态是根据输入数据流中定义的键（key）来维护和访问的。

Flink 为每个键值维护 一个状态实例，并将具有相同键的所有数据，都分区到同一个算子任务中，这个任务会维护 

和处理这个 key 对应的状态。

当任务处理一条数据时，它会自动将状态的访问范围限定为当 前数据的 key。因此，具有相同 key 的所有数据都会访问相同的状态。



值状态（Value state） 

• 将状态表示为单个的值 



列表状态（List state） 

• 将状态表示为一组数据的列表 



映射状态（Map state） 

• 将状态表示为一组 Key-Value 对 



聚合状态（Reducing state & Aggregating State） 

• 将状态表示为一个用于聚合操作的列表



Flink 的 Keyed State 支持以下数据类型： 

```scala
ValueState[T]保存单个的值，值的类型为 T。 
	get 操作: ValueState.value()
	set 操作: ValueState.update(value: T) 

ListState[T]保存一个列表，列表里的元素的数据类型为 T。基本操作如下：
	ListState.add(value: T) 
	ListState.get()返回 Iterable[T] 
	ListState.update(values: java.util.List[T]) 

MapState[K, V]保存 Key-Value 对。 
	MapState.get(key: K) 
	MapState.put(key: K, value: V) 
	MapState.contains(key: K) 
	MapState.remove(key: K) 
	ReducingState[T] 
	AggregatingState[I, O]

State.clear()是清空操作
```



状态编程：

当传感器温差大于10，输出温度

1. 键控状态new一个状态描述器实现

   通过 RuntimeContext 注册 StateDescriptor。StateDescriptor 以状态 state 的名字和存储的数据类型为参数。

```scala
val sensorData: DataStream[SensorReading] = ... 
val keyedData: KeyedStream[SensorReading, String] = sensorData.keyBy(_.id) 
val alerts: DataStream[(String, Double, Double)] = keyedData.flatMap(
    new TemperatureAlertFunction(1.7)
) 

class TemperatureAlertFunction(val threshold: Double) extends RichFlatMapFunction[SensorReading, (String, Double, Double)] {
    private var lastTempState: ValueState[Double] = _ 
    
    override def open(parameters: Configuration): Unit = { 
        val lastTempDescriptor = new ValueStateDescriptor[Double]("lastTemp", classOf[Double])
        
        lastTempState = getRuntimeContext.getState[Double](lastTempDescriptor) 
    }
    
    override def flatMap(reading: SensorReading, out: Collector[(String, Double, Double)]): Unit = {
        val lastTemp = lastTempState.value() 
        val tempDiff = (reading.temperature - lastTemp).abs 
        if (tempDiff > threshold) { 
            out.collect((reading.id, reading.temperature, tempDiff)) 
        }
        this.lastTempState.update(reading.temperature) 
    } 
}
```



2. 使用`FlatMap with keyed ValueState `的快捷方式 flatMapWithState 实现以上需求

```scala
val alerts: DataStream[(String, Double, Double)] = keyedSensorData.flatMapWithState[(String, Double, Double), Double] { 
    // 方法签名
    // fun：(T, Option[S]) => (TraversableOnce[R], Option[S])
    // T:传入参数	S：状态的类型		R：返回类型   TraversableOnce：所有集合的上层类型
    case (in: SensorReading, None) => 
    	(List.empty, Some(in.temperature)) 
    
    case (r: SensorReading, lastTemp: Some[Double]) => 
    	val tempDiff = (r.temperature - lastTemp.get).abs 	
    	if (tempDiff > 10.0) { 
            (List((r.id, r.temperature, tempDiff)), Some(r.temperature)) 
        } else { 
            (List.empty, Some(r.temperature)) 
        } 
}
```



### ProcessFunction

普通的算子无法访问事件的时间戳信息或是水位线信息，而在有些业务场景里面这种功能尤为重要，Flink提供了一系列Low-Level转换算子，可以访问时间戳、watermark以及注册定时事件。以及输出超时事件等等

Flink提供8种ProcessFunction

- ProcessFunction
- KeyedProcessFunction
- CoProcessFunction
- ProcessJoinFunction
- BroadcastProcessFunction
- KeyedBroadcastProcessFunction
- ProcessWindowFunction
- ProcessAllWindowFunction



**KeyedProcessFunction**

KeyedProcessFunction 用来操作 KeyedStream。KeyedProcessFunction 会处理流 的每一个元素，输出为 0 个、1 个或者多个元素。所有的 Process Function 都继承自 RichFunction 接口，所以都有 open()、close()和 getRuntimeContext()等方法。

而KeyedProcessFunction[KEY, IN, OUT]还额外提供了两个方法:

- processElement(v: IN, ctx: Context, out: Collector[OUT]), 流中的每一个元素 都会调用这个方法，调用结果将会放在 Collector 数据类型中输出。**Context** 可以访问元素的时间戳，元素的key，以及 **TimerService** 时间服务。**Context**还可以将结果输出到别的流(side outputs)。
- onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[OUT])是一个回调函数。当之前注册的定时器触发时调用。参数 timestamp 为定时器所定 的触发的时间戳。Collector 为输出结果的集合。OnTimerContext 和 processElement 的 Context 参数一样，提供了上下文的一些信息，例如定时器触发的时间信息(事件时间或者处理时间)。 



**TimerService 和 定时器（Timers）**

Context 和 OnTimerContext 所持有的 TimerService 对象拥有以下方法:

- currentProcessingTime(): Long 返回当前处理时间
- currentWatermark(): Long 返回当前 watermark 的时间戳 
- registerProcessingTimeTimer(timestamp: Long): Unit 会注册当前 key 的processing time 的定时器。当 processing time 到达定时时间时，触发 timer。
- registerEventTimeTimer(timestamp: Long): Unit 会注册当前 key 的 event time定时器。当水位线大于等于定时器注册的时间时，触发定时器执行回调函数。
- deleteProcessingTimeTimer(timestamp: Long): Unit 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行
- deleteEventTimeTimer(timestamp: Long): Unit 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行

当定时器 timer 触发时，会执行回调函数onTimer()。注意定时器 timer 只能在keyed streams 上面使用。



代码示例：监控温度传感器的温度值，如果温度值在一秒钟之内(processing time)连续上升，则报警

```scala
val warnings = readings .keyBy(_.id) .process(new TempIncreaseAlertFunction)

class TempIncreaseAlertFunction extends KeyedProcessFunction[String, SensorReading, String] { 
    // 保存上一个传感器温度值 
    lazy val lastTemp: ValueState[Double] = getRuntimeContext.getState( new ValueStateDescriptor[Double]("lastTemp", Types.of[Double]) )
    // 保存注册的定时器的时间戳 
    lazy val currentTimer: ValueState[Long] = getRuntimeContext.getState( new ValueStateDescriptor[Long]("timer", Types.of[Long]) )
    override def processElement(r: SensorReading, ctx: KeyedProcessFunction[String, SensorReading, String]#Context, out: Collector[String]): Unit = {
        // 取出上一次的温度 
        val prevTemp = lastTemp.value() 
        // 将当前温度更新到上一次的温度这个变量中 
        lastTemp.update(r.temperature) 
        val curTimerTimestamp = currentTimer.value() 
        if (prevTemp == 0.0 || r.temperature < prevTemp) { 
            // 温度下降或者是第一个温度值，删除定时器 
            ctx.timerService().deleteProcessingTimeTimer(curTimerTimestamp) 
            // 清空状态变量 
            currentTimer.clear() 
        } else if (r.temperature > prevTemp && curTimerTimestamp == 0) {
            // 温度上升且我们并没有设置定时器 
            val timerTs = ctx.timerService().currentProcessingTime() + 1000 
            ctx.timerService().registerProcessingTimeTimer(timerTs) 
            currentTimer.update(timerTs) 
        }
    }     
        override def onTimer(ts: Long, ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext, out: Collector[String]): Unit = {
            out.collect("传感器 id 为: " + ctx.getCurrentKey + "的传感器温度值已经连续 1s 上升了。") currentTimer.clear() 
        } 
}
```



**侧输出流**

process function 的 side outputs 功能可以产生多条流，并且这些流的数据类型可以不一样。

一个 side output 可以定义为 OutputTag[X]对象，X 是输出流的数据类型。

process function 可以通过 Context 对象发射一个事件到一个或者多个 side outputs。



代码示例：将温度值低于32F 的温度输出到 side output。

```scala
val monitoredReadings: DataStream[SensorReading] = readings.process(new FreezingMonitor) 
monitoredReadings.getSideOutput(new OutputTag[String]("freezing-alarms")).print() 
readings.print()

class FreezingMonitor extends ProcessFunction[SensorReading, SensorReading] { 
    // 定义一个侧输出标签 
    lazy val freezingAlarmOutput: OutputTag[String] = new OutputTag[String]("freezing-alarms") 
    override def processElement(r: SensorReading, ctx: ProcessFunction[SensorReading, SensorReading]#Context, out: Collector[SensorReading]): Unit = { 
        // 温度在 32F 以下时，输出警告信息 
        if (r.temperature < 32.0) { 
            ctx.output(freezingAlarmOutput, s"Freezing Alarm for ${r.id}")
            }
        // 所有数据直接常规输出到主流 
        out.collect(r) 
    } 
}
```



**CoProcessFunction**

对于两条输入流，DataStream API 提供了 CoProcessFunction 这样的 low-level操作。CoProcessFunction 提供了操作每一个输入流的方法: processElement1()和processElement2()。

类似于 ProcessFunction，这两种方法都通过 Context 对象来调用。这个 Context对象可以访问事件数据，定时器时间戳，TimerService，以及 side outputs。CoProcessFunction 也提供了 onTimer()回调函数。



### Checkpoint

 Flink的状态保存基于Chandy-Lamport算法

JobManager会发送barrier到下游，首先会发送到Source端，Source端收到barrier后，会阻塞自身任务，并把自身的State存到状态后端检查点中，之后报告HobManager

之后barrier向下游广播，下游接收到barrier后，任务阻塞并保存状态，如果下游有分流，需要下游的多个节点都收到同一个barrier id之后（barrier对齐），才开始进行checkpoint，存储完成后任务恢复，barrier向下游流动

barrier一直向下流动，流动到哪个任务节点，任务节点等这个barrier id到齐后开始checkpoint，直到sink端也保存完成后，本次checkpoint宣告完成

在barrier对齐过程中，如果有一个barrier迟迟不来，其他的节点里面，barrier之后的数据会暂时缓存起来不处理



**参数**：

```scala
env.enableCheckpointing()       // 开启checkpoint，默认500ms进行一次
env.enableCheckpointing(200L)   // 开启checkpoint，设置200ms进行一次，底层默认CheckpointingMode为EXACTLY_ONCE(还有AT_LEAST_ONCE)
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)    // AT_LEAST_ONCE精确性低一点、任务快一点
env.getCheckpointConfig.setCheckpointTimeout(60000L)   // checkpoint超过一分钟就丢弃
env.getCheckpointConfig.setMaxConcurrentCheckpoints(2) // 设置最多只能有多少Checkpoint同时进行，默认1
env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500L)   // 设置两次Checkpoint之间的时间间隔，单位ms
env.getCheckpointConfig.setPerferCheckpointForRecovery(true)  // 相比save point，是否更倾向于用checkpoint做故障恢复
env.getCheckpointConfig.setToerableCheckpointFailureNumber(0) // 经过多少次checkpoint失败后，flink作业失败。默认0次
```



**重启策略**

```scala
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 6000L))  // 参数：重启次数、重启时间间隔
env.setRestartStrategy(RestartStrategies.fixedRateRestart(3, Time.of(5, TimeUnit.MINUTES), Time.of(10, TimeUnit.SECONDS)))   // 参数： 每个测量时间间隔最大失败次数、失败率测量的时间间隔、两次连续重启尝试的时间间隔
```



**Save Point**

手动定义的存盘点，底层一样，save point可能多一个元数据、上下文的存储

只能手动触发，存盘后，之后启动就可以从这个save point里面启动

场景：当发现bug，手动保存save point，之后停止任务修复bug，之后可以从save point重启（要求算子链（id）没有更改，状态不能变）

​         集群版本迁移

​         集群资源紧张时候，可以保存部分不重要任务，跑完重要任务后从save point恢复不重要任务



给算子分配id：

```scala
dataStream.map(...).uid("1")
```



**状态一致性**

端到端的一致性：

- 幂等写入
- 事务写入

事务写入的两种方式：

- 预写日志：结果当作状态保存，当checkpoint完成后，结果再写入系统；DataStream API 提供了一个模板类：GenericWriteAheadSink，来实 

  现这种事务性 sink

- 两阶段提交：对于每个 checkpoint，sink 任务会启动一个事务，并将接下来所有接收的数据添加到事务里 ，然后将这些数据写入外部 sink 系统，但不提交它们 —— 这时只是“预提交” ，当它收到 checkpoint 完成的通知时，它才正式提交事务，实现结果的真正写入



**flink到kafka的两阶段提交**

两阶段提交需要Sink端支持事务

- 第一条数据来了之后，开启一个 kafka 的事务（transaction），正常写入 kafka 分 区日志但标记为未提交，这就是“预提交” 
- jobmanager 触发 checkpoint 操作，barrier 从 source 开始向下传递，遇到 barrier 的算子将状态存入状态后端，并通知 jobmanager 
- sink 连接器收到 barrier，保存当前状态，存入 checkpoint，通知 jobmanager，并开启下一阶段的事务，用于提交下个检查点的数据 
- jobmanager 收到所有任务的通知，发出确认信息，表示 checkpoint 完成 
- sink 任务收到 jobmanager 的确认信息，正式提交这段时间的数据 
- 外部kafka关闭事务，提交的数据可以正常消费了。

注意：checkpoint超时时间要比kafka事务超时时间小



### Flink CEP

**什么是CEP**

复杂事件处理（Complex Event Processing，CEP）

- CEP 允许在无休止的事件流中检测事件模式，让我们有机会掌握数据中重要的部分 
- 一个或多个由简单事件构成的事件流通过一定的规则匹配，然后输出 用户想得到的数据 —— 满足规则的复杂事件

Flink CEP 提供了 Pattern API，用于对输入流数据进行复杂事件规则定义，用来提取符合规则的事件序列



**Pattern API**

- 个体模式（Individual Patterns） 

  ​	– 组成复杂规则的每一个单独的模式定义，就是“个体模式” 

- 组合模式（Combining Patterns，也叫模式序列） 

  ​	–  很多个体模式组合起来，就形成了整个的模式序列 

  ​    –  模式序列必须以一个“初始模式”开始：

  ```scala
  val start = Pattern.begin("Start")
  ```

- 模式组（Groups of patterns） 

  ​	– 将一个模式序列作为条件嵌套在个体模式里，成为一组模式



**个体模式**

1. 简单条件（Simple Condition） 

   ​	– 通过 .where() 方法对事件中的字段进行判断筛选，决定是否接受该事件

   ```scala
   start.where(event => event.getName.startWith("foo"))
   ```

2. 组合条件（Combining Condition）

    – 将简单条件进行合并；.or() 方法表示或逻辑相连，where 的直接组合就是 AND

   ```scala
   pattern.where(event=> ...).or(vent=> ...)
   ```

3. 迭代条件（Iterative Condition） 

   – 能够对模式之前所有接收的事件进行处理 

   – 调用 .where( (value, ctx) => {...} )，可以调用 ctx.getEventsForPattern(“name”)

量词：追加量词，指定循环次数

```scala
// 匹配出现4次
start.times(4)

// 匹配出现2次、4次，并尽可能多重复匹配
start.times(2, 4).greedy

// 匹配出现0次或4次
start.times(4).optional

// 匹配出现2,3或4次
start.times(2, 4)

// 匹配出现1次或多次
start.oneOrMore

// 匹配出现0次、2次或多次，并尽可能多重复匹配
start.timesOrMore(2).optional.greedy
```



**模式序列**

- 严格近邻（Strict Contiguity） 

  ​		– 所有事件按照严格的顺序出现，中间没有任何不匹配的事件，由 .next() 指定 

  ​		– 例如对于模式”a next b”，事件序列 [a, c, b1, b2] 没有匹配 

- 宽松近邻（ Relaxed Contiguity ） 

  ​		– 允许中间出现不匹配的事件，由 .followedBy() 指定 

  ​		– 例如对于模式”a followedBy b”，事件序列 [a, c, b1, b2] 匹配为 {a, b1} 

- 非确定性宽松近邻（ Non-Deterministic Relaxed Contiguity ） 

  ​		– 进一步放宽条件，之前已经匹配过的事件也可以再次使用，由 .followedByAny() 指定 

  ​		– 例如对于模式”a followedByAny b”，事件序列 [a, c, b1, b2] 匹配为 {a, b1}，{a, b2}

- 除以上模式序列外，还可以定义“不希望出现某种近邻关系”： 

  ​		– .notNext() —— 不想让某个事件严格紧邻前一个事件发生 

  ​		– .notFollowedBy() —— 不想让某个事件在两个事件之间发生

- 需要注意：

  ​		– 所有模式序列必须以 .begin() 开始 

  ​		– 模式序列不能以 .notFollowedBy() 结束 

  ​		– "not" 类型的模式不能被 optional 所修饰 

  ​		– 此外，还可以为模式指定时间约束，用来要求在多长时间内匹配有效

  ```scala
  next.within(Time.seconds(10))
  ```



**匹配事件提取**

- 创建 PatternStream 之后，就可以应用 select 或者 flatselect 方法，从检测到的事件序列中提取事件了 
- select() 方法需要输入一个 select function 作为参数，每个成功匹配的事件序列都会调用它 
- select() 以一个 Map[String，Iterable [IN]] 来接收匹配到的事件序列，其中 key 就是每个模式的名称，而 value 就是所有接收到的事件的 Iterable 类型

```scala
def select(map: util.Map[String, util.List[IN]]):OUT = {
    val startEvent = map.get("start").get.next
    val endEvent = map.get("end").get.next
    OUT(startEvent, endEvent)
}
```



**超时事件提取**

- 当一个模式通过 within 关键字定义了检测窗口时间时，部分事件序列可能因为超过窗口长度而被丢弃；为了能够处理这些超时的部分匹配，select 和 flatSelect API 调用允许指定超时处理程序 
- 超时处理程序会接收到目前为止由模式匹配到的所有事件，由一个 OutputTag 定义接收到的超时事件序列

```scala
val resultStream = CEP.pattern(inStream, pattern)
val orderTimeoutOutputTag = new OutputTag[String]("orderTimeout")

val result = resultStream.select(orderTimeoutOutputTag){
    (pattern: Map(String, Iterable[Event], timestamp: Long) => TimeoutEvent()
}{
    pattern: Map(String, Iterable[Event] => ComplexEvent()
}
val timeoutResult = result.getSideOutPut(orderTimeoutOutputTag)
```



### Flink Table

**基本程序**

```scala
// 创建表的执行环境
val tableEnv = StreamTableEnvironment.create(env)		

// 创建一张表，用于读取数据 
tableEnv.connect(...).createTemporaryTable("inputTable") 
// 注册一张表，用于把计算结果输出 
tableEnv.connect(...).createTemporaryTable("outputTable")

// 通过 Table API 查询算子，得到一张结果表 
val result = tableEnv.from("inputTable").select(...) 
// 通过 SQL 查询语句，得到一张结果表 
val sqlResult = tableEnv.sqlQuery("SELECT ... FROM inputTable ...")

// 将结果表写入输出表中
result.insertInto("outputTable")
```

配置老版本的流式查询

```scala
val settings = EnvironmentSettings.newInstance() 
	.useOldPlanner() // 使用老版本 planner 
	.inStreamingMode() // 流处理模式 
	.build() 
val tableEnv = StreamTableEnvironment.create(env, settings)
```

基于老版本的批处理环境（Flink-Batch-Query）

```scala
val batchEnv = ExecutionEnvironment.getExecutionEnvironment 
val batchTableEnv = BatchTableEnvironment.create(batchEnv)
```

基于 blink 版本的流处理环境（Blink-Streaming-Query）

```scala
val bsSettings = EnvironmentSettings.newInstance() 
	.useBlinkPlanner() 
	.inStreamingMode()
	.build() 
val bsTableEnv = StreamTableEnvironment.create(env, bsSettings)
```

基于 blink 版本的批处理环境（Blink-Batch-Query）

```scala
val bbSettings = EnvironmentSettings
	.newInstance() 
	.useBlinkPlanner() 
	.inBatchMode().build() 
val bbTableEnv = TableEnvironment.create(bbSettings)
```



**连接到文件系统**

```scala
tableEnv .connect( new FileSystem().path("sensor.txt")) // 定义表数据来源，外部连接 
	.withFormat(new OldCsv()) // 定义从外部系统读取数据之后的格式化方法 
	.withSchema( new Schema()
                .field("id", DataTypes.STRING()) 
                .field("timestamp", DataTypes.BIGINT()) 
                .field("temperature", DataTypes.DOUBLE()) ) // 定义表结构 
	.createTemporaryTable("inputTable") // 创建临时表
```

**连接到** **Kafka**

```scala
tableEnv
.connect( new Kafka() 
         	.version("0.11") // 定义 kafka 的版本 
         	.topic("sensor") // 定义主题 
         	.property("zookeeper.connect", "localhost:2181")
         	.property("bootstrap.servers", "localhost:9092") 
        )
.withFormat(new Csv()) 
.withSchema(new Schema() 
            .field("id", DataTypes.STRING()) 
            .field("timestamp", DataTypes.BIGINT()) 
            .field("temperature", DataTypes.DOUBLE())
            )
.createTemporaryTable("kafkaInputTable")
```

**两种查询方式**

```scala
// 第1种
val sensorTable: Table = tableEnv.from("inputTable")
val resultTable: Table = senorTable 
	.select("id, temperature") 
	.filter("id ='sensor_1'")

// 第2种
val resultSqlTable: Table = tableEnv.sqlQuery("select id, temperature from inputTable where id ='sensor_1'")
// 第2种
val resultSqlTable: Table = tableEnv.sqlQuery( 
    """
    	|select id, temperature 
    	|from inputTable 
    	|where id = 'sensor_1' 
    """.stripMargin)
```



**代码表达**，可以给字段重命名，改顺序

```scala
val inputStream: DataStream[String] = env.readTextFile("sensor.txt") 
val dataStream: DataStream[SensorReading] = inputStream 
	.map(data => { 
        val dataArray = data.split(",") 
        SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble) }) 
val sensorTable: Table = tableEnv.fromDataStream(dataStream) 
val sensorTable2 = tableEnv.fromDataStream(dataStream, 'id, 'timestamp as 'ts)
```

基于名称对应

```scala
val sensorTable = tableEnv.fromDataStream(dataStream, 'timestamp as 'ts, 'id as 'myId, 'temperature)
```

基于位置对应

```scala
val sensorTable = tableEnv.fromDataStream(dataStream, 'myId, 'ts)
```



**创建临时视图**

```scala
// 第1种
tableEnv.createTemporaryView("sensorView", dataStream) 
// 第2种
tableEnv.createTemporaryView("sensorView", dataStream, 'id, 'temperature, 'timestamp as 'ts)
// 第3种
tableEnv.createTemporaryView("sensorView", sensorTable)
```



**输出表**

```scala
// Sink到文件系统
// 注册输出表 
tableEnv.connect( new FileSystem().path("…\\resources\\out.txt"))  // 定义到文件系统的连接 
.withFormat(new Csv())  // 定义格式化方法，Csv 格式 
.withSchema(new Schema() .field("id", DataTypes.STRING()) .field("temp", DataTypes.DOUBLE()) ) // 定义表结构 
.createTemporaryTable("outputTable") // 创建临时表 resultSqlTable.insertInto("outputTable")
```



**三种更新模式**

1. **追加模式（Append Mode）**

   ​	在追加模式下，表（动态表）和外部连接器只交换插入（Insert）消息

2. **撤回模式（Retract Mode）**

   ​	在撤回模式下，表和外部连接器交换的是：添加（Add）和撤回（Retract）消息。

   - 插入（Insert）会被编码为添加消息； 
   - 删除（Delete）则编码为撤回消息； 
   - 更新（Update）则会编码为，已更新行（上一行）的撤回消息，和更新行（新行）的添加消息。
   - 在此模式下，不能定义 key，这一点跟 upsert 模式完全不同

3. **Upsert（更新插入）模式**

   ​	在 Upsert 模式下，动态表和外部连接器交换 Upsert 和 Delete 消息。

   ​	这个模式需要一个唯一的 key，通过这个 key 可以传递更新消息。为了正确应用消息，外部连接器需要知道这个唯一 key 的属性。

   - 插入（Insert）和更新（Update）都被编码为 Upsert 消息
   - 删除（Delete）编码为 Delete 信息。 

   这种模式和 Retract 模式的主要区别在于，Update 操作是用单个消息编码的，所以效率会更高



**输出到** **Kafka**

```scala
// 输出到 kafka 
tableEnv.connect( new Kafka() 
                 .version("0.11") 
                 .topic("sinkTest") 
                 .property("zookeeper.connect", "localhost:2181") 
                 .property("bootstrap.servers", "localhost:9092") 
                ) 
	.withFormat( new Csv() ) 
	.withSchema( new Schema() 
            .field("id", DataTypes.STRING()) 
            .field("temp", DataTypes.DOUBLE()) )
	.createTemporaryTable("kafkaOutputTable") 

resultTable.insertInto("kafkaOutputTable")
```



**输出到** **ElasticSearch**

ElasticSearch 的 connector 可以在 upsert（update+insert，更新插入）模式下操作，这样就可以使用 Query 定义的键（key）与外部系统交换 UPSERT/DELETE 消息。

另外，对于“仅追加”（append-only）的查询，connector 还可以在 append 模式下操作，这样就可以与外部系统只交换 insert 消息。 

es 目前支持的数据格式，只有 Json，而 flink 本身并没有对应的支持，所以还需要引入依赖：

```xml
<dependency> 
    <groupId>org.apache.flink</groupId> 
    <artifactId>flink-json</artifactId> 
    <version>1.10.1</version> 
</dependency>
```

代码

```scala
// 输出到 es 
tableEnv.connect( 
    new Elasticsearch() 
    .version("6") 
    .host("localhost", 9200, "http") 
    .index("sensor") 
    .documentType("temp") 
) 
.inUpsertMode() // 指定是 Upsert 模式 
.withFormat(new Json()) 
.withSchema( new Schema() 
            .field("id", DataTypes.STRING()) 
            .field("count", DataTypes.BIGINT()) )
.createTemporaryTable("esOutputTable") 

aggResultTable.insertInto("esOutputTable")
```



**输出到** **MySql**

依赖：

```scala
<dependency> 
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-jdbc_2.12</artifactId> 
	<version>1.10.1</version> 
</dependency>
```

jdbc 连接的代码实现比较特殊，因为没有对应的 java/scala 类实现 ConnectorDescriptor，所以不能直接 tableEnv.connect()。不过 Flink SQL留下了执行 DDL的接口：`tableEnv.sqlUpdate()`。

对于 jdbc 的创建表操作，天生就适合直接写 DDL 来实现，所以我们的代码可以这样写

```scala
// 输出到 Mysql 
val sinkDDL: String = 
	"""
	|create table jdbcOutputTable ( 
	| id varchar(20) not null, 
	| cnt bigint not null 
	|) with ( 
	| 'connector.type' = 'jdbc', 
	| 'connector.url' = 'jdbc:mysql://localhost:3306/test', 
	| 'connector.table' = 'sensor_count', 
	| 'connector.driver' = 'com.mysql.jdbc.Driver', 
	| 'connector.username' = 'root', 
	| 'connector.password' = '123456' 
	|)
    """.stripMargin 

tableEnv.sqlUpdate(sinkDDL) 
aggResultSqlTable.insertInto("jdbcOutputTable")
```



**将表转换成** **DataStream**

最方便的转换类型就是 Row。当然，因为结果的所有字段类型都是明确的，也经常会用元组类型来表示

Table API 中表到 DataStream 有两种模式： 

- 追加模式（Append Mode） 

  ​	用于表只会被插入（Insert）操作更改的场景。 

- 撤回模式（Retract Mode）

  ​	用于任何场景。有些类似于更新模式中 Retract 模式，它只有 Insert 和 Delete 两类操作。

  ​	得到的数据会增加一个 Boolean 类型的标识位（返回的第一个字段），用它来表示到底是新增的数据（Insert），还是被删除的数据（老数据， Delete）



代码

```scala
val resultStream: DataStream[Row] = tableEnv.toAppendStream[Row](resultTable) 
val aggResultStream: DataStream[(Boolean, (String, Long))] = tableEnv.toRetractStream[(String, Long)](aggResultTable) 
resultStream.print("result")
aggResultStream.print("aggResult")
```

所以，没有经过 groupby 之类聚合操作，可以直接用 toAppendStream 来转换；而如果经过了聚合，有更新操作，一般就必须用 toRetractDstream





**Query** **的解释和执行** 

Table API 提供了一种机制来解释（Explain）计算表的逻辑和优化查询计划。这是通过TableEnvironment.explain（table）方法或 TableEnvironment.explain（）方法完成的。

explain 方法会返回一个字符串，描述三个计划： 

1. 未优化的逻辑查询计划 
2. 优化后的逻辑查询计划 
3. 实际执行计划 

```scala
val explaination: String = tableEnv.explain(resultTable) 
println(explaination)
```

Query 的解释和执行过程，老 planner 和 blink planner 大体是一致的，又有所不同。整 体来讲，Query 都会表示成一个逻辑查询计划，然后分两步解释： 

1. 优化查询计划  
2. 解释成 DataStream 或者 DataSet 程序 

而 Blink 版本是批流统一的，所以所有的 Query，只会被解释成 DataStream 程序；另外在批处理环境 TableEnvironment 下，Blink 版本要到 tableEnv.execute()执行调用才开始解释。



**动态表（Dynamic Tables）**

因为流处理面对的数据，是连续不断的，这和我们熟悉的关系型数据库中保存的“表”完全不同。所以，如果我们把流数据转换成 Table，然后执行类似于 table 的 select 操作，结果就不是一成不变的，而是随着新数据的到来，会不停更新

以随着新数据的到来，不停地在之前的基础上更新结果。这样得到的表，在 Flink Table API 概念里，就叫做“**动态表**”（Dynamic Tables）



**将动态表转换成流**

- **仅追加（Append-only）流**

  ​	仅通过插入（Insert）更改，来修改的动态表，可以直接转换为“仅追加”流。这个流中发出的数据，就是动态表中新增的每一行。

- **撤回（Retract）流**

  ​	Retract 流是包含两类消息的流，添加（Add）消息和撤回（Retract）消息。 

  ​	动态表通过将 INSERT 编码为 add 消息、DELETE 编码为 retract 消息、UPDATE 编码为被更改行（前一行）的retract 消息和更新后行（新行）的 add 消息，转换为 retract 流

- **Upsert（更新插入）流**

  ​	Upsert 流包含两种类型的消息：Upsert 消息和 delete 消息。转换为 upsert 流的动态表，需要有唯一的键（key）。

  ​	通过将 INSERT 和 UPDATE 更改编码为 upsert 消息，将 DELETE 更改编码为 DELETE 消息，就可以将具有唯一键（Unique Key）的动态表转换为流。

  ​	在代码里将动态表转换为 DataStream，仅支持 Append 和 Retract 流

  ​	而向外部系统输出动态表的 TableSink 接口，需要Sink端支持实现Upsert模式（比如ES、数据库）



**时间特性**（Time Attributes）

定义相关的时间语义和时间数据来源的信息，Table 可以提供一个逻辑上的时间字段，用于在表处理程序中，指示时间和访问相应的时间戳。

时间属性，可以是每个表 schema 的一部分。一旦定义了时间属性，它就可以作为一个字段引用，并且可以在基于时间的操作中使用。



**Process Time**

1. **DataStream** **转化成** **Table** **时指定**

```scala
val inputStream: DataStream[String] = env.readTextFile("\\sensor.txt") 
val dataStream: DataStream[SensorReading] = inputStream .map(data => { 
    val dataArray = data.split(",") 
    SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble) 
}) 
// 将 DataStream 转换为 Table，并指定时间字段 
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp, 'pt.proctime)
```

2. **定义** **Table Schema** **时指定**

```scala
tableEnv.connect( new FileSystem().path("..\\sensor.txt")) 
	.withFormat(new Csv()) 
	.withSchema(new Schema() 
            .field("id", DataTypes.STRING()) 
            .field("timestamp", DataTypes.BIGINT()) 
            .field("temperature", DataTypes.DOUBLE()) 
            .field("pt", DataTypes.TIMESTAMP(3)) .proctime() // 指定 pt 字段为处理时间 
           )  // 定义表结构  
	.createTemporaryTable("inputTable") // 创建临时表
```

3. **创建表的** **DDL** **中指定**:运行这段 DDL，必须使用 Blink Planner

```scala
val sinkDDL: String =
	"""
	|create table dataTable ( 
	| id varchar(20) not null, 
	| ts bigint, 
	| temperature double, 
	| pt AS PROCTIME() 
	|) with ( 
	| 'connector.type' = 'filesystem', 
	| 'connector.path' = 'file:///D:\\..\\sensor.txt', 
	| 'format.type' = 'csv' 
	|) 
	""".stripMargin 

tableEnv.sqlUpdate(sinkDDL) // 执行 DDL
```





**事件时间（Event Time）** 

为了处理无序事件，并区分流中的准时和迟到事件；Flink 需要从事件数据中，提取时间戳，并用来推进事件时间的进展（watermark）

1. **DataStream** **转化成** **Table** **时指定**

   ​	使用.rowtime 可以定义事件时间属性。注意，必须在转换的数据流中分配时间戳和 watermark

```scala
val inputStream: DataStream[String] = env.readTextFile("\\sensor.txt") 
val dataStream: DataStream[SensorReading] = inputStream .map(data => { 
    val dataArray = data.split(",") 
    SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble) 
}) 
	.assignAscendingTimestamps(_.timestamp * 1000L) 
// 将 DataStream 转换为 Table，并指定时间字段 
val sensorTable = tableEnv.fromDataStream(dataStream, 'id, 'timestamp.rowtime, 'temperature) 
// 或者，直接追加字段 
val sensorTable2 = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp, 'rt.rowtime)
```

2. **定义** **Table Schema** **时指定**

```scala
tableEnv.connect(
    new FileSystem().path("sensor.txt"))
	.withFormat(new Csv()) 
.withSchema(
    new Schema() 
    .field("id", DataTypes.STRING()) 
    .field("timestamp", DataTypes.BIGINT()) 
    .rowtime( new Rowtime()
             .timestampsFromField("timestamp") // 从字段中提取时间戳 
    	     .watermarksPeriodicBounded(1000)  // watermark 延迟 1 秒 
            ) 
	.field("temperature", DataTypes.DOUBLE()) ) // 定义表结构 
.createTemporaryTable("inputTable") // 创建临时表
```

3. **创建表的** **DDL** **中指定**

```scala
val sinkDDL: String = 
	"""
	|create table dataTable ( 
	| id varchar(20) not null, 
	| ts bigint, 
	| temperature double, 
	| rt AS TO_TIMESTAMP( FROM_UNIXTIME(ts) ), 
	| watermark for rt as rt - interval '1' second 
	|) with (
	| 'connector.type' = 'filesystem', 
	| 'connector.path' = 'file:///D:\\..\\sensor.txt', 
	| 'format.type' = 'csv' 
	|) 
	""".stripMargin 
tableEnv.sqlUpdate(sinkDDL) // 执行 DDL
```



**窗口（Windows）**



1. **分组窗口（Group Windows）**

分组窗口（Group Windows）会根据时间或行计数间隔，将行聚合到有限的组（Group）中，并对每个组的数据执行一次聚合函数。

具体分为滚动（Tumbling）、滑 动（Sliding）和会话（Session）窗口。

```scala
val table = input 
.window([w: GroupWindow] as 'w) // 定义窗口，别名 w 
.groupBy('w, 'a) 				// 以属性 a 和窗口 w 作为分组的 key 
.select('a, 'b.sum) 			// 聚合字段 b 的值，求和

val table = input 
	.window([w: GroupWindow] as 'w) 
	.groupBy('w, 'a) 
	.select('a, 'w.start, 'w.end, 'w.rowtime, 'b.count)
```



**滚动窗口**

滚动窗口（Tumbling windows）要用 Tumble 类来定义，另外还有三个方法： 

- over：定义窗口长度 
- on：用来分组（按时间间隔）或者排序（按行数）的时间字段 
- as：别名，必须出现在后面的 groupBy 中 

```scala
// Tumbling Event-time Window（事件时间字段 rowtime） 
.window(Tumble over 10.minutes on 'rowtime as 'w) 

// Tumbling Processing-time Window（处理时间字段 proctime） 
.window(Tumble over 10.minutes on 'proctime as 'w) 

// Tumbling Row-count Window (类似于计数窗口，按处理时间排序，10 行一组) 
.window(Tumble over 10.rows on 'proctime as 'w)
```



**滑动窗口** 

滑动窗口（Sliding windows）要用 Slide 类来定义，另外还有四个方法： 

- over：定义窗口长度 
- every：定义滑动步长 
- on：用来分组（按时间间隔）或者排序（按行数）的时间字段 
- as：别名，必须出现在后面的 groupBy 中

```scala
// Sliding Event-time Window 
.window(Slide over 10.minutes every 5.minutes on 'rowtime as 'w) 

// Sliding Processing-time window 
.window(Slide over 10.minutes every 5.minutes on 'proctime as 'w) 

// Sliding Row-count window 
.window(Slide over 10.rows every 5.rows on 'proctime as 'w)
```



**会话窗口** 

会话窗口（Session windows）要用 Session 类来定义，另外还有三个方法： 

- withGap：会话时间间隔 
- on：用来分组（按时间间隔）或者排序（按行数）的时间字段 
- as：别名，必须出现在后面的 groupBy 中

```scala
// Session Event-time Window 
.window(Session withGap 10.minutes on 'rowtime as 'w)

// Session Processing-time Window 
.window(Session withGap 10.minutes on 'proctime as 'w)
```



2. **Over Windows**

   Over window 聚合是标准 SQL 中已有的（Over 子句），可以在查询的 SELECT 子句中定义。 

   Over window 聚合，会针对每个输入行，计算相邻行范围内的聚合。

   Over windows使用.window（w:overwindows*）子句定义，并在 select（）方法中通过别名来引用。

   例子：

   ```scala
   val table = input 
   	.window([w: OverWindow] as 'w) 
   	.select('a, 'b.sum over 'w, 'c.min over 'w)
   ```

   

无界的over window是使用常量指定的。也就是说，时间间隔要指定UNBOUNDED_RANGE， 或者行计数间隔要指定 UNBOUNDED_ROW。而有界的 over window 是用间隔的大小指定的

```scala
// 无界的事件时间 over window (时间字段 "rowtime") 
.window(Over partitionBy 'a orderBy 'rowtime preceding UNBOUNDED_RANGE as 'w) 

//无界的处理时间 over window (时间字段"proctime") 
.window(Over partitionBy 'a orderBy 'proctime preceding UNBOUNDED_RANGE as 'w) 

// 无界的事件时间 Row-count over window (时间字段 "rowtime") 
.window(Over partitionBy 'a orderBy 'rowtime preceding UNBOUNDED_ROW as 'w)

//无界的处理时间 Row-count over window (时间字段 "rowtime") 
.window(Over partitionBy 'a orderBy 'proctime preceding UNBOUNDED_ROW as 'w)
```



有界的 over window

```scala
// 有界的事件时间 over window (时间字段 "rowtime"，之前 1 分钟) 
.window(Over partitionBy 'a orderBy 'rowtime preceding 1.minutes as 'w) 

// 有界的处理时间 over window (时间字段 "rowtime"，之前 1 分钟) 
.window(Over partitionBy 'a orderBy 'proctime preceding 1.minutes as 'w) 

// 有界的事件时间 Row-count over window (时间字段 "rowtime"，之前 10 行) 
.window(Over partitionBy 'a orderBy 'rowtime preceding 10.rows as 'w) 

// 有界的处理时间 Row-count over window (时间字段 "rowtime"，之前 10 行) 
.window(Over partitionBy 'a orderBy 'proctime preceding 10.rows as 'w)
```



**SQL** **中窗口的定义** 

SQL 支持以下 Group 窗口函数: 

- TUMBLE(time_attr, interval) 

  ​	定义一个滚动窗口，第一个参数是时间字段，第二个参数是窗口长度。 

- HOP(time_attr, interval, interval) 

  ​	定义一个滑动窗口，第一个参数是时间字段，第二个参数是窗口滑动步长，第三个是窗口长度。

- SESSION(time_attr, interval)

  ​	定义一个会话窗口，第一个参数是时间字段，第二个参数是窗口间隔（Gap）

```scala
另外还有一些辅助函数，可以用来选择 Group Window 的开始和结束时间戳，以及时间 属性。
这里只写 TUMBLE_*，滑动和会话窗口是类似的（HOP_*，SESSION_*）。 

TUMBLE_START(time_attr, interval) 
TUMBLE_END(time_attr, interval) 
TUMBLE_ROWTIME(time_attr, interval) 
TUMBLE_PROCTIME(time_attr, interval)
```



**Over Windows**

Over 本来就是 SQL 内置支持的语法，所以这在 SQL 中属于基本的聚合操作。所有聚合必须在同一窗口上定义，也就是说，必须是相同的分区、排序和范围。

目前仅支持在当前行范围之前的窗口（无边界和有边界）。

注意，ORDER BY 必须在单一的时间属性上指定。

```SQL
SELECT 
	COUNT(amount) 
OVER ( 
    PARTITION BY user 
    ORDER BY proctime 
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) 
FROM Orders



-- 也可以做多个聚合
SELECT 
	COUNT(amount) OVER w
	, SUM(amount) OVER w 
FROM Orders 
WINDOW w AS (
    PARTITION BY user
    ORDER BY proctime 
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
)
```



例子:开一个滚动窗口，统计 10 秒内出现的每个 sensor 的个数

```scala
def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment 
    env.setParallelism(1) 
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) 
    
    val streamFromFile: DataStream[String] = env.readTextFile("sensor.txt") 
    
    val dataStream: DataStream[SensorReading] = streamFromFile .map( data => { 
        val dataArray = data.split(",") 
        SensorReading(dataArray(0).trim, dataArray(1).trim.toLong, dataArray(2).trim.toDouble) 
    } ) 
    .assignTimestampsAndWatermarks( 
        new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
            override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L 
        } )
    
    val settings: EnvironmentSettings = EnvironmentSettings
    	.newInstance() 
    	.useOldPlanner() 
    	.inStreamingMode() 
    	.build()
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env, settings)
    
    val dataTable: Table = tableEnv.fromDataStream(dataStream, 'id, 'temperature, 'timestamp.rowtime) 
    val resultTable: Table = dataTable
    	.window(Tumble over 10.seconds on 'timestamp as 'tw)
    	.groupBy('id, 'tw) 
    	.select('id, 'id.count) 
    
    val sqlDataTable: Table = dataTable 
    	.select('id, 'temperature, 'timestamp as 'ts) 
    
    val resultSqlTable: Table = tableEnv 
    	.sqlQuery("select id, count(id) from " + sqlDataTable + " group by id,tumble(ts,interval '10' second)") 
    
    // 把 Table 转化成数据流 
    val resultDstream: DataStream[(Boolean, (String, Long))] = resultSqlTable.toRetractStream[(String, Long)] 		
    resultDstream.filter(_._1).print() 
    env.execute() 
}
```



**函数**

**自定义标量函数**

必须在 org.apache.flink.table.functions 中扩展基类 Scalar Function， 并实现（一个或多个）求值（evaluation，eval）方法。

标量函数的行为由求值方法决定求值方法必须公开声明并命名为 eval（直接 def 声明，没有 override）。

求值方法的参数类型和返回类型，确定了标量函数的参数和返回类型

```scala
// 自定义一个标量函数 
class HashCode( factor: Int ) extends ScalarFunction {
    def eval( s: String ): Int = { s.hashCode * factor } 
}

// Table API 中使用
val hashCode = new HashCode(10)
val resultTable = sensorTable .select( 'id, hashCode('id) )

// SQL中使用
tableEnv.registerFunction("hashCode", hashCode) 
val resultSqlTable = tableEnv.sqlQuery("select id, hashCode(id) from sensor")
```



**表函数**

与标量函数不同的是，它可以返回任意数量的行作为输出，而不是单个值

必须扩展 org.apache.flink.table.functions 中的基类 TableFunction 并实现（一个或多个）求值方法。

表函数的行为由其求值方法决定，求值方法必须是 public的，并命名为 eval。求值方法的参数类型，决定表函数的所有有效参数

返回表的类型由 TableFunction 的泛型类型确定。求值方法使用 protected collect（T）方法发出输出行。

在 Table API 中，Table 函数需要与.joinLateral 或.leftOuterJoinLateral 一起使用。



自定义Table Function

```scala
// 自定义 TableFunction 
class Split(separator: String) extends TableFunction[(String, Int)]{ 
    def eval(str: String): Unit = { 
        str.split(separator).foreach( word => collect((word, word.length)) ) 
    } 
}
```

在代码中调用

```scala
// Table API 中调用，需要用 joinLateral 
val resultTable = sensorTable 
	.joinLateral(split('id) as ('word, 'length)) // as 对输出行的字段重命名 
	.select('id, 'word, 'length) 

// 或者用 leftOuterJoinLateral 
val resultTable2 = sensorTable 
	.leftOuterJoinLateral(split('id) as ('word, 'length)) 
	.select('id, 'word, 'length)
```

SQL 的方式：

```sql
tableEnv.createTemporaryView("sensor", sensorTable) 
tableEnv.registerFunction("split", split) 

val resultSqlTable = tableEnv.sqlQuery( 
    """
    |select id, word, length 
    |from 
    |sensor, LATERAL TABLE(split(id)) AS newsensor(word, length) 
    """.stripMargin)
    
// 或者用左连接的方式 
val resultSqlTable2 = tableEnv.sqlQuery(
    """
    |SELECT id, word, length 
    |FROM 
    |sensor 
    | LEFT JOIN 
    | LATERAL TABLE(split(id)) AS newsensor(word, length) 
    | ON TRUE """.stripMargin )
```



**聚合函数**

继承 AggregateFunction 抽象类实现



AggregateFunction 的工作原理如下

- 首先，它需要一个累加器，用来保存聚合中间结果的数据结构（状态）。可以通过调用 AggregateFunction 的 createAccumulator（）方法创建空累加器。 
- 随后，对每个输入行调用函数的 accumulate（）方法来更新累加器。 
- 处理完所有行后，将调用函数的 getValue（）方法来计算并返回最终结果。



AggregationFunction 要求必须实现的方法： 

- createAccumulator()
- accumulate() 
- getValue()



还有一些可选择实现的方法

例如，如果聚合函数应用在会话窗口（session group window）的上下文中，则 merge（）方法是必需的。

- retract() 
- merge() 
- resetAccumulator()



自定义 AggregateFunction，计算一下每个 sensor 的平均温度值

```scala
// 定义 AggregateFunction 的 Accumulator 
class AvgTempAcc { 
    var sum: Double = 0.0 
    var count: Int = 0 
}

class AvgTemp extends AggregateFunction[Double, AvgTempAcc] { 
    override def getValue(accumulator: AvgTempAcc): Double = accumulator.sum / accumulator.count 
    
    override def createAccumulator(): AvgTempAcc = new AvgTempAcc 
    
    def accumulate(accumulator: AvgTempAcc, temp: Double): Unit ={ 
        accumulator.sum += temp 
        accumulator.count += 1 
    } 
}
```

代码中调用

```scala
// 创建一个聚合函数实例 
val avgTemp = new AvgTemp() 

// Table API 的调用 
val resultTable = sensorTable.groupBy('id) 
	.aggregate(avgTemp('temperature) as 'avgTemp) 
	.select('id, 'avgTemp)

// SQL 的实现
tableEnv.createTemporaryView("sensor", sensorTable) 
tableEnv.registerFunction("avgTemp", avgTemp) 

val resultSqlTable = tableEnv.sqlQuery( 
    """
    |SELECT 
    |id, avgTemp(temperature) 
    |FROM 
    |sensor 
    |GROUP BY id 
    """.stripMargin)
```



**表聚合函数**

可以把一个表中数据，聚合为具有多行和多列的结果表

这跟 AggregateFunction 非常类似，只是之前聚合结果是一个标量值，现在变成了一张表



TableAggregateFunction 的工作原理如下。 

- 首先，它同样需要一个累加器（Accumulator），它是保存聚合中间结果的数据结构。通过调用 TableAggregateFunction 的 createAccumulator（）方法可以创建空累加器。 
- 随后，对每个输入行调用函数的 accumulate（）方法来更新累加器。 
- 处理完所有行后，将调用函数的 emitValue（）方法来计算并返回最终结果。



AggregationFunction 要求必须实现的方法：

- createAccumulator() 
- accumulate()

除了上述方法之外，还有一些可选择实现的方法。 

- retract() 
- merge() 
- resetAccumulator() 
- emitValue()
- emitUpdateWithRetract()



自定义 TableAggregateFunction，用来提取每个 sensor 最高的两个温度值。

```scala
// 先定义一个 Accumulator
class Top2TempAcc{ 
    var highestTemp: Double = Int.MinValue 
    var secondHighestTemp: Double = Int.MinValue 
}

// 自定义 TableAggregateFunction 
class Top2Temp extends TableAggregateFunction[(Double, Int), Top2TempAcc]{ 
    override def createAccumulator(): Top2TempAcc = new Top2TempAcc 
    
    def accumulate(acc: Top2TempAcc, temp: Double): Unit ={ 
        if( temp > acc.highestTemp ){ 
            acc.secondHighestTemp = acc.highestTemp 
            acc.highestTemp = temp } 
        else if( temp > acc.secondHighestTemp ){ 
            acc.secondHighestTemp = temp 
        } 
    }
    
    def emitValue(acc: Top2TempAcc, out: Collector[(Double, Int)]): Unit ={ 
        out.collect(acc.highestTemp, 1) 
        out.collect(acc.secondHighestTemp, 2)
    } 
}
```

在代码中调用

```scala
// 创建一个表聚合函数实例 
val top2Temp = new Top2Temp() 
// Table API 的调用 
val resultTable = sensorTable.groupBy('id) 
	.flatAggregate( top2Temp('temperature) as ('temp, 'rank) ) 
	.select('id, 'temp, 'rank)
```





### 场景题

**计算最热门** **Top N** **商品** 

每隔5分钟，统计前一小时商品的pv数

1. 使用窗口聚合统计，由于窗口结束后才会触发计算，现在想来一条数据就计算一条，窗口结束后直接输出结果需要怎么做

```scala
// 使用aggregate方法，传入两个自定义类（AggregateFunction、WindowFunction）
val aggStream: DataStream[ItemViewCount] = dataStream
      .filter(_.behavior == "pv")
      .keyBy("itemId")
      .timeWindow(Time.hours(1), Time.minutes(5))
      /*
       每当一个窗口关闭时候，该方法输出一次结果
       为什么要用aggregate方法：要得到窗口信息，需要用到全窗口函数，但是全窗口函数数据全聚合在一起，实时性不好
                                使用aggregate方法，可以传入窗口函数，而且可以每传一条数据就计算一次，做增量聚合
                                等窗口时间到了后，调用全窗口函数，之前聚合好的状态传入窗口函数，再拿到窗口信息，返回包装好的样例类
      */
      .aggregate(new Countagg(),new ItemViewWindowResult())
```

AggregateFunction实现类

```scala
                              // 传入数据类型 累加器类型 返回数据类型
class Countagg() extends AggregateFunction[UserBehavior, Long, Long]{
  override def createAccumulator(): Long = 0L

  // 每来一条数据，调用一次此方法，此方法返回的就是acc的值，acc值在方法里做了修改
  override def add(in: UserBehavior, acc: Long): Long = acc + 1

  // 得到的结果就是acc
  override def getResult(acc: Long): Long = acc

  // 用在SessionWindow里面做窗口合并
  override def merge(acc: Long, acc1: Long): Long = acc + acc1
}
```

WindowFunction实现类

```scala
                                          // 传入的数据(这里是累加器) 返回的样例类 keyedWindowStream中的key，用tuple包装  窗口类型
class ItemViewWindowResult() extends WindowFunction[Long, ItemViewCount, Tuple, TimeWindow]{
               // keyedWindowStream中的key，用tuple包装  窗口类型  窗口聚合的结果，存在iterator中 返回数据的收集器
               // Iterable：本来是要存放全窗口函数中所有的元素，但是由于只传入了结果，所以只存结果，迭代器长度为1
  override def apply(key: Tuple, window: TimeWindow, input: Iterable[Long], out: Collector[ItemViewCount]): Unit = {
    val itemId = key.asInstanceOf[Tuple1[Long]].f0 // f0 是对应的元素类型
    val windowEnd = window.getEnd
    val count = input.iterator.next()  // 全窗口函数，每个窗口聚合的结果存在iterator
    out.collect(ItemViewCount(itemId, windowEnd, count))
  }
}
```



2. 输出每个窗口pv排名前五的url

```scala
// 使用process function，定时器设置为window end
val resultStream = aggStream
        .keyBy("windowEnd")
        .process(new TopNHotItems(5))
```

KeyedProcessFunction实现类

```scala
class TopNHotItems(topSize: Int) extends KeyedProcessFunction[Tuple, ItemViewCount, String]{

  private var itemViewCountListState: ListState[ItemViewCount] = _

  override def open(parameters: Configuration): Unit = {
    itemViewCountListState = getRuntimeContext.getListState(new ListStateDescriptor[ItemViewCount]("ItemViewCount-List", classOf[ItemViewCount]) )
  }

  override def processElement(value: ItemViewCount, ctx: KeyedProcessFunction[Tuple, ItemViewCount, String]#Context, out: Collector[String]): Unit = {
    //每当来一条数据，加入ListState中
    itemViewCountListState.add(value)

    // 定义一个定时器
    ctx.timerService().registerEventTimeTimer(value.windowEnd + 1)
  }

  // 定时器触发，可以认为数据都已经到齐，可以排序了
  override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[Tuple, ItemViewCount, String]#OnTimerContext, out: Collector[String]): Unit = {
    // 为了方便排序，定义一个ListBuffer，存ListState中的数据
    val allItemViewCounts: ListBuffer[ItemViewCount] = ListBuffer()
    val iter = itemViewCountListState.get().iterator()

    //iterator不可排序，ListBuffer可排序，所以取出放到ListBuffer中
    while(iter.hasNext){
      allItemViewCounts += iter.next()
    }

    //状态清空
    itemViewCountListState.clear()
                                                                // 柯里化，倒序排序
    val sortedItemViewCounts = allItemViewCounts.sortBy(_.count)(Ordering.Long.reverse).take(topSize)

    val result = new StringBuilder
    // 排名信息格式化
    for(i <- sortedItemViewCounts.indices){
      result.append("........")
    }

    Thread.sleep(1000) // 控制显示频率

    out.collect(result.toString())
  }
}
```



**统计每小时的全站PV**

1. 统计每天的pv，数据量太大导致数据倾斜怎么办

   给每条数据分配一个随机key

```scala
al pvStream = dataStream
      .filter(_.behavior == "pv")
      .map(new MyMapper)
      .keyBy(_._1)

class MyMapper extends MapFunction[UserBehavior, (String, Long)]{
  override def map(value: UserBehavior): (String, Long) = {
    (Random.nextString(10),1L)
  }
}
```



2. 使用了随机key之后，控制台输出每个key的结果，没有聚合出全站的pv怎么办

   ​	使用process function，每来一天数据，count值加一，定时器定时为窗口的结束时间

```scala
val totalPvStream = pvStream
         .keyBy(_.windowEnd)  // 注意要先key by windowend
         .process(new TotalPvCountResult()) // 使用process，使用定时器功能让数据到齐后再输出

class TotalPvCountResult() extends KeyedProcessFunction[Long, PvCount, PvCount]{
  // 定义状态，保存所有count总和
  lazy val totalPvCountResultState = getRuntimeContext.getState(new ValueStateDescriptor[Long]("pvCount", classOf[Long]))

  override def processElement(value: PvCount, ctx: KeyedProcessFunction[Long, PvCount, PvCount]#Context, out: Collector[PvCount]): Unit = {
    // count值叠加
    val currentTotalCount = totalPvCountResultState.value()
    totalPvCountResultState.update(currentTotalCount + value.pvCount)

    // 注册定时器
    ctx.timerService().registerEventTimeTimer(value.windowEnd + 1)
  }

  override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[Long, PvCount, PvCount]#OnTimerContext, out: Collector[PvCount]): Unit = {
    val totalPvCount = totalPvCountResultState.value()
    out.collect(PvCount(ctx.getCurrentKey, totalPvCount))
    totalPvCountResultState.clear()
  }
}
```



**统计每小时PV前五的Url，有乱序数据**

1. 乱序数据怎么处理

   ​		允许迟到，超时数据放到侧输出流里面

   main方法：

   ```scala
    val aggStream : DataStream[PageViewCount]= dataStream
         .filter(_.method == "GET")
         .keyBy(_.url)
         .timeWindow(Time.minutes(10), Time.seconds(5))
         .allowedLateness(Time.minutes(1))
         .sideOutputLateData(new OutputTag[ApacheLogEvent]("late"))
         .aggregate(new PageViewCountAgg(), new PageViewCountWindowResult())
   ```

   process方法

   ```scala
   class TopNHotPages(n :Int) extends KeyedProcessFunction[Long, PageViewCount, String]{
     // lazy val pageViewCountListState = getRuntimeContext.getListState(new ListStateDescriptor[PageViewCount]("pageViewCountList", classOf[PageViewCount]))
     lazy val pageViewCountMapState: MapState[String, Long] = getRuntimeContext.getMapState(new MapStateDescriptor[String, Long]("pageViewCountMap", classOf[String], classOf[Long]))
     override def processElement(value: PageViewCount, ctx: KeyedProcessFunction[Long, PageViewCount, String]#Context, out: Collector[String]): Unit = {
       // pageViewCountListState.add(value)
       pageViewCountMapState.put(value.url, value.count)
       // 定义定时器1，窗口时间到达后就输出结果
       ctx.timerService().registerEventTimeTimer(value.windowEnd + 1 )
   
       // 另外定义一个定时器，窗口关闭后清理状态
       ctx.timerService().registerEventTimeTimer(value.windowEnd + 60000L )
     }
   
     override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[Long, PageViewCount, String]#OnTimerContext, out: Collector[String]): Unit = {
       val allPageValueCount: ListBuffer[(String, Long)] = ListBuffer()
   
       // 因为数据流是按照窗口结束时间来分组的，所以getCurrentKey可以得到窗口结束时间
       if(timestamp == ctx.getCurrentKey + 60000L){
         pageViewCountMapState.clear()
         return
       }
   
       val iter = pageViewCountMapState.entries().iterator()
       while(iter.hasNext){
         val entry = iter.next()
         allPageValueCount += ((entry.getKey, entry.getValue))
       }
   
       // 状态清空
       // 为什么不直接清空状态了：因为迟到数据到来之前，定时器已经触发一次,清空数据后，迟到数据到来，就没法排序了，只能等窗口关闭后再清空数据
       // pageViewCountListState.clear()
   
   
       // 按访问量排序并输出TOPN
       val sortedPageViewCount = allPageValueCount.sortWith(_._2 > _._2).take(n)
   
       // 排名信息格式化
       val result = new StringBuilder
       for(i <- sortedPageViewCount.indices){
         result.append("........")
       }
   
       Thread.sleep(1000) // 控制显示频率
   
       out.collect(result.toString())
     }
   }
   
   ```

   

**统计每天的UV**

1. 增量聚合怎么做（因为窗口存不下所有状态）

   ​		每来一条数据，定义一个Trigger，在redis里面聚合

   ``` scala
    val uvStream = dataStream
         .filter(_.behavior == "pv")
         .map(data => ("uv", data.userId))
         .keyBy(_._1)
         .timeWindow(Time.hours(1))
         .trigger(new MyTrigger())   // 自定义触发器
         .process(new UvCountWithBloom()) // 使用位图存储状态，节省State内存空间
   ```

   Trigger类(每来一个数据，直接触发窗口计算，之后清空窗口状态)

   ```scala
   class MyTrigger() extends Trigger[(String, Long), TimeWindow]{
     // 每来一个数据触发此方法
     override def onElement(element: (String, Long), timestamp: Long, window: TimeWindow, ctx: Trigger.TriggerContext): TriggerResult = TriggerResult.FIRE_AND_PURGE
     // ProcessTime改变时候触发此方法
     override def onProcessingTime(time: Long, window: TimeWindow, ctx: Trigger.TriggerContext): TriggerResult = TriggerResult.CONTINUE
     // EventTime改变时候触发此方法
     override def onEventTime(time: Long, window: TimeWindow, ctx: Trigger.TriggerContext): TriggerResult = TriggerResult.CONTINUE
     // 清理工作
     override def clear(window: TimeWindow, ctx: Trigger.TriggerContext): Unit = {}
   }
   ```

   

2. 每天的UV缓存起来数据太多占用空间太大怎么办

   ​	因为只是统计UV，使用布隆过滤器即可，借用redis的位图来实现，在process方法中做

   ​	当来了一条数据，数据hash化后看对应的位图地址有没有置为1，没有的话就置为1，然后redis里存uv count的数据加一

   ​    如果对应位图地址的数据已经置为1，标表示是同一用户，uv count的数据不做操作

   ```scala
   class UvCountWithBloom() extends ProcessWindowFunction[(String, Long), UvCount, String, TimeWindow]{
     // 定义redis连接和布隆过滤器
     lazy val jedis = new Jedis("localhost", 6379)
     lazy val bloomFilter = new Bloom(1<<29)  //
   
     // 本来是收集了所有数据，窗口触发时候才会计算，但是由于定义了触发器，每来一条数据都会有一次窗口计算
     override def process(key: String, context: Context, elements: Iterable[(String, Long)], out: Collector[UvCount]): Unit = {
       // redis 的位运算可以当场扩展，先定义redis存储位图的key
       val storedBitmapKey = context.window.getEnd.toString
   
       // 当前窗口的uv count值，保存在redis hash表里面（windowEnd, uv count）
       val uvCountMap = "uvcount"
       var count = 0L
       val currentKey = context.window.getEnd.toString
       // redis中取出当前窗口key的值
       if(jedis.hget(uvCountMap, currentKey) != null){
         count = jedis.hget(uvCountMap, currentKey).toLong
       }
   
       // 由于求uv，所以要对用户去重：先判断userId对应hash位图位置count是否为0
       val userId = elements.last._2.toString
       // 计算hash值，对应位图中的偏移量
       val offset = bloomFilter.hash(userId, 61)
       // 位操作命令，取出对应值
       val isExists = jedis.getbit(storedBitmapKey, offset)
       if(!isExists){
         // 如果不存在，位图对应位置置为1，count值加1
         jedis.setbit(storedBitmapKey, offset, true)
         jedis.hset(uvCountMap, currentKey, (count + 1).toString)
       }
     }
   }
   ```

   

**刷单行为检测**

1. 每天下单量超过100的人加入黑名单，第二天0点解禁

   ​	先定义一个每天0点的定时器，时间一到清空存黑名单的状态

      判断下单有没有达到100，达到的话加入黑名单

   main方法：

   ```scala
   // 按照广告id和userId分组，点击量超过100的用户进入黑名单放入测输出流中
   val filterBlackListUserStream: DataStream[AdClickLog] = adLogStream
         .keyBy(x => (x.adId, x.userId))
         .process(new FilterBlackListUserResult(100))
   ```

   process方法

   ```scala
   class FilterBlackListUserResult(maxCount: Long) extends KeyedProcessFunction[(Long, Long), AdClickLog, AdClickLog]{
     // 定义状态储存用户点击量
     lazy val countState = getRuntimeContext.getState(new ValueStateDescriptor[Long]("count", classOf[Long]))
     // 定义每天0点定时清空状态时间戳
     lazy val resetTimerTsState = getRuntimeContext.getState(new ValueStateDescriptor[Long]("reset-ts", classOf[Long]))
     // 定义布尔状态保存用户有没有进过黑名单
     lazy val isBlackState = getRuntimeContext.getState(new ValueStateDescriptor[Boolean]("reset-ts", classOf[Boolean]))
   
     override def processElement(value: AdClickLog, ctx: KeyedProcessFunction[(Long, Long), AdClickLog, AdClickLog]#Context, out: Collector[AdClickLog]): Unit = {
       val currCount = countState.value()
   
       // 只要是第一个数据来了，直接定义一个定时器
       if(currCount == 0){
         val ts = (ctx.timerService().currentProcessingTime() / (1000 * 60 *60 *24) + 1) * (1000 * 60 * 60 * 24) - 8 *  (1000 * 60 * 60 * 24)
         resetTimerTsState.update(ts)
         ctx.timerService().registerProcessingTimeTimer(ts)
       }
   
       //判断count值是否已经达到判断的阈值
       if(currCount >= maxCount){
         // 判断是否已经在黑名单里面
         if(!isBlackState.value()){
           isBlackState.update(true)
           ctx.output(new OutputTag[BlackListUserWarning]("BlackList"), BlackListUserWarning(value.userId, value.adId, s"点击超过$maxCount 次"))
         }
         return
       }
   
       // 正常情况，count原样 + 1 ，正常输出
       countState.update(currCount + 1)
       println(ctx.getCurrentKey,this,countState.value())
       out.collect(value)
     }
   
     override def onTimer(timestamp: Long, ctx: KeyedProcessFunction[(Long, Long), AdClickLog, AdClickLog]#OnTimerContext, out: Collector[AdClickLog]): Unit = {
       if(timestamp == resetTimerTsState.value()){
         countState.clear()
         resetTimerTsState.clear()
         isBlackState.clear()
       }
     }
   }
   ```

   



**恶意登录行为检测**

1. 当2s内连续登录失败两次，输出警告

   ​	可以使用Flink CEP实现

   main

   ```scala
   // 定义CEP模式
   val loginFailPattern = Pattern
           .begin[LoginEvent]("firstFail").where(_.eventType == "fail")
           .next("secondFail").where(_.eventType == "fail")
           .within(Time.seconds(2))
   
   // 模式应用到数据流
   val patternStream = CEP.pattern(loginEventStream.keyBy(_.userId), loginFailPattern)
   
   // 得到匹配出的数据流
   val LoginFailWarning = patternStream.select(new LoginFailEventMatch())
   ```

   自定义CEP select方法

   ```scala
   class LoginFailEventMatch() extends PatternSelectFunction[LoginEvent, LoginFailWarning]{
     // map key：命名（firstFail or secondFail） value: 匹配值，有可能匹配多个，所以用list
     override def select(map: util.Map[String, util.List[LoginEvent]]): LoginFailWarning = {
       val firstFaiEvent = map.get("firstFail").get(0)
       val secondFailEvent = map.get("secondFail").iterator().next()
       LoginFailWarning(firstFaiEvent.userId, firstFaiEvent.timestamp, secondFailEvent.timestamp, "login fail")
     }
   }
   ```

   



订单超时检测

1. 找出超过15分钟未支付订单

   ​	使用CEP实现

   main

   ```scala
   // 定义Pattern
   val orderPayPattern = Pattern
         .begin[OrderEvent]("create").where(_.eventType == "create")
         .followedBy("pay").where(_.eventType == "pay")
         .within(Time.minutes(15))
   
   // pattern应用到数据流上
   val patternStream = CEP.pattern(orderEventStream, orderPayPattern)
   
   // 超时数据放到侧输出流上
   val orderTimeoutOutputTag = new OutputTag[OrderResult]("orderTimeout")
   val resultStream = patternStream.select(orderTimeoutOutputTag,
         // 超时输出侧输出流
         new OrderTimeoutSelect(),
         // 未超时输出正常流
         new OrderPaySelect()
       )
   ```

   实现PatternTimeoutFunction、PatternSelectFunction

   ```scala
   // 输出超时流
   class OrderTimeoutSelect() extends PatternTimeoutFunction[OrderEvent, OrderResult]{
     // map: 匹配的超时事件集合  l: 超时所在时间戳
     override def timeout(map: util.Map[String, util.List[OrderEvent]], l: Long): OrderResult = {
       val timeoutOrderId = map.get("create").iterator().next().orderId
       OrderResult(timeoutOrderId, "timeout: " +  l)
     }
   }
   // 输出正常流
   class OrderPaySelect() extends PatternSelectFunction[OrderEvent, OrderResult]{
     override def select(map: util.Map[String, util.List[OrderEvent]]): OrderResult = {
       val payOrderId = map.get("pay").iterator().next().orderId
       OrderResult(payOrderId, "payed success")
     }
   }
   ```





**双流Join**

两种不同的流join得到新流

main方法

```scala
  // 订单数据
val resource = getClass.getResource("OrderLog.csv")
val orderEventStream = env.readTextFile(resource.getPath)
      .map(data => {
        val arr = data.split(",")
        OrderEvent(arr(0).toLong, arr(1), arr(2), arr(3).toLong)
      })
      .filter(_.eventType == "pay")
      .assignAscendingTimestamps(_.timestamp * 1000L)
      .keyBy(_.txId)

// 到账数据
val resource2 = getClass.getResource("ReceiptLog.csv")
val receiptEventStream = env.readTextFile(resource2.getPath)
      .map(data => {
        val arr = data.split(",")
        ReceiptEvent(arr(0), arr(1), arr(2).toLong)
      })
      .assignAscendingTimestamps(_.timestamp * 1000L)
      .keyBy(_.txId)

// 合流
val resultStream = orderEventStream.intervalJoin(receiptEventStream)
      .between(Time.seconds(-3), Time.seconds(5))
      .process(new TxMatchWithJoinResult())

```

实现ProcessJoinFunction

```scala
class TxMatchWithJoinResult() extends ProcessJoinFunction[OrderEvent, ReceiptEvent, (OrderEvent, ReceiptEvent)]{
  override def processElement(left: OrderEvent, right: ReceiptEvent, ctx: ProcessJoinFunction[OrderEvent, ReceiptEvent, (OrderEvent, ReceiptEvent)]#Context, out: Collector[(OrderEvent, ReceiptEvent)]): Unit = {
    out.collect((left, out))
  }
}
```



注意：

​	双流intervalJoin，以第一条流为基准，每一条数据都匹配到另一条流里面对应时间范围内所有数据

​	基于KeyedStream提供的interval join机制，intervaljoin 连接两个keyedStream, 按照相同的key在一个相对数据时间的时间段内进行连接

​	合流的时候，上面例子因为已经做了keyBy，所以自动按照key相等条件来join

​	如果是两个非keyedStream做join，那么join完毕后，还需要做一个keyBy才行



**双流join2**

订单数据与到账数据对比，找出下单但是到账数据没查到的流

上面的intervalJoin方法只能得到匹配的流，这里要得到不匹配的流，

main：两条流用connect方法连在一起

```scala
// 合流,如果之前两条流做过KeyBy，connect时候就已经按照key连接在一起了
val resultStream = orderEventStream.connect(receiptEventStream)
      .process(new TxPayMatchResult())
```



实现Co Process方法

有时候可能是订单数据先到，有时候可能是到账数据先到

所以两种情况都要判断下

```scala
class TxPayMatchResult() extends CoProcessFunction[OrderEvent, ReceiptEvent, (OrderEvent, ReceiptEvent)]{
  // 定义状态，保存交易对应的订单状态和到账状态
  lazy val payEventState = getRuntimeContext.getState[OrderEvent](new ValueStateDescriptor[OrderEvent]("pay", classOf[OrderEvent]))
  lazy val receiptEventState = getRuntimeContext.getState[ReceiptEvent](new ValueStateDescriptor[ReceiptEvent]("receipt", classOf[ReceiptEvent]))
  // 侧输出流标签
  val unmatchedPayEventOutputTag = new OutputTag[OrderEvent]("unmatched-pay")
  val unmatchedReceiptEventOutputTag = new OutputTag[OrderEvent]("unmatched-receipt")

  override def processElement1(pay: OrderEvent, ctx: CoProcessFunction[OrderEvent, ReceiptEvent, (OrderEvent, ReceiptEvent)]#Context, out: Collector[(OrderEvent, ReceiptEvent)]): Unit = {
    // 订单支付事件来了，要判断是否有到账事件
    val receipt = receiptEventState.value()
    if(receipt != null){
      // 如果已经有到账，正常输出匹配，清空状态
      out.collect((pay, receipt))
      receiptEventState.clear()
    }else{
      // 如果没来，注册定时器等待5s
      ctx.timerService().registerEventTimeTimer(pay.timestamp * 1000L + 5000L)
      // 状态更新
      payEventState.update(pay)
    }
  }

  override def processElement2(receipt: ReceiptEvent, ctx: CoProcessFunction[OrderEvent, ReceiptEvent, (OrderEvent, ReceiptEvent)]#Context, out: Collector[(OrderEvent, ReceiptEvent)]): Unit = {
    // 到账事件来了，要判断是否有支付事件
    val pay = payEventState.value()
    if(pay != null){
      // 如果已经有支付，正常输出匹配，清空状态
      out.collect((pay, receipt))
      payEventState.clear()
    }else{
      // 如果没来，注册定时器等待3s
      ctx.timerService().registerEventTimeTimer(pay.timestamp * 1000L + 3000L)
      // 状态更新
      receiptEventState.update(receipt)
    }
  }

  override def onTimer(timestamp: Long, ctx: CoProcessFunction[OrderEvent, ReceiptEvent, (OrderEvent, ReceiptEvent)]#OnTimerContext, out: Collector[(OrderEvent, ReceiptEvent)]): Unit = {
    // 定时器触发，判断哪个状态没存在，就是另一个事件没来，输出到侧输出流
    if(payEventState.value() != null){
      ctx.output(unmatchedPayEventOutputTag, payEventState.value())
    }

    if(receiptEventState.value() != null){
      ctx.output(unmatchedReceiptEventOutputTag, receiptEventState.value())
    }

    // 清空状态
    payEventState.clear()
    receiptEventState.clear()
  }
}
```

