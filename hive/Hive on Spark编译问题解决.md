## 1. 与Hadoop兼容问题

Hadoop版本：3.1.3

Hive版本：3.1.3

启动Hive报错

```
java.lang.NoSuchMethodError: com.google.common.base.Preconditions.checkArgument(ZLjava/lang/String;Ljava/lang/Object;)V
```

原因：

对比两个框架的Guava的pom文件发现，Hadoop Guava和Hive Guava依赖版本不一致

Hadoop-3.1.3版本，guava依赖的版本突然升级到了guava-27.0-jre。

Hive-3的所有发行版本的guava依赖均为guava-19.0。



**解决步骤与思路**

查看每个版本Guava中checkArgument方法的变更历史，发现在某个版本，hadoop使用到的Guava checkArgument方法打上了废弃标志，原因是这个方法爆出了漏洞。

一般被废弃的方法，都有注释写明可以用哪个方法代替。将Hive中Guava的依赖升级，将原checkArgument方法方法改为注释中写的新的 checkArgument方法。并重新编译Hive即可。



## 2. Hive插入数据StatsTask失败

insert语句报错：

```
FAILED: Execution Error, return code 1 from org.apache.hadoop.hive.ql.exec.StatsTask
```

上述问题是由Hive自身存在的bug所致，bug详情可参照以下连接https://issues.apache.org/jira/browse/HIVE-19316。



该bug已经在3.2.0, 4.0.0, 4.0.0-alpha-1等版本修复了，所以可以参考修复问题的PR，再修改Hive源码并重新编译。把这个PR的版本制作为补丁，应用后编译出Jar包即可。



## 3. Spark兼容问题

使用Spark引擎，insert语句报错：

```
Job failed with java.lang.NoSuchMethodError: org.apache.spark.api.java.JavaSparkContext.accumulator(Ljava/lang/Object;Ljava/lang/String;Lorg/apache/spark/AccumulatorParam;)Lorg/apache/spark/Accumulator;
```



原因：

Hive中自带Spark为2.3.0。目前集群中使用的Spark是3.1.3



**解决步骤与思路**

将spark依赖的版本改为3.1.3

按照解决Guava的思路，将问题重新修复一遍。该问题与问题1是同一类问题。