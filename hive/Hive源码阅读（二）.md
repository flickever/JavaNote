# Hive源码阅读（二）

**总流程图**

```flow
hql=>operation: Hive SQL
AST=>operation: AST
QB=>operation: QB
optree=>operation: Opeator Tree
tasktree=>operation: Task Tree

hql(right)->AST(right)->QB(right)->optree(right)->tasktree
```


### **第一部分：语法分析**

**Hive SQL转为AST语法树**

1. `ParseUtils`封装了`ParseDriver `对SQL的解析工作，`ParseUtils`的parse方法：

```java
  /** Parses the Hive query. */
  public static ASTNode parse(
      String command, Context ctx, String viewFullyQualifiedName) throws ParseException {
    ParseDriver pd = new ParseDriver(); // 解析驱动类
    // 核心解析流程
    ASTNode tree = pd.parse(command, ctx, viewFullyQualifiedName);
    tree = findRootNonNullToken(tree);
    handleSetColRefs(tree);
    return tree;
  }
```

2. `ParseDriver` 对command进行词法分析和语法解析（统称为语法分析），返回一个抽象语法树AST，进入`parseDriver`的parse方法：

```java
 /**
   * Parses a command, optionally assigning the parser's token stream to the
   * given context.
   *
   * @param command
   *          command to parse
   *
   * @param ctx
   *          context with which to associate this parser's token stream, or
   *          null if either no context is available or the context already has
   *          an existing stream
   *
   * @return parsed AST
   */
  public ASTNode parse(String command, Context ctx, String viewFullyQualifiedName)
      throws ParseException {
    if (LOG.isDebugEnabled()) {
      LOG.debug("Parsing command: " + command);
    }

    // 词法分析
    HiveLexerX lexer = new HiveLexerX(new ANTLRNoCaseStringStream(command));
    // 根据词法分析的结果得到tokens，此时不只是单纯的字符串，而是具有特殊意义的字符串的封装，本身是一个流
    TokenRewriteStream tokens = new TokenRewriteStream(lexer);
    if (ctx != null) {
      if (viewFullyQualifiedName == null) {
        // Top level query
        ctx.setTokenRewriteStream(tokens);
      } else {
        // It is a view
        ctx.addViewTokenRewriteStream(viewFullyQualifiedName, tokens);
      }
      lexer.setHiveConf(ctx.getConf());
    }
    // 解析Token
    HiveParser parser = new HiveParser(tokens);
    if (ctx != null) {
      parser.setHiveConf(ctx.getConf());
    }
    parser.setTreeAdaptor(adaptor);
    HiveParser.statement_return r = null;
    try {
      // 得到结果
      r = parser.statement();
    } catch (RecognitionException e) {
      e.printStackTrace();
      throw new ParseException(parser.errors);
    }

    if (lexer.getErrors().size() == 0 && parser.errors.size() == 0) {
      LOG.debug("Parse Completed");
    } else if (lexer.getErrors().size() != 0) {
      throw new ParseException(lexer.getErrors());
    } else {
      throw new ParseException(parser.errors);
    }

    ASTNode tree = (ASTNode) r.getTree();
    tree.setUnknownTokenBoundaries();
    return tree;
  }
```

**词法分析器Lexer - HiveLexerX**

- 输入：HiveSQL
- 输出：一串Toker，这里是TokenRewriteStream
- 也称词法分析器 **Lexical Analyzer(LA)**或者Scanner
- 可以参考《编译原理》

文件定义了一些hive的关键字，form、where，数字的定义格式【0–9】，分隔符，比较符之类的etc。每一个关键字都会变成一个token。

**语法解析 HiveParser:**

- 获得ASTNode
- HiveParser.statement().getTree()
- HiveParser是Antlr根据HiveParser.g生成的文件



### **第二部分：语义解析初步 **

**SemanticAnalyzer**

- 输入：AST树
- 输出：Operator树

`org/apache/hadoop/hive/ql/parse/SemanticAnalyzer.java`

继承自BaseSemanticAnalyzer的语义分析器有很多种，其中最重要的是用于查询的SemanticAnalyzer类，在Hive3.1.2中，这个类有接近一万五千行代码

输入的ASTTree后续的QB的生成，逻辑执行计划、逻辑执行计划的优化、物理执行计划的切分、物理执行计划的优化、以及mr任务的生成全部都在这1万多行的代码里边的逻辑中。

接下来看这个类中的方法

#### 2.1 **生成QB**

`genResolvedParseTree()`

