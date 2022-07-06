# 03_DWS层

### 1. DWS 层的定位

- 轻度聚合，因为 DWS 层要应对很多实时查询，如果是完全的明细那么查询的压力是非常大的
-  将更多的实时数据以主题的方式组合起来便于管理，同时也能减少维度查询的次数



### 2. 访客主体宽表

**场景：**

将数据按照维度分组聚合，形成DWS宽表

宽表的字段来自于不同的kafka Topic

**要求：**

不同流里面的kafka的数据，需要先聚合，汇总后的字段数据全都插入到同一个kafka Topic

**解决：**

- 接收各个明细数据，变为数据流
- 把数据流合并在一起，成为一个相同格式对象的数据流（union 算子）
- 对合并的流进行聚合，聚合的时间窗口决定了数据的时效性（滚动窗口）
- 把聚合结果写在数据库中（clickhouse）

**表内容**

- 度量包括 PV、UV、跳出次数、进入页面数(session_count)、连续访问时长
- 维度包括在分析中比较重要的几个字段：渠道、地区、版本、新老用户进行聚合

**实现真正增量聚合**

对`WindowedStream`调用`reduce`方法

```java
	/**
	 * Applies the given window function to each window. The window function is called for each
	 * evaluation of the window for each key individually. The output of the window function is
	 * interpreted as a regular non-windowed stream.
	 *
	 * <p>Arriving data is incrementally aggregated using the given reducer.
	 *
	 * @param reduceFunction The reduce function that is used for incremental aggregation.
	 * @param function The window function.
	 * @return The data stream that is the result of applying the window function to the window.
	 */
public <R> SingleOutputStreamOperator<R> reduce(
		ReduceFunction<T> reduceFunction,
        WindowFunction<T, R, K, W> function){...}
```

