# Hive源码分析（三）

本次分析Operator类和优化器Optimizers

### Operator类

地址：`org/apache/hadoop/hive/ql/exec/Operator.java`

```java
  /**
   * Process the row.
   *
   * @param row
   *          The object representing the row.
   * @param tag
   *          The tag of the row usually means which parent this row comes from.
   *          Rows with the same tag should have exactly the same rowInspector
   *          all the time.
   */
  public abstract void process(Object row, int tag) throws HiveException;
```

row是一行数据，tag代表是那张表。用一个整数代表是那张表。

hadoop在执行任务的时候会在每个节点创建一个进程。

- 每个进程一个实例
- 每个实例开始执行一次initialize()方法
- 每个实例执行多次process()方法，每行执行一次，这个进程有几行就执行几次
- 每个实例最后执行一次close()方法

对于Operator比较重要的有group by Operator和join Operator



#### **group by Operator**

前文章节SemanticAnalyzer生成一个QB，之后递归genplan()，然后是genBodyPlan()，genBodyPlan会对group by进行处理：

```java
@SuppressWarnings("nls")
  private Operator genBodyPlan(QB qb, Operator input, Map<String, Operator> aliasToOpInfo)
      throws SemanticException {
    QBParseInfo qbp = qb.getParseInfo();

    TreeSet<String> ks = new TreeSet<String>(qbp.getClauseNames());
    Map<String, Operator<? extends OperatorDesc>> inputs = createInputForDests(qb, input, ks);

    Operator curr = input;

    List<List<String>> commonGroupByDestGroups = null;

    // If we can put multiple group bys in a single reducer, determine suitable groups of
    // expressions, otherwise treat all the expressions as a single group
    if (conf.getBoolVar(HiveConf.ConfVars.HIVEMULTIGROUPBYSINGLEREDUCER)) {
      try {
        commonGroupByDestGroups = getCommonGroupByDestGroups(qb, inputs);
      } catch (SemanticException e) {
        LOG.error("Failed to group clauses by common spray keys.", e);
      }
    }

    if (commonGroupByDestGroups == null) {
      commonGroupByDestGroups = new ArrayList<List<String>>();
      commonGroupByDestGroups.add(new ArrayList<String>(ks));
    }

    if (!commonGroupByDestGroups.isEmpty()) {

      // Iterate over each group of subqueries with the same group by/distinct keys
      for (List<String> commonGroupByDestGroup : commonGroupByDestGroups) {
        if (commonGroupByDestGroup.isEmpty()) {
          continue;
        }

        String firstDest = commonGroupByDestGroup.get(0);
        input = inputs.get(firstDest);

        // Constructs a standard group by plan if:
        // There is no other subquery with the same group by/distinct keys or
        // (There are no aggregations in a representative query for the group and
        // There is no group by in that representative query) or
        // The data is skewed or
        // The conf variable used to control combining group bys into a single reducer is false
        if (commonGroupByDestGroup.size() == 1 ||
            (qbp.getAggregationExprsForClause(firstDest).size() == 0 &&
                getGroupByForClause(qbp, firstDest).size() == 0) ||
            conf.getBoolVar(HiveConf.ConfVars.HIVEGROUPBYSKEW) ||
            !conf.getBoolVar(HiveConf.ConfVars.HIVEMULTIGROUPBYSINGLEREDUCER)) {

          // Go over all the destination tables
          for (String dest : commonGroupByDestGroup) {
            curr = inputs.get(dest);

            if (qbp.getWhrForClause(dest) != null) {
              ASTNode whereExpr = qb.getParseInfo().getWhrForClause(dest);
              curr = genFilterPlan((ASTNode) whereExpr.getChild(0), qb, curr, aliasToOpInfo, false, false);
            }
            // Preserve operator before the GBY - we'll use it to resolve '*'
            Operator<?> gbySource = curr;

            if ((qbp.getAggregationExprsForClause(dest).size() != 0
                || getGroupByForClause(qbp, dest).size() > 0)
                && (qbp.getSelForClause(dest).getToken().getType() != HiveParser.TOK_SELECTDI
                || qbp.getWindowingExprsForClause(dest) == null)) {
              // multiple distincts is not supported with skew in data
              if (conf.getBoolVar(HiveConf.ConfVars.HIVEGROUPBYSKEW) &&
                  qbp.getDistinctFuncExprsForClause(dest).size() > 1) {
                throw new SemanticException(ErrorMsg.UNSUPPORTED_MULTIPLE_DISTINCTS.
                    getMsg());
              }
              // insert a select operator here used by the ColumnPruner to reduce
              // the data to shuffle
              curr = genSelectAllDesc(curr);
              // Check and transform group by *. This will only happen for select distinct *.
              // Here the "genSelectPlan" is being leveraged.
              // The main benefits are (1) remove virtual columns that should
              // not be included in the group by; (2) add the fully qualified column names to unParseTranslator
              // so that view is supported. The drawback is that an additional SEL op is added. If it is
              // not necessary, it will be removed by NonBlockingOpDeDupProc Optimizer because it will match
              // SEL%SEL% rule.
              ASTNode selExprList = qbp.getSelForClause(dest);
              if (selExprList.getToken().getType() == HiveParser.TOK_SELECTDI
                  && selExprList.getChildCount() == 1 && selExprList.getChild(0).getChildCount() == 1) {
                ASTNode node = (ASTNode) selExprList.getChild(0).getChild(0);
                if (node.getToken().getType() == HiveParser.TOK_ALLCOLREF) {
                  curr = genSelectPlan(dest, qb, curr, curr);
                  RowResolver rr = opParseCtx.get(curr).getRowResolver();
                  qbp.setSelExprForClause(dest, SemanticAnalyzer.genSelectDIAST(rr));
                }
              }
              // Hive Map端预聚合
              if (conf.getBoolVar(HiveConf.ConfVars.HIVEMAPSIDEAGGREGATE)) {
                if (!conf.getBoolVar(HiveConf.ConfVars.HIVEGROUPBYSKEW)) {
                  // 预聚合不倾斜
                  curr = genGroupByPlanMapAggrNoSkew(dest, qb, curr);
                } else {
                  // 预聚合倾斜两道作业
                  curr = genGroupByPlanMapAggr2MR(dest, qb, curr);
                }
              } else if (conf.getBoolVar(HiveConf.ConfVars.HIVEGROUPBYSKEW)) {
                // 不预聚合倾斜两道作业
                curr = genGroupByPlan2MR(dest, qb, curr);
              } else {
                // 不语句和不倾斜一道作业，最原始
                curr = genGroupByPlan1MR(dest, qb, curr);
              }
            }
            if (LOG.isDebugEnabled()) {
              LOG.debug("RR before GB " + opParseCtx.get(gbySource).getRowResolver()
                  + " after GB " + opParseCtx.get(curr).getRowResolver());
            }

            curr = genPostGroupByBodyPlan(curr, dest, qb, aliasToOpInfo, gbySource);
          }
        } else {
          // multi group by
          curr = genGroupByPlan1ReduceMultiGBY(commonGroupByDestGroup, qb, input, aliasToOpInfo);
        }
      }
    }


    if (LOG.isDebugEnabled()) {
      LOG.debug("Created Body Plan for Query Block " + qb.getId());
    }

    return curr;
  }
```