```java
  boolean genResolvedParseTree(ASTNode ast, PlannerContext plannerCtx) throws SemanticException {
    ASTNode child = ast;
    this.ast = ast;
    viewsExpanded = new ArrayList<String>();
    ctesExpanded = new ArrayList<String>();

    // 1. analyze and process the position alias
    // step processPositionAlias out of genResolvedParseTree

    // 2. analyze create table command
    .....

    // 3. analyze create view command
    .......

    // 4. continue analyzing from the child ASTNode.
    Phase1Ctx ctx_1 = initPhase1Ctx();
    if (!doPhase1(child, qb, ctx_1, plannerCtx)) {
      // if phase1Result false return
      return false;
    }
    LOG.info("Completed phase 1 of Semantic Analysis");

    // 5. Resolve Parse Tree
    // Materialization is allowed if it is not a view definition
    getMetaData(qb, createVwDesc == null);
    LOG.info("Completed getting MetaData in Semantic Analysis");

    plannerCtx.setParseTreeAttr(child, ctx_1);

    return true;
  }
```

doPhase1执行成功那么就会得到一个QB

`doPhase1`

```java
/**
   * Phase 1: (including, but not limited to):
   *
   * 1. Gets all the aliases for all the tables / subqueries and makes the
   * appropriate mapping in aliasToTabs, aliasToSubq 2. Gets the location of the
   * destination and names the clause "inclause" + i 3. Creates a map from a
   * string representation of an aggregation tree to the actual aggregation AST
   * 4. Creates a mapping from the clause name to the select expression AST in
   * destToSelExpr 5. Creates a mapping from a table alias to the lateral view
   * AST's in aliasToLateralViews
   *
   * @param ast
   * @param qb
   * @param ctx_1
   * @throws SemanticException
   */
  @SuppressWarnings({"fallthrough", "nls"})
  public boolean doPhase1(ASTNode ast, QB qb, Phase1Ctx ctx_1, PlannerContext plannerCtx)
      throws SemanticException {

    boolean phase1Result = true;
    QBParseInfo qbp = qb.getParseInfo();
    boolean skipRecursion = false;

    if (ast.getToken() != null) {
      skipRecursion = true;
      switch (ast.getToken().getType()) {
      case HiveParser.TOK_SELECTDI:
        qb.countSelDi();
        // fall through
      // 重点关注select类型token
      case HiveParser.TOK_SELECT:
        qb.countSel();// 标记qb类型
        qbp.setSelExprForClause(ctx_1.dest, ast);
        ............

      // where类型token
      case HiveParser.TOK_WHERE:
        //对where的孩子进行处理，为什么是ast.getChild(0)？这个是和之前的HiveParser.g结构相辅相成的
        qbp.setWhrExprForClause(ctx_1.dest, ast);
        if (!SubQueryUtils.findSubQueries((ASTNode) ast.getChild(0)).isEmpty()) {
          queryProperties.setFilterWithSubQuery(true);
        }
        break;

      ..........

      case HiveParser.TOK_GROUPBY:
      case HiveParser.TOK_ROLLUP_GROUPBY:
      case HiveParser.TOK_CUBE_GROUPBY:
      case HiveParser.TOK_GROUPING_SETS:
        ...........
            
      ...........

    if (!skipRecursion) {
      // Iterate over the rest of the children
      int child_count = ast.getChildCount();
      for (int child_pos = 0; child_pos < child_count && phase1Result; ++child_pos) {
        // Recurse
        phase1Result = phase1Result && doPhase1(
            (ASTNode)ast.getChild(child_pos), qb, ctx_1, plannerCtx);
      }
    }
    return phase1Result;
  }
```

1. 参数qb是一个空的QB，在不同case类型下对齐进行填满。
2. doPhase1对ASTTree中的每个元素的TOK类型进行case，针对于不同的case对节点数据进行填充。
3. for遍历整棵ASTTree，中间对每个元素递归调用doPhase1，这种方式是一种深度优先搜索的算法。
4. 经过一轮深度优先遍历，不带元数据的QB树就生成了。
    

doPhase1执行完毕之后得到QB，QB里边的只是一些关键字还有一些表的名字，但是和hdfs的文件路径对应不起来，所以需要metaData映射关系，之后在SemanticAnalyzer中调用了getMetaData

`getMetaData`

