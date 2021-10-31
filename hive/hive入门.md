## hive使用笔记

### metastore

**Metadata**：元数据。元数据包含用Hive创建的database、table、表的字段等元信息。元数据存储在关系型数据库中。如hive内置的Derby、第三方如MySQL等。

**Metastore**：元数据服务，作用是：客户端连接metastore服务，metastore再去连接MySQL数据库来存取元数据。有了metastore服务，就可以有多个客户端同时连接，而且这些客户端不需要知道MySQL数据库的用户名和密码，只需要连接metastore 服务即可。

**Metastore**三种启动方式：

​		**1. 内嵌模式**，使用的是内嵌的Derby数据库来存储元数据，也不需要额外起Metastore服务。数据库和Metastore服务都嵌入在主Hive Server进程中。这个是默认的，配置简单，但是一次只能一个客户端连接，适用于用来实验，不适用于生产环境。

​		解压hive安装包 bin/hive 启动即可使用

​		缺点：不同路径启动hive，每一个hive拥有一套自己的元数据，无法共享。



​		**2.本地模式**采用外部数据库来存储元数据，目前支持的数据库有：MySQL、Postgres、Oracle、MS SQL Server.在这里我们使用MySQL。

​		本地模式不需要单独起metastore服务，用的是跟hive在同一个进程里的metastore服务。也就是说当你启动一个hive 服务，里面默认会帮我们启动一个metastore服务。

​		hive根据hive.metastore.uris 参数值来判断，**如果为空，则为本地模式**。

​		缺点是：每启动一次hive服务，都内置启动了一个metastore。

​	**3.远程模式**下，需要单独起metastore服务，然后每个客户端都在配置文件里配置连接到该metastore服务。远程模式的metastore服务和hive运行在不同的进程里。

​	在生产环境中，建议用远程模式来配置Hive Metastore。

​	在这种情况下，其他依赖hive的软件都可以通过Metastore访问hive。