`HiveConf.ConfVars.HIVEMAPSIDEAGGREGATE`：在map阶段进行预聚合减少数据量`HiveConf.ConfVars.HIVEGROUPBYSKEW`：将一个group by拆成2个group by减少数据量

Hive Group By

- `HiveConf.ConfVars.HIVEMAPSIDEAGGREGATE`：hive.map.aggr，使用Map端预聚合
- `HiveConf.ConfVars.HIVEGROUPBYSKEW`：hive.groupby.skewindata，是否优化倾斜的查询为两道作业

Hive GroupBy hive.groupby.skewindata关闭的时候只有一道mr作业，当参数打开的时候，会进行预聚合，整个过程是2道mr作业。

这样我们就能完美了吗，我们的group by就不会倾斜了吗？大部分的group by是不会倾斜的，但是有一种是特殊的。

**代数型聚合与非代数聚合**

- 代数型聚合：可以通过部分结果计算出最终结果的聚合方法，如count、sum
- 非代数型聚合：无法通过部分结果计算出最终结果的聚合方法，如percentile，median

Group By优化只适用于代数型聚合，代数型UDAF，思考为什么？



**group by聚合**

group by的聚合逻辑就是这个process方法， process方法会调用 processHashAggr和 processAggr方法，即hash聚合和普通聚合的方法。

