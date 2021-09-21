# Flink CDC

### Flink CDC产生背景

CDC 是 Change Data Capture（变更数据获取）的简称。核心思想是，监测并捕获数据库的变动（包括数据或数据表的插入、更新以及删除等），将这些变更按发生的顺序完整记录下来，写入到消息中间件中以供其他服务进行订阅及消费

Flink基于读取数据库Binlog来捕捉数据变化，可以实现**同步全量数据**以及**捕捉增量数据**，Flink CDC默认使用Debezium来读取binlog



### 代码方式读取

这里使用MySQL作为数据源，

```scala
import com.alibaba.ververica.cdc.connectors.mysql.MySQLSource;
import com.alibaba.ververica.cdc.debezium.DebeziumSourceFunction;
import com.alibaba.ververica.cdc.debezium.StringDebeziumDeserializationSchema;
import org.apache.flink.api.common.restartstrategy.RestartStrategies;
import org.apache.flink.runtime.state.filesystem.FsStateBackend;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.CheckpointConfig;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import java.util.Properties;

public class FlinkCDC {
 public static void main(String[] args) throws Exception {
 //1.创建执行环境
 StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
 env.setParallelism(1);
 //2.Flink-CDC 将读取 binlog 的位置信息以状态的方式保存在 CK,如果想要做到断点续传,需要从 Checkpoint 或者 Savepoint 启动程序
 //2.1 开启 Checkpoint,每隔 5 秒钟做一次 CK
 env.enableCheckpointing(5000L);
 //2.2 指定 CK 的一致性语义
 env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
 //2.3 设置任务关闭的时候保留最后一次 CK 数据
env.getCheckpointConfig().enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
 //2.4 指定从 CK 自动重启策略
 env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 2000L));
 //2.5 设置状态后端
 env.setStateBackend(new FsStateBackend("hdfs://hadoop102:8020/flinkCDC"));
 //2.6 设置访问 HDFS 的用户名
 System.setProperty("HADOOP_USER_NAME", "atguigu");
 //3.创建 Flink-MySQL-CDC 的 Source
 //initial (default): Performs an initial snapshot on the monitored database tables upon 
first startup, and continue to read the latest binlog.
 //latest-offset: Never to perform snapshot on the monitored database tables upon first 
startup, just read from the end of the binlog which means only have the changes since the 
connector was started.
 //timestamp: Never to perform snapshot on the monitored database tables upon first 
startup, and directly read binlog from the specified timestamp. The consumer will traverse the 
binlog from the beginning and ignore change events whose timestamp is smaller than the 
specified timestamp.
 //specific-offset: Never to perform snapshot on the monitored database tables upon 
first startup, and directly read binlog from the specified offset.
 DebeziumSourceFunction<String> mysqlSource = MySQLSource.<String>builder()
 .hostname("hadoop102")
 .port(3306)
 .username("root")
 .password("000000")
 .databaseList("gmall-flink")
 .tableList("gmall-flink.z_user_info") //可选配置项,如果不指定该参数,则会读取上一个配置下的所有表的数据，注意：指定的时候需要使用"db.table"的方式
 .startupOptions(StartupOptions.initial())
 .deserializer(new StringDebeziumDeserializationSchema())
```



读取数据库有四种模式

1. initial模式：在启动时，先对已有的数据做一个快照，当已有数据导入完成时，启动增量同步任务
2. latest-offset模式：启动时只读取binlog的末尾
3. timestamp模式：从指定的时间戳开始读取
4. specific-offset模式：从指定偏移量读取



需要注意的点：

​	当选择表的时候，db名也要加上





### Flink SQL方式读取

flink sql方式一次只能读取一张表

优点：不需要自定义反序列化器来格式化消息，内部自己已经做好了

```scala
import org.apache.flink.api.common.restartstrategy.RestartStrategies;
import org.apache.flink.runtime.state.filesystem.FsStateBackend;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.environment.CheckpointConfig;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

public class FlinkSQL_CDC {
 public static void main(String[] args) throws Exception {
 //1.创建执行环境
 StreamExecutionEnvironment env = 
StreamExecutionEnvironment.getExecutionEnvironment();
 env.setParallelism(1);
 StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
 //2.创建 Flink-MySQL-CDC 的 Source
 tableEnv.executeSql("CREATE TABLE user_info (" +
 " id INT," +
 " name STRING," +
 " phone_num STRING" +
 ") WITH (" +
 " 'connector' = 'mysql-cdc'," +
 " 'hostname' = 'hadoop102'," +
 " 'port' = '3306'," +
 " 'username' = 'root'," +
 " 'password' = '000000'," +
 " 'database-name' = 'gmall-flink'," +
 " 'table-name' = 'z_user_info'" +
 ")");
 tableEnv.executeSql("select * from user_info").print();
 env.execute();
 } }
```



### 自定义反序列化器

有时候返回的消息格式可能不符合我们的要求，所以需要自己重写反序列化器