```java
  public void getMetaData(QB qb) throws SemanticException {
    getMetaData(qb, false);
  }

  public void getMetaData(QB qb, boolean enableMaterialization) throws SemanticException {
    try {
      if (enableMaterialization) {
        getMaterializationMetadata(qb);
      }
      getMetaData(qb, null);
    } catch (HiveException e) {
      // Has to use full name to make sure it does not conflict with
      // org.apache.commons.lang.StringUtils
      LOG.error(org.apache.hadoop.util.StringUtils.stringifyException(e));
      if (e instanceof SemanticException) {
        throw (SemanticException)e;
      }
      throw new SemanticException(e.getMessage(), e);
    }
  }
```

getMetaData会递归的去取元数据（从mysql中），经过doPhase1和getMetaData得到一个完整的QB，接下来就是逻辑执行技术的生成。



#### 2.2 生成OP

- genPlan()实现QB->Operator
- genPlan() 也是深度优先的递归

`genOPTree`

```java
Operator genOPTree(ASTNode ast, PlannerContext plannerCtx) throws SemanticException {
  // fetch all the hints in qb
  List<ASTNode> hintsList = new ArrayList<>();
  getHintsFromQB(qb, hintsList);
  getQB().getParseInfo().setHintList(hintsList);
  return genPlan(qb);
}
```

`genPlan`

该方法递归过程，不同重载方法之间有引用

```java
  private Operator genPlan(QB parent, QBExpr qbexpr) throws SemanticException {
    if (qbexpr.getOpcode() == QBExpr.Opcode.NULLOP) {
      boolean skipAmbiguityCheck = viewSelect == null && parent.isTopLevelSelectStarQuery();
      return genPlan(qbexpr.getQB(), skipAmbiguityCheck);
    }
    if (qbexpr.getOpcode() == QBExpr.Opcode.UNION) {
      Operator qbexpr1Ops = genPlan(parent, qbexpr.getQBExpr1());
      Operator qbexpr2Ops = genPlan(parent, qbexpr.getQBExpr2());

      return genUnionPlan(qbexpr.getAlias(), qbexpr.getQBExpr1().getAlias(),
          qbexpr1Ops, qbexpr.getQBExpr2().getAlias(), qbexpr2Ops);
    }
    return null;
  }
```