hash聚合：Map端预聚合

是否hash聚合在于input条目数是否大于map聚合设定的条目数

```java
 @Override
  public void process(Object row, int tag) throws HiveException {
    firstRow = false;
    ObjectInspector rowInspector = inputObjInspectors[0];
    // Total number of input rows is needed for hash aggregation only
    if (hashAggr) {
      numRowsInput++;
      // if hash aggregation is not behaving properly, disable it
      if (numRowsInput == numRowsCompareHashAggr) {
        numRowsCompareHashAggr += groupbyMapAggrInterval;
        // map-side aggregation should reduce the entries by at-least half
        if (numRowsHashTbl > numRowsInput * minReductionHashAggr) {
          LOG.warn("Disable Hash Aggr: #hash table = " + numRowsHashTbl
              + " #total = " + numRowsInput + " reduction = " + 1.0
              * (numRowsHashTbl / numRowsInput) + " minReduction = "
              + minReductionHashAggr);
          flushHashTable(true);
          hashAggr = false;
        } else {
          if (LOG.isTraceEnabled()) {
            LOG.trace("Hash Aggr Enabled: #hash table = " + numRowsHashTbl
                + " #total = " + numRowsInput + " reduction = " + 1.0
                * (numRowsHashTbl / numRowsInput) + " minReduction = "
                + minReductionHashAggr);
          }
        }
      }
    }

    try {
      countAfterReport++;
      newKeys.getNewKey(row, rowInspector);

      if (groupingSetsPresent) {
        Object[] newKeysArray = newKeys.getKeyArray();
        Object[] cloneNewKeysArray = new Object[newKeysArray.length];
        for (int keyPos = 0; keyPos < groupingSetsPosition; keyPos++) {
          cloneNewKeysArray[keyPos] = newKeysArray[keyPos];
        }

        for (int groupingSetPos = 0; groupingSetPos < groupingSets.size(); groupingSetPos++) {
          for (int keyPos = 0; keyPos < groupingSetsPosition; keyPos++) {
            newKeysArray[keyPos] = null;
          }

          FastBitSet bitset = groupingSetsBitSet[groupingSetPos];
          // Some keys need to be left to null corresponding to that grouping set.
          for (int keyPos = bitset.nextClearBit(0); keyPos < groupingSetsPosition;
                  keyPos = bitset.nextClearBit(keyPos+1)) {
            newKeysArray[keyPos] = cloneNewKeysArray[keyPos];
          }

          newKeysArray[groupingSetsPosition] = newKeysGroupingSets[groupingSetPos];
          processKey(row, rowInspector);
        }
      } else {
        processKey(row, rowInspector);
      }
    } catch (HiveException e) {
      throw e;
    } catch (Exception e) {
      throw new HiveException(e);
    }
  }
```



#### Join Operator

Join是Hive是最难的部分，也是最需要优化的部分

常用Join方法

- 普通（Reduce端）Join， Common (Reduce-Side) Join
- 广播（Map端）Join，Broadcast（Map-Side）Join
- Bucket Map Join
- Sort Merge Bucket Map Join
- 倾斜Join，Skew Join



**Common join**

从最简单的Common Join开始，此join是所有join的基础。

- 也叫做Reduce端Join
- 背景知识：Hive只支持等值Join，不支持非等值Join
- 扫描N张表，Join Key相同的放在一起（相同Reduce） -> 结果

流程:

- Mapper: 扫描，并处理N张表，生成发给Reduce的<Key, Value> Key = {JoinKey, TableAlias}, Value = {row}
- Shuffle阶段
  - JoinKey相同的Reduce放到相同的
  - TableAlias 是排序的标识，就是表的编号，相同表的数据在一起是排序的。
- Reducer: 处理Join Key并输出结果
- 最坏的情况
  - 所有的数据都被发送到相同的结点，同一个Reduce