### 创建表

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
   [(col_name data_type [COMMENT col_comment], ...)] 
   [COMMENT table_comment] 
   [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
   [CLUSTERED BY (col_name, col_name, ...) 
   [SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
   [ROW FORMAT row_format] 
   [STORED AS file_format] 
   [LOCATION hdfs_path]
```

```shell
[EXTERNAL]:用于创建外部表，与后面的location '/PATH'一起用
PARTITIONED BY：用于创建分区
CLUSTERED BY：用于分桶
LIKE： 允许用户复制现有的表结构，但是不复制数据
ROW FORMAT row_format：按照指定样的方式读取，一般是ROW FORMAT delimited按行读取==和SERDE serde_name二选一==
STORED AS file_format：指定文件存储格式，不写就是默认textfile
```

### 修改表

#### 增加分区

增加一个分区

```sql
alter table table_name add partition (分区='自定义分区名') location '/user/hive/warehouse/database_name/table_name/分区='自定义分区名''

hadoop fs -put 文件 /user/hive/warehouse/database_name/table_name/分区=自定义分区名
```

增加多个分区

```sql
ALTER TABLE table_name ADD PARTITION (dt='2008-08-08', country='us') location
 '/path/to/us/part080808' PARTITION (dt='2008-08-09', country='us') location
 '/path/to/us/part080809';  //一次添加多个分区
```

#### 删除分区

```sql
alter table table_name drop if exists partition(分区名 = '具体分区名');
```

#### 修改分区

```sql
ALTER TABLE table_name PARTITION (dt='2008-08-08') RENAME TO PARTITION (dt='20080808');
```

#### 修改列

```sql
修改列
test_change (a int, b int, c int);

ALTER TABLE test_change CHANGE a a1 INT; //修改a字段名


修改a，b列，但a执行要晚于b
// will change column a's name to a1, a's data type to string, and put it after column b. The new table's structure is: b int, a1 string, c int
ALTER TABLE test_change CHANGE a a1 STRING AFTER b; 

// will change column b's name to b1, and put it as the first column. The new table's structure is: b1 int, a ints, c int
ALTER TABLE test_change CHANGE b b1 INT FIRST; 
```

#### 表命令

show tables;                                       显示当前数据库所有表

show databases |schemas;             显示所有数据库

show partitions table_name;           显示分区

desc formatted table_name;           格式化美观的显示表的metadata数据





### 建立分区表

```sql
--1.建立分区表
create table table_name(id int,name string,age int,country string) partitioned by (guojia string) row format delimited fields terminatered by ','
--注意，分区用到的字符和表内的列不能同名

--2.通过load命令把数据加载进表内，不可以使用hadoop fs -put
load data local inpath '/路径' into table table_name partition (guojia = 'XXX');
```

分区表使用

```sql
select * from table_name where country = 'XXX';

select * from table_name where guojia = 'XXX';
```





### 建立分桶表

作用：当做普通表一样查询使用 只不过底层在join查询的时候会进行优化

```sql
--建立目标表
create table bucket(Sno int,Sname string,Sex string,Sage int,Sdept string) clustered by (Sno) into 4 buckets row format delimited fields terminated by ',';

--建立临时表
create table bucket_tmp(Sno int,Sname string,Sex string,Sage int,Sdept string)  row format delimited fields terminated by ',';

--设置reduces数目，要和桶数目一样
set hive.enforce.bucketing = true;
set mapreduce.job.reduces=4;

--加载文件到临时表里
load data local inpath ‘/PATH’ into table bucket_tmp; --hadoop fs -put也行

--把分桶查询结果写入目标表
insert overwrite taable bucket select * from bucket_tmp cluster by(Sno)
```

分桶表的注意事项

- 分桶表的字段必须是表中的字段
- 分桶表也是一种优化表 意味着建表的时候可以选择是否创建
- 分桶表主要**优化join查询 减少笛卡尔积**



### insert使用

#### 多重插入

```sql
from table_name
insert into table table_name1 select id
insert into table table_name2 select name
--从table_name中搜索到的id列插入table_name1中，搜索到的name列插入table_name2中
```



#### 动态插入

```sql
--假设内容如下
2015-05-10,ip1
2015-05-10,ip2
2015-06-14,ip3
2015-06-14,ip4
2015-06-15,ip1
2015-06-15,ip2

---创建目标表
create table table_name(ip string) partitioned by (month string,day string);

--创建临时表
create table table_name_tmp(day  string,ip string) row format delimited fields terminated by ',';

--开启动态分区，关闭严格模式
set hive.exec.dynamic.partition=true;    --是否开启动态分区功能，默认false关闭。
set hive.exec.dynamic.partition.mode=nonstrict;   
--动态分区的模式，默认strict，表示必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。
insert overwrite table d_p_t partition (month,day) 
select ip,substr(day,1,7) as month,day from dynamic_partition_table;

select substr(day,1,7) as month,day,ip from dynamic_partition_table; 
--错误的 动态分区是根据位置来确定的
```





### 分桶查询

```sql
select * from table_name cluster by (列名)
--指定分桶个数（按照指定来）
set mapreduce.job.reduces=2;
Number of reduce tasks not specified. Defaulting to jobconf value of: 2
--现象：cluster by  会根据指定的字段分桶 并且分桶内根据这个字段排序（正序）
--分且排序（字段）
```

#### 排序

不能用cluster by和sort by配合，需要改用distribute by

例如：

```sql
select * from student DISTRIBUTE by(Sno) sort by(sage);
```

用cluster by和sort by配合会报错

#### order by全局排序

无视规则，直接把reduces的个数限定为1



### hive函数

#### 内置函数

创建虚表daul,可以自己练习

```shell
创建一个 dual 表
create table dual(id string);
load 一个文件（只有一行内容：内容为一个空格）到 dual 表
select substr('angelababy',2,3) from dual;
```

按照word文档练习



#### 自定义函数

在idea中导入相应的jar包和插件

自定义类中继承UT类，并可以多次重载evaluate方法

之后再hive中利用add jar命令： hive> add jar /linux内jar包路径

再hive里面自定义临时函数名，断开连接，名字失效：

```sql
create temporary function 临时函数名 as '项目路径';  

--之后可以使用
select 临时函数名(参数1,参数2....) from table_name;
```





### hive join方式

数据：

```sql
+-------+---------+--+
| a.id  | a.name  |
+-------+---------+--+
| 1     | a       |
| 2     | b       |
| 3     | c       |
| 4     | d       |
| 7     | y       |
| 8     | u       |
+-------+---------+--+

+-------+---------+--+
| b.id  | b.name  |
+-------+---------+--+
| 2     | bb      |
| 3     | cc      |
| 7     | yy      |
| 9     | pp      |
+-------+---------+--+
```

#### outer join

```sql
select * from a full outer join b on a.id = b.id;
+-------+---------+-------+---------+--+
| a.id  | a.name  | b.id  | b.name  |
+-------+---------+-------+---------+--+
| 2     | b       | 2     | bb      |
| 4     | d       | NULL  | NULL    |
| 8     | u       | NULL  | NULL    |
| 1     | a       | NULL  | NULL    |
| 3     | c       | 3     | cc      |
| 7     | y       | 7     | yy      |
| NULL  | NULL    | 9     | pp      |
+-------+---------+-------+---------+--+
--外关联显示左右两边不管是否join上 都显示
```

#### inner join

```sql
select * from a inner join b on a.id = b.id;

+-------+---------+-------+---------+--+
| a.id  | a.name  | b.id  | b.name  |
+-------+---------+-------+---------+--+
| 2     | b       | 2     | bb      |
| 3     | c       | 3     | cc      |
| 7     | y       | 7     | yy      |
+-------+---------+-------+---------+--+
--内关联只显示join上左右两边结果
```

#### left join

```sql
select * from a left join b on a.id = b.id;

+-------+---------+-------+---------+--+
| a.id  | a.name  | b.id  | b.name  |
+-------+---------+-------+---------+--+
| 1     | a       | NULL  | NULL    |
| 2     | b       | 2     | bb      |
| 3     | c       | 3     | cc      |
| 4     | d       | NULL  | NULL    |
| 7     | y       | 7     | yy      |
| 8     | u       | NULL  | NULL    |
+-------+---------+-------+---------+--+
```

#### left semi join

```sql
select * from a left semi join b on a.id = b.id;
相当于
select a.* from a inner join b on a.id = b.id;


+-------+---------+--+
| a.id  | a.name  |
+-------+---------+--+
| 2     | b       |
| 3     | c       |
| 7     | y       |
+-------+---------+--+
```



### explode 和lateral view配合产生虚表

explode：把array或map炸开，行转为列

```sql
select explode(location) from test_message;

select name,explode(location) from test_message; 报错
当使用UDTF函数的时候,hive只允许对拆分字段进行访问的。
```

配合使用：

```sql
select table_name.列，tmp.c1 from table_name lateral view explode(table_name.任意列) tmp as c1
--as 后面跟的是虚表的列名
```

实际应用

```sql
select  *  from test_message lateral view explode(location) tmp as c1;

--查询出每个人连同他的城市名称

select  test_message.name ,tmp.c1 from test_message lateral view explode(location) tmp as c1;

+--------------------+----------+--+
| test_message.name  |  tmp.c1  |
+--------------------+----------+--+
| allen              | usa      |
| allen              | china    |
| allen              | japan    |
| kobe               | usa      |
| kobe               | england  |
| kobe               | japan    |
+--------------------+----------+--+
```



### 行列转换

#### 单列转多行

```sql
+-----------------+-----------------+-----------------+--+
| col2row_2.col1  | col2row_2.col2  | col2row_2.col3  |
+-----------------+-----------------+-----------------+--+
| a               | b               | ["1","2","3"]   |
| c               | d               | ["4","5","6"]   |
+-----------------+-----------------+-----------------+--+

+-------+-------+----------+--+
| col1  | col2  | tmp.col  |
+-------+-------+----------+--+
| a     | b     | 1        |
| a     | b     | 2        |
| a     | b     | 3        |
| c     | d     | 4        |
| c     | d     | 5        |
| c     | d     | 6        |
+-------+-------+----------+--+

select explode(split(col3,",")) from col2row_21;
--如果explode函数接受的参数不是array 可以尝试使用其他函数比如split进行切割 返回的结果array
--str_to_map  切割返回map
```

#### 多列转单行

```sql
collect_set   去重收集的元素 
collect_list   非去重收集

concat_ws(参数1，参数2)
参数1：指定拼接数据的分隔符
参数2：指定待拼接的字符串  要求类型是String字符串


select concat_ws("-",collect_set(cast(col3 as string))) from row2col_1 group by col1,col2;
```



### reflect

可以调用java里面函数

```sql
reflect(class, method[, arg1[, arg2..]])
         类名    方法名   参数...
```



#### json字符使用

1.get_json_object

2.json_tuple   ---这个产生虚表

```sql
--get_json_object 普通函数 输入一行输出一行
--缺点：如果需要解析多个  需要写多次该函数
select get_json_object(t.json,'$.id'), get_json_object(t.json,'$.total_number')

--json_tuple   UDTF 表生成函数 
--通常配合lateral view使用
select t2.* from tmp_json_test t1 lateral view json_tuple(t1.json, 'id', 'total_number') t2 as c1, c2;
```



3.使用第三方serDe解析，只要再建表hi后导入即可

```sql
ROW FORMAT SERDE 'org.openx.data.jsonserde.JsonSerDe'
```



4.复杂类型解析

```sql
select json_tuple(json, 'website', 'name') from (SELECT explode(split(regexp_replace(regexp_replace('[{"website":"www.itcast.cn","name":"allenwoon"},{"website":"cloud.itcast.com","name":"carbondata 中文文档"}]', '\\}\\,\\{','\\}\\;\\{'),'\\[|\\]',''),'\\;')) as json) itcast;
```





### 窗口函数

over(partition by xxx  order by xxx);

可以配合聚合函数使用

```sql
--默认从第一行聚合当前行
select cookieid,createtime,pv,
sum(pv) over(partition by cookieid order by createtime) as pv1 
from itcast_t1;

--还可以配合windows子句 控制行的范围 从哪里到哪里
rows between含义,也叫做window子句：
- preceding：往前
- following：往后
- current row：当前行
- unbounded：边界
- unbounded preceding 表示从前面的起点
- unbounded following：表示到后面的终点


/*
- 如果不指定rows between,默认为从起点到当前行;
- 如果不指定order by，则将分组内所有值累加;
- 关键是理解rows between含义,也叫做window子句：
  - preceding：往前
  - following：往后
  - current row：当前行
  - unbounded：起点
  - unbounded preceding 表示从前面的起点
  - unbounded following：表示到后面的终点
*/
```

ROW_NUMBER()   :正常排序

RANK()：                   出现并列后下一位空出顺延

DENSE_RANK()：     出现并列后下一位不空出

ntile(NUM)可以看成是：把有序的数据集合平均分配到指定的数量（num）个桶中, 将桶号分配给每一行。如果不能平均分配，则优先分配较小编号的桶，并且各个桶中能放的行数最多相差1。

### 数据压缩

map段

```sql
1）开启 hive 中间传输数据压缩功能
set hive.exec.compress.intermediate=true;
2）开启 mapreduce 中 map 输出压缩功能
set mapreduce.map.output.compress=true;
3）设置 mapreduce 中 map 输出数据的压缩方式
set mapreduce.map.output.compress.codec=
org.apache.hadoop.io.compress.SnappyCodec;
```

reduce段

```sql
1）开启 hive 最终输出数据压缩功能
set hive.exec.compress.output=true;
2）开启 mapreduce 最终输出数据压缩
set mapreduce.output.fileoutputformat.compress=true;
3）设置 mapreduce 最终数据输出压缩方式
set mapreduce.output.fileoutputformat.compress.codec =
org.apache.hadoop.io.compress.SnappyCodec;
4）设置 mapreduce 最终数据输出压缩为块压缩
set mapreduce.output.fileoutputformat.compress.type=BLOCK;
```

使用insert + select语法把数据从student表中插入到stu_snappy中

```sql
insert into table stu_snappy select * from student where sage >18;
```

### hive文件存储格式

默认：textfile

格式：

```sql
stored as 文件格式

支持格式：textFile、ORC（列式存储）、PARQUET（列式存储）、SequenceFile、AVRO
默认是textFile，推荐orc---是行列混合存储
```

行列存储：

```sql
行式存储
a1b1c1a2b2c2a3b3c3  适合插入更新 不适合查询

列式存储
a1a2a3b1b2b3c1c2c3  适合查询 不适合插入更新
```
