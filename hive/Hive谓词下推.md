# Hive谓词下推

文章来源：[上篇](https://zhuanlan.zhihu.com/p/78266517)、[下篇](https://zhuanlan.zhihu.com/p/78783181)

### **几个概念**

- Preserved Row table（保留表）

  在`outer join`中需要返回所有数据的表叫做保留表，也就是说在`left outer join`中，左表需要返回所有数据，则左表是保留表；`right outer join中`右表则是保留表；在`full outer join`中左表和右表都要返回所有数据，则左右表都是保留表。

- Null Supplying table（空表）

  在`outer join`中对于没有匹配到的行需要用`null`来填充的表称为`Null Supplying table`。在`left outer join`中，左表的数据全返回，对于左表在右表中无法匹配的数据的相应列用`null`表示，则此时右表是`Null Supplying table`，相应的如果是`right outer join`的话，左表是`Null Supplying table`。但是在`full outer join`中左表和右表都是`Null Supplying table`，因为左表和右表都会用`null`来填充无法匹配的数据。

- During Join predicate（Join中的谓词）

  `Join`中的谓词是指 `Join On`语句中的谓词。如：`R1 join R2 on R1.x = 5  the predicate R1.x = 5`是`Join`中的谓词

- After Join predicate（Join之后的谓词）

  `where`语句中的谓词称之为`Join`之后的谓词



### **谓词下推**

谓词下推的基本思想：将过滤表达式尽可能移动至靠近数据源的位置，以使真正执行时能直接跳过无关的数据。

在hive官网上给出了`outer join`【`left outer join、right out join、full outer join`】的谓下推规则：

1. `Join`(只包括`left join ,right join,full join`)中的谓词如果是保留表的，则不会下推 
2. `Join`(只包括`left join ,right join,full join`)之后的谓词如果是`Null Supplying tables`的，则不会下推



这种规则在hive2.x版本以后，就不是很准确了，hive2.x对CBO做了优化，CBO也对谓词下推规则产生了一些影响。

因此在hive2.1.1中影响谓词下推规则的，主要有两方面

- Hive逻辑执行计划层面的优化
- CBO



### **case1 inner join 中的谓词**

sql如下

```sql
select t1.*,t2.* from test1 t1 join test2 t2 on t1.id=t2.id and t1.openid='pear' and t2.openid='apple';
```

这种情况，不管CBO优化有没有开启，hive都会做谓词下推；在`on`中的两个条件都提至`table scan`阶段进行过滤



### **case2 inner join 之后的谓词**

sql如下

```sql
select t1.*,t2.* from test1 t1 join test2 t2 on t1.id=t2.id where t1.openid='pear'  and t2.openid='apple';
```

不管CBO优化有没有开启，结果不变

`inner join`之后的谓词 与 `inner join`之中的谓词的效果是完全一样的，这点很容易理解



### **case3 left join 中的谓词**

**关闭CBO**

```sql
set hive.cbo.enable=false;

explain select t1.*,t2.* from test1 t1 left join test2 t2 on t1.id=t2.id and t1.openid='pear'  and t2.openid='apple';
```

在`hive.cbo.enable=false`的情况下，test2表进行谓词下推，`openid = 'apple'`条件放在`table scan`之后，而test1为保留表，没有下推

查询的过程：遍历test1所有的数据，提取test2中`openid`为`apple`的数据

然后，拿test1中 `openid`为`pear`的数据与test2表中的这条关联，关联上的就展示，关联不上的，补`null`；另外，test1中的`openid`不为`pear`的数据不与test2表做关联，直接补`null`。。。



**开启CBO**

由于`join`中谓词执行过程的特殊性，CBO也不可能再做什么优化，因此，打开CBO开关，执行计划不变：





### **case4 left join 之后的谓词**

**关闭CBO**

```sql
set hive.cbo.enable=false;

explain select t1.*,t2.* from test1 t1 left join test2 t2 on t1.id=t2.id where t1.openid='pear'  and t2.openid='apple';
```

屏蔽掉CBO对谓词下推的影响后，test1进行了下推，将`predicate: (openid = 'pear') (type: boolean)`过滤条件下推到 `tablescan`处进行；

而test2没有下推，最后是在 `join`之后，才进行了 `predicate: (_col7 = 'apple') (type: boolean)`过滤。这个也符合【 `Join`之后的谓词如果是 `Null Supplying tables`的，则不会下推】。



**开启CBO**

`join`之后的 `where`无论怎样都是要过滤一遍的，这里的test2实际上是可以下推的，下推了会更优化，也不会改变最终的结果

打CBO开关 `set hive.cbo.enable=true` 后，我们会发现，test2也进行了谓词下推，做到了更优化



### **case5 full join 中的谓词**

**关闭CBO**

```sql
set hive.cbo.enable=false;

explain select t1.*,t2.* from test1 t1 full join test2 t2 on t1.id=t2.id and t1.openid='pear'  and t2.openid='apple';
```

`full join` 的左右表很矛盾，即是保留表，也是 `Null Supplying tables`。 但是有一条不变，就是左右表的数据都一定是要保留表下来的，因此也不难理解，这里为什么不能进行谓词下推，只要下推了，就不能保证两个表的数据都保留。。。

所以，数据过滤是在join'操作后进行的

这也算是符合了【 `Join`(只包括 `left join ,right join,full join`)中的谓词如果是保留表的，则不会下推】



过程如下：

test1, test2 两个表全遍历，test1表 拿 `openid`为 `pear`的与test2表中的数据关联，其它 `openid`不为 `pear`的 ，test2表直接补 `null`。 而test2 表拿 `openid`为 `apple`的与test1表中的数据关联，其它 `openid`不为 `apple`的，test1 表直接补 `null`。



因此打开CBO开关，也不会做任何优化



### **case6 full join 之后的谓词**

**关掉CBO**

```sql
set hive.cbo.enable=false;

explain select t1.*,t2.* from test1 t1 full join test2 t2 on t1.id=t2.id where t1.openid='pear'  and t2.openid='apple';
```

关掉CBO开关，我们会发现，没有进行谓词下推的优化，而是在 `full join`之后，做了 `predicate: ((_col1 = 'pear') and (_col7 = 'apple')) (type: boolean)`的过滤，这也符合【 `Join`(只包括 `left join ,right join,full join`)之后的谓词如果是 `Null Supplying tables`的，则不会下推】这一规则，毕竟test1,test2也属于 `Null Supplying tables` 。

实际上，这里是可以下推的，与 `left join`同理，下推后，不影响最后的结果：



**开启CBO**

```sql
set hive.cbo.enable=true;
explain select t1.*,t2.* from test1 t1 full join test2 t2 on t1.id=t2.id where t1.openid='pear'  and t2.openid='apple';
```

打开CBO开关，两表都进行了下推