```java
 @Override
  /*
   * row:当前行
   * tag:表编号，如a join b join c，a:0,b:1,c:2
   * shuffle保证数据排好序
   */
  public void process(Object row, int tag) throws HiveException {
    try {
      reportProgress();

      lastAlias = alias;
      alias = (byte) tag;

      if (!alias.equals(lastAlias)) {
        nextSz = joinEmitInterval;
      }

      List<Object> nr = getFilteredValue(alias, row);

      if (handleSkewJoin) {
        skewJoinKeyContext.handleSkew(tag);
      }

      // number of rows for the key in the given table
      long sz = storage[alias].rowCount();
      StructObjectInspector soi = (StructObjectInspector) inputObjInspectors[tag];
      StructField sf = soi.getStructFieldRef(Utilities.ReduceField.KEY
          .toString());
      List keyObject = (List) soi.getStructFieldData(row, sf);
      // Are we consuming too much memory
      // 如果读到了最后一张表，开始处理join
      if (alias == numAliases - 1 && !(handleSkewJoin && skewJoinKeyContext.currBigKeyTag >= 0) &&
          !hasLeftSemiJoin) {
        if (sz == joinEmitInterval && !hasFilter(condn[alias-1].getLeft()) &&
                !hasFilter(condn[alias-1].getRight())) {
          // The input is sorted by alias, so if we are already in the last join
          // operand,
          // we can emit some results now.
          // Note this has to be done before adding the current row to the
          // storage,
          // to preserve the correctness for outer joins.
          // 如果读到了最后一张表，开始处理join
          checkAndGenObject();
          storage[alias].clearRows();
        }
      } else {
        if (LOG.isInfoEnabled() && (sz == nextSz)) {
          // Print a message if we reached at least 1000 rows for a join operand
          // We won't print a message for the last join operand since the size
          // will never goes to joinEmitInterval.
          LOG.info("table " + alias + " has " + sz + " rows for join key " + keyObject);
          // 否则，把前面的表加入内存
          nextSz = getNextSize(nextSz);
        }
      }

      // Add the value to the vector
      // if join-key is null, process each row in different group.
      StructObjectInspector inspector =
          (StructObjectInspector) sf.getFieldObjectInspector();
      if (SerDeUtils.hasAnyNullObject(keyObject, inspector, nullsafes)) {
        endGroup();
        startGroup();
      }
      storage[alias].addRow(nr);
    } catch (Exception e) {
      e.printStackTrace();
      throw new HiveException(e);
    }
  }
```

可以看到左边的表放到内存（放不下才会放到磁盘），因此我们join的时候要把小表放到左边，提供性能。

用到的特殊的类`CommonJoinOperator`：

