# Hive 3.1.3基础

## 1. DDL语句

### 1.1 数据库操作

**创建数据库**

```sql
CREATE DATABASE [IF NOT EXISTS] database_name
[COMMENT database_comment]
[LOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)]; -- dbproperties可以自定义创建人、创建时间等属性
```



**展示所有数据库**

```sql
SHOW DATABASES [LIKE 'identifier_with_wildcards'];
```

like通配表达式说明：*表示任意个任意字符，|表示或的关系。



**查看数据库信息**

```sql
DESCRIBE DATABASE [EXTENDED] db_name;
```



**修改数据库**

```sql
--修改dbproperties
ALTER DATABASE database_name SET DBPROPERTIES (property_name=property_value, ...);

--修改location
ALTER DATABASE database_name SET LOCATION hdfs_path;

--修改owner user
ALTER DATABASE database_name SET OWNER USER user_name;
```



**删除数据库**

```sql
drop database db_hive3 cascade; -- cascade在数据库内有表的情况下强制删除
```



### 1.2 表操作

**创建表**

```sql
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name   
[(col_name data_type [COMMENT col_comment], ...)]
[COMMENT table_comment]
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)]
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS]
[ROW FORMAT row_format] 
[STORED AS file_format]
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
```



**关键字说明**

1. **TEMPORARY**

​			临时表，该表只在当前会话可见，会话结束，表会被删除

2. **EXTERNAL**

​			外部表，与之相对应的是内部表（管理表）。管理表意味着Hive会完全接管该表，包括元数据和HDFS中的数据。而外部表则意味着Hive只接管元数据，而不完全接管HDFS中的数据。

​			当删除外部表时，对应的HDFS中数据不会删除。