```scala
import com.alibaba.fastjson.JSONObject;
import com.alibaba.ververica.cdc.connectors.mysql.MySQLSource;
import com.alibaba.ververica.cdc.debezium.DebeziumDeserializationSchema;
import com.alibaba.ververica.cdc.debezium.DebeziumSourceFunction;
import io.debezium.data.Envelope;
import org.apache.flink.api.common.restartstrategy.RestartStrategies;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.runtime.state.filesystem.FsStateBackend;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.environment.CheckpointConfig;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;
import org.apache.kafka.connect.data.Field;
import org.apache.kafka.connect.data.Struct;
import org.apache.kafka.connect.source.SourceRecord;
import java.util.Properties;

public class Flink_CDCWithCustomerSchema {
 	public static void main(String[] args) throws Exception {
        
 	//1.创建执行环境
 	StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
 	env.setParallelism(1);
        
 	//2.创建 Flink-MySQL-CDC 的 Source
 	DebeziumSourceFunction<String> mysqlSource = MySQLSource.<String>builder()
 		.hostname("hadoop102")
 		.port(3306)
 		.username("root")
 		.password("000000")
 		.databaseList("gmall-flink")
 		.tableList("gmall-flink.z_user_info") //可选配置项,如果不指定该参数,则会读取上一个配置下的所有表的数据,注意：指定的时候需要使用"db.table"的方式
		.startupOptions(StartupOptions.initial())
 		.deserializer(new CustomerDeserializationSchema())
        .build() ;
        
 	//3.使用 CDC Source 从 MySQL 读取数据
 	DataStreamSource<String> mysqlDS = env.addSource(mysqlSource);
        
        
        
public class CustomerDeserializationSchema extends DebeziumDeserializationSchema<String>{ 
    //自定义数据解析器
	@Override
 	public void deserialize(SourceRecord sourceRecord, Collector<String>collector) throws Exception {
 	   //获取主题信息,包含着数据库和表名 mysql_binlog_source.gmall-flink.z_user_info
       String topic = sourceRecord.topic();
       String[] arr = topic.split("\\.");
       String db = arr[1];
       String tableName = arr[2];
       //获取操作类型 READ DELETE UPDATE CREATE,需要注意，只能从工具类里面得到
 	   Envelope.Operation operation = Envelope.operationFor(sourceRecord);
       //获取值信息并转换为 Struct 类型,Struct是Kafka中的一个类
       Struct value = (Struct) sourceRecord.value();
 	   //获取变化后的数据
       Struct after = value.getStruct("after");
 	   //创建 JSON 对象用于存储数据信息
 	   JSONObject data = new JSONObject();
 	   for (Field field : after.schema().fields()) {
 			Object o = after.get(field);
 			data.put(field.name(), o);
 		}
		//创建 JSON 对象用于封装最终返回值数据信息
 		JSONObject result = new JSONObject();
        result.put("operation", operation.toString().toLowerCase());
        result.put("data", data);
        result.put("database", db);
        result.put("table", tableName);
        //发送数据至下游
        collector.collect(result.toJSONString());
 	}
    
	@Override
	public TypeInformation<String> getProducedType() {
 		return TypeInformation.of(String.class);
	}
 })

```



### CDC 2.0解决的问题

#### 无锁读取

**存在问题**：在初始化模式读取已有数据的时候，1.0版本需要锁表，因为不知道什么时候会更新数据但是数据不被捕捉

**解决办法**：要求表有主键

1. 当读取数据时，记录下当前的 Binlog 位置信息，作为低位点，全量数据读取完后，记录下此时时间戳作为高位点，读取出的数据放在buffer里面
2. 在增量部分消费从低位点到高位点的 Binlog，根据主键，对 buffer 中的数据进行修正并输出



#### 并发读取

**存在问题**：1.0版本不能并发读取数据

**解决办法**：

1. 对于目标表按照主键进行数据分片，设置每个切片的区间为左闭右开或者左开右闭来保证数据的连续性

2. 将划分好的 Chunk 分发给多个 SourceReader，每个 SourceReader 读取表中的一部分数据，实现了并行读取的目标
3. 在 Snapshot Chunk 读取完成之后，有一个汇报的流程，汇报完成后，全量任务完成
4. 这里参考了Netflix DBlog 的论文中的无锁算法原理



**存在问题**：并发读取时候，有数据更改怎么办

**解决办法**：

1. 在增量部分对数据补齐
2. 每个Chunk 的低位点和高位点都不一样，针对每个Chunk，已经读取的binlong不能重复读取，防止数据重复
3. 把每个Chunk对应的高低位点都记录下来
4. 读取后面的binlog，不是对数据改变操作的binlong（如insert数据）无条件输出
5. 读取后面的binlog，输出高于全局位点的**改变**数据无条件返回作为更新数据
6. 遍历每个Chunk切片信息，如果新来的数据属于当前切片的主键区间，且位点大于当前切片的高位点，输出这个数据
7. 不符合以上条件的数据不输出



**存在问题**：1.0 checkpoint出错了要重头再来

**解决办法**：

1. 在每个 Chunk 读取的时候可以单独做 CheckPoint，某个 Chunk 读取失败只需要单独执行该 Chunk 的任务，而不需要像 1.x 中失败了只能从头读取。