```java
private void genObject(int aliasNum, boolean allLeftFirst, boolean allLeftNull)
      throws HiveException {
    JoinCondDesc joinCond = condn[aliasNum - 1];
    int type = joinCond.getType();
    int left = joinCond.getLeft();
    int right = joinCond.getRight();

    if (needsPostEvaluation && aliasNum == numAliases - 2) {
      int nextType = condn[aliasNum].getType();
      if (nextType == JoinDesc.RIGHT_OUTER_JOIN || nextType == JoinDesc.FULL_OUTER_JOIN) {
        // Initialize container to use for storing tuples before emitting them
        rowContainerPostFilteredOuterJoin = new HashMap<>();
      }
    }

    boolean[] skip = skipVectors[aliasNum];
    boolean[] prevSkip = skipVectors[aliasNum - 1];

    // search for match in the rhs table
    // 内存中小表
    AbstractRowContainer<List<Object>> aliasRes = storage[order[aliasNum]];

    boolean needToProduceLeftRow = false;
    boolean producedRow = false;
    boolean done = false;
    boolean loopAgain = false;
    boolean tryLOForFO = type == JoinDesc.FULL_OUTER_JOIN;

    boolean rightFirst = true;
    AbstractRowContainer.RowIterator<List<Object>> iter = aliasRes.rowIter();
    int pos = 0;
    for (List<Object> rightObj = iter.first(); !done && rightObj != null;
         rightObj = loopAgain ? rightObj : iter.next(), rightFirst = loopAgain = false, pos++) {
      System.arraycopy(prevSkip, 0, skip, 0, prevSkip.length);

      boolean rightNull = rightObj == dummyObj[aliasNum];
      if (hasFilter(order[aliasNum])) {
        filterTags[aliasNum] = getFilterTag(rightObj);
      }
      skip[right] = rightNull;
	  // 判断是inner join还是别的join
      if (type == JoinDesc.INNER_JOIN) {
        innerJoin(skip, left, right);
      } else if (type == JoinDesc.LEFT_SEMI_JOIN) {
        if (innerJoin(skip, left, right)) {
          // if left-semi-join found a match and we do not have any additional predicates,
          // skipping the rest of the rows in the rhs table of the semijoin
          done = !needsPostEvaluation;
        }
      } else if (type == JoinDesc.LEFT_OUTER_JOIN ||
          (type == JoinDesc.FULL_OUTER_JOIN && rightNull)) {
        int result = leftOuterJoin(skip, left, right);
        if (result < 0) {
          continue;
        }
        done = result > 0;
      } else if (type == JoinDesc.RIGHT_OUTER_JOIN ||
          (type == JoinDesc.FULL_OUTER_JOIN && allLeftNull)) {
        if (allLeftFirst && !rightOuterJoin(skip, left, right) ||
          !allLeftFirst && !innerJoin(skip, left, right)) {
          continue;
        }
      } else if (type == JoinDesc.FULL_OUTER_JOIN) {
        if (tryLOForFO && leftOuterJoin(skip, left, right) > 0) {
          loopAgain = allLeftFirst;
          done = !loopAgain;
          tryLOForFO = false;
        } else if (allLeftFirst && !rightOuterJoin(skip, left, right) ||
          !allLeftFirst && !innerJoin(skip, left, right)) {
          continue;
        }
      }
      intermediate[aliasNum] = rightObj;

      if (aliasNum == numAliases - 1) {
        if (!(allLeftNull && rightNull)) {
          needToProduceLeftRow = true;
          if (needsPostEvaluation) {
            // This is only executed for outer joins with residual filters
            boolean forward = createForwardJoinObject(skipVectors[numAliases - 1]);
            producedRow |= forward;
            done = (type == JoinDesc.LEFT_SEMI_JOIN) && forward;
            if (!rightNull &&
                    (type == JoinDesc.RIGHT_OUTER_JOIN || type == JoinDesc.FULL_OUTER_JOIN)) {
              if (forward) {
                // This record produced a result this time, remove it from the storage
                // as it will not need to produce a result with NULL values anymore
                rowContainerPostFilteredOuterJoin.put(pos, null);
              } else {
                // We need to store this record (if it is not done yet) in case
                // we should produce a result
                if (!rowContainerPostFilteredOuterJoin.containsKey(pos)) {
                  Object[] row = Arrays.copyOfRange(forwardCache, offsets[aliasNum], offsets[aliasNum + 1]);
                  rowContainerPostFilteredOuterJoin.put(pos, row);
                }
              }
            }
          } else {
            createForwardJoinObject(skipVectors[numAliases - 1]);
          }
        }
      } else {
        // recursively call the join the other rhs tables
        genObject(aliasNum + 1, allLeftFirst && rightFirst, allLeftNull && rightNull);
      }
    }

    // Consolidation for outer joins
    if (needsPostEvaluation && aliasNum == numAliases - 1 &&
            needToProduceLeftRow && !producedRow && !allLeftNull) {
      if (type == JoinDesc.LEFT_OUTER_JOIN || type == JoinDesc.FULL_OUTER_JOIN) {
        // If it is a LEFT / FULL OUTER JOIN and the left record did not produce
        // results, we need to take that record, replace the right side with NULL
        // values, and produce the records
        int i = numAliases - 1;
        for (int j = offsets[i]; j < offsets[i + 1]; j++) {
          forwardCache[j] = null;
        }
        internalForward(forwardCache, outputObjInspector);
        countAfterReport = 0;
      }
    } else if (needsPostEvaluation && aliasNum == numAliases - 2) {
      int nextType = condn[aliasNum].getType();
      if (nextType == JoinDesc.RIGHT_OUTER_JOIN || nextType == JoinDesc.FULL_OUTER_JOIN) {
        // If it is a RIGHT / FULL OUTER JOIN, we need to iterate through the row container
        // that contains all the right records that did not produce results. Then, for each
        // of those records, we replace the left side with NULL values, and produce the
        // records.
        // Observe that we only enter this block when we have finished iterating through
        // all the left and right records (aliasNum == numAliases - 2), and thus, we have
        // tried to evaluate the post-filter condition on every possible combination.
        Arrays.fill(forwardCache, null);
        for (Object[] row : rowContainerPostFilteredOuterJoin.values()) {
          if (row == null) {
            continue;
          }
          System.arraycopy(row, 0, forwardCache, offsets[numAliases - 1], row.length);
          internalForward(forwardCache, outputObjInspector);
          countAfterReport = 0;
        }
      }
    }
  }
```