3. **data_type**

   ​	Hive的基本数据类型可以做类型转换，转换的方式包括隐式转换以及显示转换。

   ​	隐式转换参考：[LanguageManual Types - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/hive/languagemanual+types#LanguageManualTypes-AllowedImplicitConversions)

​		   显示转换使用`cast`即可

4. **PARTITIONED BY**

​		   创建分区表

5. **CLUSTERED BY ... SORTED BY...INTO ... BUCKETS**

   ​	创建分桶表

6. **ROW FORMAT**

   ​	指定SERDE，SERDE是Serializer and Deserializer的简写。Hive使用SERDE序列化和反序列化每行数据。详情可参考 [Hive-Serde](https://cwiki.apache.org/confluence/display/Hive/DeveloperGuide#DeveloperGuide-HiveSerDe)。语法说明如下：

   - 自定义分隔符

     ```hive
     ROW FORAMT DELIMITED 
     [FIELDS TERMINATED BY char] 
     [COLLECTION ITEMS TERMINATED BY char] 
     [MAP KEYS TERMINATED BY char] 
     [LINES TERMINATED BY char] 
     [NULL DEFINED AS char]
     
     -- fields terminated by ：列分隔符
     -- collection items terminated by ： map、struct和array中每个元素之间的分隔符
     -- map keys terminated by ：map中的key与value的分隔符
     -- lines terminated by ：行分隔符
     ```

   - SERDE关键字可用于指定其他内置的SERDE或者用户自定义的SERDE。例如JSON SERDE，可用于处理JSON字符串

     ```hive
     ROW FORMAT SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value,property_name=property_value, ...)] 
     ```

7. **STORED AS**

​		指定文件格式，常用的文件格式有，textfile（默认值），sequence file，orc file、parquet file等等。

8. **LOCATION**

   指定表所对应的HDFS路径，若不指定路径，其默认值为`${hive.metastore.warehouse.dir}/db_name.db/table_name`

9. **TBLPROPERTIES**

   用于配置表的一些KV键值对参数



**创建表**

Create Table As Select（CTAS）建表

```hive
CREATE [TEMPORARY] TABLE [IF NOT EXISTS] table_name 
[COMMENT table_comment] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
[AS select_statement]
```

该语法允许用户利用select查询语句返回的结果，直接建表，表的结构和查询语句的结构保持一致，且保证包含select查询语句放回的内容。



Create Table Like语法

```hive
CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS] [db_name.]table_name
[LIKE exist_table_name]
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
[TBLPROPERTIES (property_name=property_value, ...)]
```



**查看表**

展示所有表

```hive
SHOW TABLES [IN database_name] LIKE ['identifier_with_wildcards'];
-- show tables like 'stu*';
```

like通配表达式说明：*表示任意个任意字符，|表示或的关系。



查看表信息

```hive
DESCRIBE [EXTENDED | FORMATTED] [db_name.]table_name
```

​			EXTENDED：展示详细信息

​			FORMATTED：对详细信息进行格式化的展示



**修改表**

1. **重命名表**

   ```hive
   ALTER TABLE table_name RENAME TO new_table_name
   ```

2. **增加列**

   ```hive
   ALTER TABLE table_name ADD COLUMNS (col_name data_type [COMMENT col_comment], ...) -- 只能新增于末尾
   ```

3. **更新列**

   ```hive
   ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENT col_comment] [FIRST|AFTER column_name]
   ```

4. **替换列**

   ```hive
   ALTER TABLE table_name REPLACE COLUMNS (col_name data_type [COMMENT col_comment], ...)
   -- 该语句允许用户用新的列集替换表中原有的全部列
   ```



**删除表**

```hive
DROP TABLE [IF EXISTS] table_name;
```



**清空表**

```hive
TRUNCATE [TABLE] table_name
```



## 2. DML语句

### 2.1 load

```hive
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)];
```



### 2.2 Insert

**Select插入**

```hive
INSERT (INTO | OVERWRITE) TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement;
```



**Insert Value**

```hive
INSERT (INTO | OVERWRITE) TABLE tablename [PARTITION (partcol1[=val1], partcol2[=val2] ...)] 
VALUES values_row [, values_row ...]
```



**查询结果写入目标路径**

```hive
INSERT OVERWRITE [LOCAL] DIRECTORY directory
  [ROW FORMAT row_format] [STORED AS file_format] select_statement;
 
-- insert overwrite local directory '/opt/module/datas/student' ROW FORMAT SERDE 
-- 'org.apache.hadoop.hive.serde2.JsonSerDe'
-- select id,name from student;
```



### 2.3 Export&Import

Export导出语句可将表的数据和元数据信息一并到处的HDFS路径，Import可将Export导出的内容导入Hive，表的数据和元数据信息都会恢复。

Export和Import可用于两个Hive实例之间的数据迁移。

```hive
--导出
EXPORT TABLE tablename TO 'export_target_path'

--导入
IMPORT [EXTERNAL] TABLE new_or_original_tablename FROM 'source_path' [LOCATION 'import_target_path']
```

**案例**

```hive
--导出
hive>
export table default.student to '/user/hive/warehouse/export/student';

--导入
hive>
import table student2 from '/user/hive/warehouse/export/student';
```



## 3. 查询

```hive
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
  FROM table_reference       -- 从什么表查
  [WHERE where_condition]   -- 过滤
  [GROUP BY col_list]        -- 分组查询
   [HAVING col_list]          -- 分组后过滤
  [ORDER BY col_list]        -- 排序
  [CLUSTER BY col_list
    | [DISTRIBUTE BY col_list] [SORT BY col_list]
```



### 3.1 Limit语句

```hive
hive (default)> select * from emp limit 5; 
hive (default)> select * from emp limit 2,3; -- 表示从第2行开始，向下抓取3行
```



### 3.2 关系运算函数

**如下操作符主要用于where和having语句中**

| **操作符**               | **支持的数据类型** | **描述**                                                     |
| ------------------------ | ------------------ | ------------------------------------------------------------ |
| A=B                      | 基本数据类型       | 如果A等于B则返回true，反之返回false                          |
| A<=>B                    | 基本数据类型       | 如果A和B都为null或者都不为null，则返回true，如果只有一边为null，返回false |
| A<>B, A!=B               | 基本数据类型       | A或者B为null则返回null；如果A不等于B，则返回true，反之返回false |
| A<B                      | 基本数据类型       | A或者B为null，则返回null；如果A小于B，则返回true，反之返回false |
| A<=B                     | 基本数据类型       | A或者B为null，则返回null；如果A小于等于B，则返回true，反之返回false |
| A>B                      | 基本数据类型       | A或者B为null，则返回null；如果A大于B，则返回true，反之返回false |
| A>=B                     | 基本数据类型       | A或者B为null，则返回null；如果A大于等于B，则返回true，反之返回false |
| A [not] between B and  C | 基本数据类型       | 如果A，B或者C任一为null，则结果为null。如果A的值大于等于B而且小于或等于C，则结果为true，反之为false。如果使用not关键字则可达到相反的效果。 |
| A is null                | 所有数据类型       | 如果A等于null，则返回true，反之返回false                     |
| A is not null            | 所有数据类型       | 如果A不等于null，则返回true，反之返回false                   |
| in（数值1，数值2）       | 所有数据类型       | 使用 in运算显示列表中的值                                    |
| A [not] like B           | string 类型        | B是一个SQL下的简单正则表达式，也叫通配符模式，如果A与其匹配的话，则返回true；反之返回false。B的表达式说明如下：‘x%’表示A必须以字母‘x’开头，‘%x’表示A必须以字母‘x’结尾，而‘%x%’表示A包含有字母‘x’,可以位于开头，结尾或者字符串中间。如果使用not关键字则可达到相反的效果。 |
| A rlike B, A regexp  B   | string 类型        | B是基于java的正则表达式，如果A与其匹配，则返回true；反之返回false。匹配使用的是JDK中的正则表达式接口实现的，因为正则也依据其中的规则。例如，正则表达式必须和整个字符串A相匹配，而不是只需与其字符串匹配。 |



### 3.3 逻辑运算函数

| **操作符** | **含义** |
| ---------- | -------- |
| and        | 逻辑并   |
| or         | 逻辑或   |
| not        | 逻辑否   |



### 3.4 聚合函数

count(*)，表示统计所有行数，包含null值；

count(某列)，表示该列一共有多少行，不包含null值；

max()，求最大值，不包含null，除非所有值都是null；

min()，求最小值，不包含null，除非所有值都是null；

sum()，求和，不包含null。

avg()，求平均值，不包含null。



### 3.5 多表连接

```hive
select 
    e.ename, 
    d.dname, 
    l.loc_name
from emp e 
join dept d
on d.deptno = e.deptno 
join location l
on d.loc = l.loc;
```

大多数情况下，Hive会对每对join连接对象启动一个MapReduce任务。本例中会首先启动一个MapReduce job对表e和表d进行连接操作，然后会再启动一个MapReduce job将第一个MapReduce job的输出和表l进行连接操作。



### 3.6 union & union all

union和union all都是上下拼接sql的结果，这点是和join有区别的，join是左右关联，union和union all是上下拼接。**union去重，union all不去重**。

union和union all在上下拼接sql结果时有两个要求：

- 两个sql的结果，列的个数必须相同
- 两个sql的结果，上下所对应列的类型必须一致



### 3.7 Order By

全局排序，只有一个Reduce。

- asc（ascend）：升序（默认）
- desc（descend）：降序



### 3.8 Sort By

Sort By：对于大规模的数据集order by的效率非常低。在很多情况下，并不需要全局排序，此时可以使用**Sort by**。

Sort by为每个reduce产生一个排序文件。每个Reduce内部进行排序，对全局结果集来说不是排序。



### 3.9 Distribute By

Distribute By：在有些情况下，我们需要控制某个特定行应该到哪个Reducer，通常是为了进行后续的聚集操作。**distribute by**子句可以做这件事。

**distribute by**类似MapReduce中partition（自定义分区），进行分区，结合sort by使用。 

对于distribute by进行测试，一定要分配多reduce进行处理，否则无法看到distribute by的效果。



### 3.10 Cluster By

当distribute by和sort by字段相同时，可以使用cluster by方式。

cluster by除了具有distribute by的功能外还兼具sort by的功能。但是排序只能是升序排序，不能指定排序规则为asc或者desc。



## 4. 函数

### 4.1 算术运算函数

| **运算符** | **描述**       |
| ---------- | -------------- |
| A+B        | A和B 相加      |
| A-B        | A减去B         |
| A*B        | A和B 相乘      |
| A/B        | A除以B         |
| A%B        | A对B取余       |
| A&B        | A和B按位取与   |
| A\|B       | A和B按位取或   |
| A^B        | A和B按位取异或 |
| ~A         | A按位取反      |



### 4.2 UDTF函数

接收一行数据，产生一行或多行数据

```hive
select
    cate,
    count(*)
from
(
    select
        movie,
        cate
    from
    (
        select
            movie,
            split(category,',') cates
        from movie_info
    )t1 lateral view explode(cates) tmp as cate
)t2
group by cate;

```

explode是表生成函数的一种（UDTF）



**常用UDTF函数**

- explode：按照分隔符将数组或Map切分成多行
- posexplode：按照分隔符将数组或Map切分成多行，且附带序号
- inline：将结构体数组提取出来并插入到表中
- stack：把M列转换成N行，每行有M/N个字段，其中n必须是个常数



**lateral view**

lateral view可以将源表中每行的UDTF函数输出结果，与源表中的该行连接起来，形成一个虚拟表

```hive
表1 lateral view UDTF(col_name) 虚拟表名 as (列名1, 列名2...)
```



### 4.3 窗口函数

1. 聚合函数

   - max：最大值

   - min：最小值

   - sum：求和

   - avg：平均值

   - count：计数

   ```
   如果聚合函数的语句中带有order by，则默认输出结果是第一行到当前行。
   如果不带有order by，则默认输出结果是整个窗口的结果
   ```

2. 跨行取值函数

   - lead：后一行值

   - lag：前一行值

3. first_value和last_value

4. 排名函数

   - rank 
   - dense_rank
   - row_number

   

### 4.4 自定义UDF函数

**导入依赖**

```xml
<dependencies>
	<dependency>
		<groupId>org.apache.hive</groupId>
		<artifactId>hive-exec</artifactId>
		<version>3.1.3</version>
	</dependency>
</dependencies>
```



**创建一个类**

```java
import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentLengthException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentTypeException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

/**
 * 我们需计算一个要给定基本数据类型的长度
 */
public class MyUDF extends GenericUDF {
    /**
     * 判断传进来的参数的类型和长度
     * 约定返回的数据类型
     */
    @Override
    public ObjectInspector initialize(ObjectInspector[] arguments) throws UDFArgumentException {

        if (arguments.length !=1) {
            throw  new UDFArgumentLengthException("please give me  only one arg");
        }

        if (!arguments[0].getCategory().equals(ObjectInspector.Category.PRIMITIVE)){
            throw  new UDFArgumentTypeException(1, "i need primitive type arg");
        }

        return PrimitiveObjectInspectorFactory.javaIntObjectInspector;
    }

    /**
     * 解决具体逻辑的
     */
    @Override
    public Object evaluate(DeferredObject[] arguments) throws HiveException {

        Object o = arguments[0].get();
        if(o==null){
            return 0;
        }

        return o.toString().length();
    }

    @Override
    // 用于获取解释的字符串
    public String getDisplayString(String[] children) {
        return "";
    }
}
```



**创建临时函数**

```hive
hive (default)> add jar /opt/module/hive/datas/myudf.jar;

hive (default)> create temporary function my_len as "com.atguigu.hive.udf.MyUDF";

hive (default)> 
select 
    ename,
    my_len(ename) ename_len 
from emp;
```

注意：临时函数只跟会话有关系，跟库没有关系。只要创建临时函数的会话不断，在当前会话下，任意一个库都可以使用，其他会话全都不能使用。



**创建永久函数**

注意：因为add jar本身也是临时生效，所以在创建永久函数的时候，需要制定路径（并且因为元数据的原因，这个路径还得是HDFS上的路径）。

```hive
hive (default)> 
create function my_len2 
as "com.hive.udf.MyUDF" 
using jar "hdfs://hadoop102:8020/udf/myudf.jar";


hive (default)> 
select 
    ename,
    my_len2(ename) ename_len 
from emp;

hive (default)> drop function my_len2;
```

注意：永久函数跟会话没有关系，创建函数的会话断了以后，其他会话也可以使用。

永久函数创建的时候，在函数名之前需要自己加上库名，如果不指定库名的话，会默认把当前库的库名给加上。

永久函数使用的时候，需要在指定的库里面操作，或者在其他库里面使用的话加上，**库名.函数名**。



## 5.分区&分桶

### 5.1 分区表

Hive中的分区就是把一张大表的数据按照业务需要分散的存储到多个目录，每个目录就称为该表的一个分区。在查询时通过where子句中的表达式选择查询所需要的分区，这样的查询效率会提高很多。



### 5.2 修复分区

Hive将分区表的所有分区信息都保存在了元数据中，只有元数据与HDFS上的分区路径一致时，分区表才能正常读写数据。若用户手动创建/删除分区路径，Hive都是感知不到的，这样就会导致Hive的元数据和HDFS的分区路径不一致。再比如，若分区表为外部表，用户执行drop partition命令后，分区元数据会被删除，而HDFS的分区路径不会被删除，同样会导致Hive的元数据和HDFS的分区路径不一致。

若出现元数据和HDFS路径不一致的情况，可通过如下几种手段进行修复。

**add partition**

若手动创建HDFS的分区路径，Hive无法识别，可通过add partition命令增加分区元数据信息，从而使元数据和分区路径保持一致。



**drop partition**

若手动删除HDFS的分区路径，Hive无法识别，可通过drop partition命令删除分区元数据信息，从而使元数据和分区路径保持一致。



**msck**

若分区元数据和HDFS的分区路径不一致，还可使用msck命令进行修复，以下是该命令的用法说明。

```hive
hive (default)> 
msck repair table table_name [add/drop/sync partitions];
```

说明：

`msck repair table table_name add partitions`：该命令会增加HDFS路径存在但元数据缺失的分区信息。

`msck repair table table_name drop partitions`：该命令会删除HDFS路径已经删除但元数据仍然存在的分区信息。

`msck repair table table_name sync partitions`：该命令会同步HDFS路径和元数据分区信息，相当于同时执行上述的两个命令。

`msck repair table table_name`：等价于msck repair table table_name **add** partitions命令。





### 5.3 动态分区

动态分区是指向分区表insert数据时，被写往的分区不由用户指定，而是由每行数据的最后一个字段的值来动态的决定。使用动态分区，可只用一个insert语句将数据写入多个分区。

**需要开启相关参数**

```hive
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.exec.max.dynamic.partitions=1000;
set hive.exec.max.dynamic.partitions.pernode=100;
hive.exec.max.created.files=100000;
hive.error.on.empty.partition=false;
```



### 5.4 分桶表

```hive
hive (default)> 
create table stu_buck(
    id int, 
    name string
)
clustered by(id) [sorted by(id)] -- 加sorted by就是分桶排序表
into 4 buckets
row format delimited fields terminated by '\t';
```



导入数据到分桶表中

```hive
hive (default)> 
load data local inpath '/opt/module/hive/datas/student.txt' 
into table stu_buck_sort;
```



## 6. 文件格式和压缩

### 6.1 Hadoop压缩概述

| **压缩格式** | **算法** | **文件扩展名** | **是否可切分** |
| ------------ | -------- | -------------- | -------------- |
| DEFLATE      | DEFLATE  | .deflate       | 否             |
| Gzip         | DEFLATE  | .gz            | 否             |
| bzip2        | bzip2    | .bz2           | **是**         |
| LZO          | LZO      | .lzo           | **是**         |
| Snappy       | Snappy   | .snappy        | 否             |

为了支持多种压缩/解压缩算法，Hadoop引入了编码/解码器，如下表所示：

Hadoop查看支持压缩的方式hadoop checknative。

Hadoop在driver端设置压缩。

| **压缩格式** | **对应的编码/解码器**                      |
| ------------ | ------------------------------------------ |
| DEFLATE      | org.apache.hadoop.io.compress.DefaultCodec |
| gzip         | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2        | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO          | com.hadoop.compression.lzo.LzopCodec       |
| Snappy       | org.apache.hadoop.io.compress.SnappyCodec  |

压缩性能的比较：

| **压缩算法** | **原始文件大小** | **压缩文件大小** | **压缩速度** | **解压速度** |
| ------------ | ---------------- | ---------------- | ------------ | ------------ |
| gzip         | 8.3GB            | 1.8GB            | 17.5MB/s     | 58MB/s       |
| bzip2        | 8.3GB            | 1.1GB            | 2.4MB/s      | 9.5MB/s      |
| LZO          | 8.3GB            | 2.9GB            | 49.3MB/s     | 74.6MB/s     |



### 6.2 ORC

ORC（Optimized Row Columnar）file format是Hive 0.11版里引入的一种**列式存储**的文件格式。ORC文件能够提高Hive读写数据和处理数据的性能。

**列存储的特点**：因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。

![image-20230123171254338](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/HiveOrc%E6%96%87%E4%BB%B6%E7%BB%84%E6%88%90.png)

每个Orc文件由Header、Body和Tail三部分组成。

其中Header内容为ORC，用于表示文件类型。

Body由1个或多个stripe组成，每个stripe一般为HDFS的块大小，每一个stripe包含多条记录，这些记录按照列进行独立存储，每个stripe里有三部分组成，分别是Index Data，Row Data，Stripe Footer。

**Index Data**：一个轻量级的index，默认是为各列每隔1W行做一个索引。每个索引会记录第n万行的位置，和最近一万行的最大值和最小值等信息。

**Row Data**：存的是具体的数据，按列进行存储，并对每个列进行编码，分成多个Stream来存储。

**Stripe Footer**：存放的是各个Stream的位置以及各column的编码信息。

**Tail**：由File Footer和PostScript组成。File Footer中保存了各Stripe的其实位置、索引长度、数据长度等信息，各Column的统计信息等；PostScript记录了整个文件的压缩类型以及File Footer的长度信息等。

在读取ORC文件时，会先从最后一个字节读取PostScript长度，进而读取到PostScript，从里面解析到File Footer长度，进而读取FileFooter，从中解析到各个Stripe信息，再读各个Stripe，即从后往前读。

**Orc参数如下**：

| **参数**             | **默认值** | **说明**                                |
| -------------------- | ---------- | --------------------------------------- |
| orc.compress         | ZLIB       | 压缩格式，可选项：NONE、ZLIB,、SNAPPY   |
| orc.compress.size    | 262,144    | 每个压缩块的大小（ORC文件是分块压缩的） |
| orc.stripe.size      | 67,108,864 | 每个stripe的大小                        |
| orc.row.index.stride | 10,000     | 索引步长（每隔多少行数据建一条索引）    |

**为什么Orc + Snappy能分块**

因为Orc压缩是分块压缩，参见` orc.compress.size  `，Snappy会对每个块做压缩



### 6.3 Parquet

Parquet文件是Hadoop生态中的一个通用的文件格式，它也是一个列式存储的文件格式

![image-20230123172003829](https://raw.githubusercontent.com/flickever/NotePictures/master/Note/HiveParquet%E6%96%87%E4%BB%B6%E7%BB%84%E6%88%90.png)

首尾中间由若干个Row Group和一个Footer（File Meta Data）组成。

每个Row Group包含多个Column Chunk，每个Column Chunk包含多个Page。以下是Row Group、Column Chunk和Page三个概念的说明：

**行组（Row Group）**：一个行组对应逻辑表中的若干行。 

**列块（Column Chunk）**：一个行组中的一列保存在一个列块中。 

**页（Page）**：一个列块的数据会划分为若干个页。 

**Footer（File Meta Data）**中存储了每个行组（Row Group）中的每个列快（Column Chunk）的元数据信息，元数据信息包含了该列的数据类型、该列的编码方式、该类的Data Page位置等信息。