```java
  @SuppressWarnings("nls")
  public Operator genPlan(QB qb, boolean skipAmbiguityCheck)
      throws SemanticException {

    // First generate all the opInfos for the elements in the from clause
    // Must be deterministic order map - see HIVE-8707
    Map<String, Operator> aliasToOpInfo = new LinkedHashMap<String, Operator>();

    // Recurse over the subqueries to fill the subquery part of the plan
    for (String alias : qb.getSubqAliases()) {
      QBExpr qbexpr = qb.getSubqForAlias(alias);
      Operator<?> operator = genPlan(qb, qbexpr);
      aliasToOpInfo.put(alias, operator);
      if (qb.getViewToTabSchema().containsKey(alias)) {
        // we set viewProjectToTableSchema so that we can leverage ColumnPruner.
        if (operator instanceof LimitOperator) {
          // If create view has LIMIT operator, this can happen
          // Fetch parent operator
          operator = operator.getParentOperators().get(0);
        }
        if (operator instanceof SelectOperator) {
          if (this.viewProjectToTableSchema == null) {
            this.viewProjectToTableSchema = new LinkedHashMap<>();
          }
          viewProjectToTableSchema.put((SelectOperator) operator, qb.getViewToTabSchema()
              .get(alias));
        } else {
          throw new SemanticException("View " + alias + " is corresponding to "
              + operator.getType().name() + ", rather than a SelectOperator.");
        }
      }
    }

    // Recurse over all the source tables
    for (String alias : qb.getTabAliases()) {
      if(alias.equals(DUMMY_TABLE)) {
        continue;
      }
      Operator op = genTablePlan(alias, qb);
      aliasToOpInfo.put(alias, op);
    }

    if (aliasToOpInfo.isEmpty()) {
      qb.getMetaData().setSrcForAlias(DUMMY_TABLE, getDummyTable());
      TableScanOperator op = (TableScanOperator) genTablePlan(DUMMY_TABLE, qb);
      op.getConf().setRowLimit(1);
      qb.addAlias(DUMMY_TABLE);
      qb.setTabAlias(DUMMY_TABLE, DUMMY_TABLE);
      aliasToOpInfo.put(DUMMY_TABLE, op);
    }

    Operator srcOpInfo = null;
    Operator lastPTFOp = null;

    if(queryProperties.hasPTF()){
      //After processing subqueries and source tables, process
      // partitioned table functions

      HashMap<ASTNode, PTFInvocationSpec> ptfNodeToSpec = qb.getPTFNodeToSpec();
      if ( ptfNodeToSpec != null ) {
        for(Entry<ASTNode, PTFInvocationSpec> entry : ptfNodeToSpec.entrySet()) {
          ASTNode ast = entry.getKey();
          PTFInvocationSpec spec = entry.getValue();
          String inputAlias = spec.getQueryInputName();
          Operator inOp = aliasToOpInfo.get(inputAlias);
          if ( inOp == null ) {
            throw new SemanticException(generateErrorMessage(ast,
                "Cannot resolve input Operator for PTF invocation"));
          }
          lastPTFOp = genPTFPlan(spec, inOp);
          String ptfAlias = spec.getFunction().getAlias();
          if ( ptfAlias != null ) {
            aliasToOpInfo.put(ptfAlias, lastPTFOp);
          }
        }
      }

    }

    // For all the source tables that have a lateral view, attach the
    // appropriate operators to the TS
    genLateralViewPlans(aliasToOpInfo, qb);


    // process join
    if (qb.getParseInfo().getJoinExpr() != null) {
      ASTNode joinExpr = qb.getParseInfo().getJoinExpr();

      if (joinExpr.getToken().getType() == HiveParser.TOK_UNIQUEJOIN) {
        QBJoinTree joinTree = genUniqueJoinTree(qb, joinExpr, aliasToOpInfo);
        qb.setQbJoinTree(joinTree);
      } else {
        QBJoinTree joinTree = genJoinTree(qb, joinExpr, aliasToOpInfo);
        qb.setQbJoinTree(joinTree);
        /*
         * if there is only one destination in Query try to push where predicates
         * as Join conditions
         */
        Set<String> dests = qb.getParseInfo().getClauseNames();
        if ( dests.size() == 1 && joinTree.getNoOuterJoin()) {
          String dest = dests.iterator().next();
          ASTNode whereClause = qb.getParseInfo().getWhrForClause(dest);
          if ( whereClause != null ) {
            extractJoinCondsFromWhereClause(joinTree, qb, dest,
                (ASTNode) whereClause.getChild(0),
                aliasToOpInfo );
          }
        }

        if (!disableJoinMerge) {
          mergeJoinTree(qb);
        }
      }

      // if any filters are present in the join tree, push them on top of the
      // table
      pushJoinFilters(qb, qb.getQbJoinTree(), aliasToOpInfo);
      srcOpInfo = genJoinPlan(qb, aliasToOpInfo);
    } else {
      // Now if there are more than 1 sources then we have a join case
      // later we can extend this to the union all case as well
      srcOpInfo = aliasToOpInfo.values().iterator().next();
      // with ptfs, there maybe more (note for PTFChains:
      // 1 ptf invocation may entail multiple PTF operators)
      srcOpInfo = lastPTFOp != null ? lastPTFOp : srcOpInfo;
    }

    Operator bodyOpInfo = genBodyPlan(qb, srcOpInfo, aliasToOpInfo);

    if (LOG.isDebugEnabled()) {
      LOG.debug("Created Plan for Query Block " + qb.getId());
    }

    if (qb.getAlias() != null) {
      rewriteRRForSubQ(qb.getAlias(), bodyOpInfo, skipAmbiguityCheck);
    }

    setQB(qb);
    return bodyOpInfo;
  }
```



#### 2.3 **表达式分析**

- 类型推导：100->INT，100.1 ->DOUBLE，'Hello'-> STRING， TRUE -> BOOL
- 隐式类型转换 ：1+2.5 => double(1) + 2.5
- NULL值类型转换
- f(g(A), B) => A，g()，B，f() ，逆波兰表达式
- BOOL表达式分析：`(C1 and C2) or C3 ` =>`(C1 or C3) and (C2 or C3)`
  - `(T.A>10 AND P.B<100) OR T.B>10` => `(T.A>10 OR T.B>10) AND (P.B<100 OR T.B>10) `
  - 当条件变换为合取范式时，可以对AND连接的每一项进行下推优化



#### 2.4 Operator分类

**Operators对应SQL**

| Operators          | 描述                     | 对应SQL关键字            |
| ------------------ | ------------------------ | ------------------------ |
| TableScanOperator  | 从表中读数据             | From                     |
| ReduceSinkOperator | 数据发给Reduce端聚合     | 语句有Join/GroupBy的会有 |
| JoinOperator       | 两个表关联Join           | Join                     |
| SelectOperator     | 选择部分列输出           | Select                   |
| FileSinkOperator   | 建立结果数据并发送给文件 | 每个SQL都有              |
| FilterOperator     | 过滤输入数据             | Where/Having             |
| GroupByOperator    | 对数据Group By操作       | Group By/Distinct        |