每来一条数据就会读取一下小表的内容，如果小表比较小，过程会比较快



**MapJoin**

- 也叫广播Join，Broadcast Join
- 从 (n-1)张小表创建Hashtable，Hashtable的键是 Joinkey, 把这张Hashtable广播到每一个结点的map上，只处理大表.
- 每一个大表的mapper在小表的hashtable中查找join key -> Join Result
- Ex: Join by “CityId”

MapJoin适合小表足够小的情况，否则就走 ReduceSinkOperator



如何决定MapJoin

内存要求：N-1 张小表必须能够完全读入内存

Hive决定MapJoin的两种方式（手动／自动）

- 手动，通过Query Hints（不再推荐）:

```sql
SELECT /*+ MAPJOIN(cities) */ FROM cities JOIN sales on cities.cityId=sales.cityId;
```


/*+ MAPJOIN(cities) */ 会决定把cities读入内存，放在hashTable里边，分发到每一个节点。

- 自动，打开(“hive.auto.convert.join”)，需要如果N-1张小表小于: “hive.mapjoin.smalltable.filesize”这个值
  



**MapJoin Optimizers**

- 构造查询计划Query Plan时，决定MapJoin优化
- 逻辑优化器Logical (Compile-time) optimizers ：修改逻辑执行计划，把JoinOperator修改成MapJoinOperator
- 物理优化器Physical (Runtime) optimizers： 修改物理执行计划(MapRedWork, TezWork, SparkWork), 引入条件判断等机制

逻辑优化之后ReduceSinkOperator.和普通的join operator被摘掉，换成mapjoin。

物理执行计划会被关联到具体的执行引擎，逻辑执行计划的小表部分会在本地执行，即左边小表在本地执行，逻辑执行计划的大表部分会被在远端执行。



**MapJoin Optimizers (MR)**

Query Hint: 编译时知道哪个表是小表的情况.（手动模式，加一个/*+ MAPJOIN(cities) */ 注释）

Logical Optimizer逻辑优化器: MapJoinProcessor

Auto-conversion: 编译时不知道哪个表是小表的情况（自动模式）

Physical Optimizer物理优化器: CommonJoinResolver, MapJoinResolver.

创建Conditional Tasks 把每个表是小表的情况考虑进去

Noconditional mode: 如果没有子查询的话，表的大小是在编译时可以知道的，否则是不知道的(join of intermediate results…)

自动模式模式分了三种情况，其中一个属于小表，这是前两种情况，第三种是都不是小表。

这个过程在CommonJoinResolver中





**BucketMapJoin**

- Bucketed 表: 根据不同的值分成不同的桶
- CREATE TABLE cities (cityid int, value string) CLUSTERED BY (cityId) INTO 2 BUCKETS;即建表的时候指定桶。
- 如果把分桶键（Bucket Key）作为关联键（Join Key）: For each bucket of table, rows with matching joinKey values will be in corresponding bucket of other table
- 像Mapjoin, but big-table mappers load to memory only relevant small-table bucket‟s hashmap

Ex: Bucketed by “CityId”, Join by “CityId”



**Bucket MapJoin 执行过程**

- 与MapJoin非常类似
- HashTableSink (小表) 写Hashtable是每个桶写一个Hashtable，而不是每张表写一个
- HashTableLoader (大表Mapper mapper) 也是每个桶读取一次HashTable



**SMB Join**

- CREATE TABLE cities (cityid int, cityName string) CLUSTERED BY (cityId) SORTED BY (cityId) INTO 2 BUCKETS;
- Join tables are bucketed and sorted (per bucket)
- This allows sort-merge join per bucket.
- Advance table until find a match

建表的时候对桶内的指定字段进行排序，这样的安排可以直接使用common join operator,不需要使用map join operator，直接把表读出来交给common join operator



**SMB Join**

- MR和Spark执行方式相同
- 用mapper处理大表，处理过程中直接读取对应的小表
- Map直接读取小表中相应的文件，相应的部分，避免了广播的开销
- 小表没有大小的限制
- 前提是，要知道经常使用哪个键做Join



**SMB Join Optimizers: MR**

- SMB 需要识别大表，以便在大表上运行mapper，执行过程中读取小表， 通常来说，在编译时决定
- 手动方法，用户可以手动提供hints
- Triggered by “hive.optimize.bucketmapjoin.sortedmerge”
- Logical Optimizer逻辑优化器: SortedMergeBucketMapJoinProc
- 自动触发: “hive.auto.convert.sortmerge.join.bigtable.selection.policy” 一个处理类
- Triggered by “hive.auto.convert.sortmerge.join”
- Logical Optimizer: SortedBucketMapJoinProc

逻辑优化器SortedMergeBucketMapjoinProc的处理过程：

`org/apache/hadoop/hive/ql/optimizer/SortedMergeBucketMapjoinProc.java`

```java
  @Override
  public Object process(Node nd, Stack<Node> stack, NodeProcessorCtx procCtx,
      Object... nodeOutputs) throws SemanticException {
    if (nd instanceof SMBMapJoinOperator) {
      return null;
    }

    MapJoinOperator mapJoinOp = (MapJoinOperator) nd;
    SortBucketJoinProcCtx smbJoinContext = (SortBucketJoinProcCtx) procCtx;

    boolean convert =
        canConvertBucketMapJoinToSMBJoin(mapJoinOp, stack, smbJoinContext, nodeOutputs);

    // Throw an error if the user asked for sort merge bucketed mapjoin to be enforced
    // and sort merge bucketed mapjoin cannot be performed
    if (!convert &&
        pGraphContext.getConf().getBoolVar(
            HiveConf.ConfVars.HIVEENFORCESORTMERGEBUCKETMAPJOIN)) {
      throw new SemanticException(ErrorMsg.SORTMERGE_MAPJOIN_FAILED.getMsg());
    }

    if (convert) {
      convertBucketMapJoinToSMBJoin(mapJoinOp, smbJoinContext);
    }
    return null;
  }
```

join operator是最基本的，其他的mapjoin，SMB都是属于优化。



**倾斜关联Skew Join**

- 倾斜键Skew keys = 高频出现的键, 非常多的键，多到一个reduce处理不了
- 使用Common Join处理非倾斜键，使用Map Join处理倾斜键.
- A join B on A.id=B.id, 如果A 表中id=1倾斜, 那么查询会变成

```sql
A join B on A.id=B.id and A.id!=1 
union
A join B on A.id=B.id and A.id=1
```

判断是否是倾斜的，主要是判断建是不是倾斜的，那么怎么判断一个建是不是倾斜的呢？



**Skew Join Optimizers (Compile Time, MR)**

-  建表时指定倾斜键: create table … skewed by (key) on (key_value);
- 开关“hive.optimize.skewjoin.compiletime”
- Logical Optimizer逻辑优化器: SkewJoinOptimizer查看元数据

直接指定倾斜建，是最好的一种，他会直接给出union的方式处理倾斜

但是实际环境是苛刻的很多情况并不知道那个建会倾斜，往下看



Skew Join Optimizers (Runtime, MR)

- 开关“hive.optimize.skewjoin”
- Physical Optimizer: SkewJoinResolver
- JoinOperator处理时候计数，如果某个可以被某个节点处理次数超过 “hive.skewjoin.key” 域值
- 倾斜键Skew key被跳过并且把值拷到单独的目录
- ConditionalTask会单独针对倾斜的键作处理，并将结果作Union

即最终结果是倾斜的建处理之后的结果加上未倾斜的common join的结果。不可否认这是一种笨重的方法，最好的方法是直接指定那个键会倾斜，单独处理倾斜。当出现处理慢的时候我们排查是join慢还是group by慢，如果是join慢能不能用map join处理，如果是group by慢，能不能进行预聚合。
