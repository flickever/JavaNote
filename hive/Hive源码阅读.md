# Hive源码阅读（一）

## 第一部分：读取命令

### run方法

入口在org.apache.hadoop.hive.cli.CliDriver

首先运行的是run方法，可以看到

首先是对系统输入参数做一个校验，校验参数失败返回系统参数1

注册系统输入输出流，失败返回系统参数3

对用户传参校验（-e -f 等等）

调用`executeDriver(ss, conf, oproc)`(核心)

```java
 public  int run(String[] args) throws Exception {

    OptionsProcessor oproc = new OptionsProcessor();
    if (!oproc.process_stage1(args)) {
      return 1;
    }

    // NOTE: It is critical to do this here so that log4j is reinitialized
    // before any of the other core hive classes are loaded
    boolean logInitFailed = false;
    String logInitDetailMessage;
    try {
      logInitDetailMessage = LogUtils.initHiveLog4j();
    } catch (LogInitializationException e) {
      logInitFailed = true;
      logInitDetailMessage = e.getMessage();
    }

    CliSessionState ss = new CliSessionState(new HiveConf(SessionState.class));
    ss.in = System.in;
    try {
      ss.out = new PrintStream(System.out, true, "UTF-8");
      ss.info = new PrintStream(System.err, true, "UTF-8");
      ss.err = new CachingPrintStream(System.err, true, "UTF-8");
    } catch (UnsupportedEncodingException e) {
      return 3;
    }

    if (!oproc.process_stage2(ss)) {
      return 2;
    }

    if (!ss.getIsSilent()) {
      if (logInitFailed) {
        System.err.println(logInitDetailMessage);
      } else {
        SessionState.getConsole().printInfo(logInitDetailMessage);
      }
    }

    // set all properties specified via command line
    // 获得输入配置，封装为Map 
    HiveConf conf = ss.getConf();
    for (Map.Entry<Object, Object> item : ss.cmdProperties.entrySet()) {
      conf.set((String) item.getKey(), (String) item.getValue());
      ss.getOverriddenConfigurations().put((String) item.getKey(), (String) item.getValue());
    }

    // read prompt configuration and substitute variables.
    prompt = conf.getVar(HiveConf.ConfVars.CLIPROMPT); // 打印hive前缀
    prompt = new VariableSubstitution(new HiveVariableSource() {
      @Override
      public Map<String, String> getHiveVariable() {
        return SessionState.get().getHiveVariables();
      }
    }).substitute(conf, prompt);
    prompt2 = spacesForString(prompt);

    if (HiveConf.getBoolVar(conf, ConfVars.HIVE_CLI_TEZ_SESSION_ASYNC)) {
      // Start the session in a fire-and-forget manner. When the asynchronously initialized parts of
      // the session are needed, the corresponding getters and other methods will wait as needed.
      SessionState.beginStart(ss, console);
    } else {
      SessionState.start(ss);
    }

    ss.updateThreadName();

    // Create views registry
    HiveMaterializedViewsRegistry.get().init();

    // execute cli driver work
    try {
      return executeDriver(ss, conf, oproc);
    } finally {
      ss.resetThreadName();
      ss.close();
    }
  }
```



### executeDriver方法

读取hive的输出sql，解析后把语句传入`processLine`方法

```java
private int executeDriver(CliSessionState ss, HiveConf conf, OptionsProcessor oproc)
      throws Exception {

    CliDriver cli = new CliDriver();
    cli.setHiveVariables(oproc.getHiveVariables());

    // use the specified database if specified
    cli.processSelectDatabase(ss);

    // Execute -i init files (always in silent mode)
    cli.processInitFiles(ss);

    if (ss.execString != null) {
      int cmdProcessStatus = cli.processLine(ss.execString);
      return cmdProcessStatus;
    }

    try {
      if (ss.fileName != null) {
        return cli.processFile(ss.fileName);
      }
    } catch (FileNotFoundException e) {
      System.err.println("Could not open input file for reading. (" + e.getMessage() + ")");
      return 3;
    }
    if ("mr".equals(HiveConf.getVar(conf, ConfVars.HIVE_EXECUTION_ENGINE))) {
      console.printInfo(HiveConf.generateMrDeprecationWarning());
    }

    setupConsoleReader();

    String line;
    int ret = 0;  // 系统退出参数
    String prefix = "";
    String curDB = getFormattedDb(conf, ss);
    String curPrompt = prompt + curDB;
    String dbSpaces = spacesForString(curDB);

    // 
    while ((line = reader.readLine(curPrompt + "> ")) != null) {
      if (!prefix.equals("")) {
        prefix += '\n';
      }
      if (line.trim().startsWith("--")) {
        continue;
      }
      if (line.trim().endsWith(";") && !line.trim().endsWith("\\;")) {
        line = prefix + line;
        ret = cli.processLine(line, true);
        prefix = "";
        curDB = getFormattedDb(conf, ss);
        curPrompt = prompt + curDB;
        dbSpaces = dbSpaces.length() == curDB.length() ? dbSpaces : spacesForString(curDB);
      } else {
        prefix = prefix + line;
        curPrompt = prompt2 + dbSpaces;
        continue;
      }
    }

    return ret;
  }
```



### processLine方法

1. 可以通过Ctrl+C杀死该任务
2. 按照分号切分语句（有可能读进来的是文件）
3. 调用`processCmd`

```scala
 public int processLine(String line, boolean allowInterrupting) {
    SignalHandler oldSignal = null;
    Signal interruptSignal = null;

    if (allowInterrupting) { // 可以通过Ctrl+C杀死该任务
      // Remember all threads that were running at the time we started line processing.
      // Hook up the custom Ctrl+C handler while processing this line
      interruptSignal = new Signal("INT");
      oldSignal = Signal.handle(interruptSignal, new SignalHandler() {
        private boolean interruptRequested;

        @Override
        public void handle(Signal signal) {
          boolean initialRequest = !interruptRequested;
          interruptRequested = true;

          // Kill the VM on second ctrl+c
          if (!initialRequest) {
            console.printInfo("Exiting the JVM");
            System.exit(127);
          }

          // Interrupt the CLI thread to stop the current statement and return
          // to prompt
          console.printInfo("Interrupting... Be patient, this might take some time.");
          console.printInfo("Press Ctrl+C again to kill JVM");

          // First, kill any running MR jobs
          HadoopJobExecHelper.killRunningJobs();
          TezJobExecHelper.killRunningJobs();
          HiveInterruptUtils.interrupt();
        }
      });
    }

    try {
      int lastRet = 0, ret = 0;

      // we can not use "split" function directly as ";" may be quoted
      List<String> commands = splitSemiColon(line);

      String command = "";
      for (String oneCmd : commands) {

        if (StringUtils.endsWith(oneCmd, "\\")) {
          command += StringUtils.chop(oneCmd) + ";";
          continue;
        } else {
          command += oneCmd;
        }
        if (StringUtils.isBlank(command)) {
          continue;
        }

        ret = processCmd(command);
        command = "";
        lastRet = ret;
        boolean ignoreErrors = HiveConf.getBoolVar(conf, HiveConf.ConfVars.CLIIGNOREERRORS);
        if (ret != 0 && !ignoreErrors) {
          return ret;
        }
      }
      return lastRet;
    } finally {
      // Once we are done processing the line, restore the old handler
      if (oldSignal != null && interruptSignal != null) {
        Signal.handle(interruptSignal, oldSignal);
      }
    }
  }
```



### processCmd方法

解析特殊命令

quit/exit：退出

source：执行一个hql文件

！开头命令：shell命令

以上情况都不是，就表示这是hql命令，调用processLocalCmd方法

```java
  public int processCmd(String cmd) {
    CliSessionState ss = (CliSessionState) SessionState.get();
    ss.setLastCommand(cmd);

    ss.updateThreadName();

    // Flush the print stream, so it doesn't include output from the last command
    ss.err.flush();
    String cmd_trimmed = HiveStringUtils.removeComments(cmd).trim();
    String[] tokens = tokenizeCmd(cmd_trimmed);
    int ret = 0;

    if (cmd_trimmed.toLowerCase().equals("quit") || cmd_trimmed.toLowerCase().equals("exit")) {

      // if we have come this far - either the previous commands
      // are all successful or this is command line. in either case
      // this counts as a successful run
      ss.close();
      System.exit(0);

    } else if (tokens[0].equalsIgnoreCase("source")) {
      String cmd_1 = getFirstCmd(cmd_trimmed, tokens[0].length());
      cmd_1 = new VariableSubstitution(new HiveVariableSource() {
        @Override
        public Map<String, String> getHiveVariable() {
          return SessionState.get().getHiveVariables();
        }
      }).substitute(ss.getConf(), cmd_1);

      File sourceFile = new File(cmd_1);
      if (! sourceFile.isFile()){
        console.printError("File: "+ cmd_1 + " is not a file.");
        ret = 1;
      } else {
        try {
          ret = processFile(cmd_1);
        } catch (IOException e) {
          console.printError("Failed processing file "+ cmd_1 +" "+ e.getLocalizedMessage(),
            stringifyException(e));
          ret = 1;
        }
      }
    } else if (cmd_trimmed.startsWith("!")) {
      // for shell commands, use unstripped command
      String shell_cmd = cmd.trim().substring(1);
      shell_cmd = new VariableSubstitution(new HiveVariableSource() {
        @Override
        public Map<String, String> getHiveVariable() {
          return SessionState.get().getHiveVariables();
        }
      }).substitute(ss.getConf(), shell_cmd);

      // shell_cmd = "/bin/bash -c \'" + shell_cmd + "\'";
      try {
        ShellCmdExecutor executor = new ShellCmdExecutor(shell_cmd, ss.out, ss.err);
        ret = executor.execute();
        if (ret != 0) {
          console.printError("Command failed with exit code = " + ret);
        }
      } catch (Exception e) {
        console.printError("Exception raised from Shell command " + e.getLocalizedMessage(),
            stringifyException(e));
        ret = 1;
      }
    }  else { // local mode
      try {

        try (CommandProcessor proc = CommandProcessorFactory.get(tokens, (HiveConf) conf)) {
          if (proc instanceof IDriver) {
            // Let Driver strip comments using sql parser
            ret = processLocalCmd(cmd, proc, ss);
          } else {
            ret = processLocalCmd(cmd_trimmed, proc, ss);
          }
        }
      } catch (SQLException e) {
        console.printError("Failed processing command " + tokens[0] + " " + e.getLocalizedMessage(),
          org.apache.hadoop.util.StringUtils.stringifyException(e));
        ret = 1;
      }
      catch (Exception e) {
        throw new RuntimeException(e);
      }
    }

    ss.resetThreadName();
    return ret;
  }
```



### processLocalCmd方法

调用真正执行hql的run方法，根据run方法的返回数据，在控制台输出数据以及打印信息

```java
int processLocalCmd(String cmd, CommandProcessor proc, CliSessionState ss) {
    boolean escapeCRLF = HiveConf.getBoolVar(conf, HiveConf.ConfVars.HIVE_CLI_PRINT_ESCAPE_CRLF);
    int ret = 0;

    if (proc != null) {
      if (proc instanceof IDriver) {
        IDriver qp = (IDriver) proc;
        PrintStream out = ss.out;
        long start = System.currentTimeMillis();
        if (ss.getIsVerbose()) {
          out.println(cmd);
        }
		// 真正的执行方法
        ret = qp.run(cmd).getResponseCode();
        if (ret != 0) {
          qp.close();
          return ret;
        }

        // query has run capture the time
        long end = System.currentTimeMillis();
        double timeTaken = (end - start) / 1000.0;

        ArrayList<String> res = new ArrayList<String>();

        printHeader(qp, out);

        // print the results
        int counter = 0;
        try {
          if (out instanceof FetchConverter) {
            ((FetchConverter) out).fetchStarted();
          }
          while (qp.getResults(res)) {
            for (String r : res) {
                  if (escapeCRLF) {
                    r = EscapeCRLFHelper.escapeCRLF(r);
                  }
              out.println(r);
            }
            counter += res.size();
            res.clear();
            if (out.checkError()) {
              break;
            }
          }
        } catch (IOException e) {
          console.printError("Failed with exception " + e.getClass().getName() + ":" + e.getMessage(),
              "\n" + org.apache.hadoop.util.StringUtils.stringifyException(e));
          ret = 1;
        }

        qp.close();

        if (out instanceof FetchConverter) {
          ((FetchConverter) out).fetchFinished();
        }

        console.printInfo(
            "Time taken: " + timeTaken + " seconds" + (counter == 0 ? "" : ", Fetched: " + counter + " row(s)"));
      } else {
        String firstToken = tokenizeCmd(cmd.trim())[0];
        String cmd_1 = getFirstCmd(cmd.trim(), firstToken.length());

        if (ss.getIsVerbose()) {
          ss.out.println(firstToken + " " + cmd_1);
        }
        CommandProcessorResponse res = proc.run(cmd_1);
        if (res.getResponseCode() != 0) {
          ss.out
              .println("Query returned non-zero code: " + res.getResponseCode() + ", cause: " + res.getErrorMessage());
        }
        if (res.getConsoleMessages() != null) {
          for (String consoleMsg : res.getConsoleMessages()) {
            console.printInfo(consoleMsg);
          }
        }
        ret = res.getResponseCode();
      }
    }

    return ret;
  }
```



### run方法

这是一个接口方法，由`org.apache.hadoop.hive.ql.Driver`实现

第二个入参代表是否已经编译

调用`runInternal`方法

```java
public CommandProcessorResponse run(String command, boolean alreadyCompiled) {

    try {
      runInternal(command, alreadyCompiled);
      return createProcessorResponse(0);
    } catch (CommandProcessorResponse cpr) {

    SessionState ss = SessionState.get();
    if(ss == null) {
      return cpr;
    }
    MetaDataFormatter mdf = MetaDataFormatUtils.getFormatter(ss.getConf());
    if(!(mdf instanceof JsonMetaDataFormatter)) {
      return cpr;
    }
    /*Here we want to encode the error in machine readable way (e.g. JSON)
     * Ideally, errorCode would always be set to a canonical error defined in ErrorMsg.
     * In practice that is rarely the case, so the messy logic below tries to tease
     * out canonical error code if it can.  Exclude stack trace from output when
     * the error is a specific/expected one.
     * It's written to stdout for backward compatibility (WebHCat consumes it).*/
    try {
      if(downstreamError == null) {
        mdf.error(ss.out, errorMessage, cpr.getResponseCode(), SQLState);
        return cpr;
      }
      ErrorMsg canonicalErr = ErrorMsg.getErrorMsg(cpr.getResponseCode());
      if(canonicalErr != null && canonicalErr != ErrorMsg.GENERIC_ERROR) {
        /*Some HiveExceptions (e.g. SemanticException) don't set
          canonical ErrorMsg explicitly, but there is logic
          (e.g. #compile()) to find an appropriate canonical error and
          return its code as error code. In this case we want to
          preserve it for downstream code to interpret*/
        mdf.error(ss.out, errorMessage, cpr.getResponseCode(), SQLState, null);
        return cpr;
      }
      if(downstreamError instanceof HiveException) {
        HiveException rc = (HiveException) downstreamError;
        mdf.error(ss.out, errorMessage,
                rc.getCanonicalErrorMsg().getErrorCode(), SQLState,
                rc.getCanonicalErrorMsg() == ErrorMsg.GENERIC_ERROR ?
                        org.apache.hadoop.util.StringUtils.stringifyException(rc)
                        : null);
      }
      else {
        ErrorMsg canonicalMsg =
                ErrorMsg.getErrorMsg(downstreamError.getMessage());
        mdf.error(ss.out, errorMessage, canonicalMsg.getErrorCode(),
                SQLState, org.apache.hadoop.util.StringUtils.
                stringifyException(downstreamError));
      }
    }
    catch(HiveException ex) {
      console.printError("Unable to JSON-encode the error",
              org.apache.hadoop.util.StringUtils.stringifyException(ex));
    }
    return cpr;
    }
  }
```



### runInternal方法

如果已编译，.......

如果未编译，调用`compileInternal`方法

`compileInternal`调用`compile`方法

之后run方法调用`execute`方法



## 第二部分：编译

### compileInternal方法

包括三部分：

1. 解析语法，转为抽象语法树（AST）
2. 编译器编译为Operator树（操作树）
3. 优化：逻辑优化和物理优化

`compileInternal`方法调用了`compile`方法，该方法内部实现了解析器，编译器，优化器三个内容

```java
private void compileInternal(String command, boolean deferClose) throws CommandProcessorResponse {
    Metrics metrics = MetricsFactory.getInstance();
    if (metrics != null) {
      metrics.incrementCounter(MetricsConstant.WAITING_COMPILE_OPS, 1);
    }

    PerfLogger perfLogger = SessionState.getPerfLogger();
    perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.WAIT_COMPILE);
    final ReentrantLock compileLock = tryAcquireCompileLock(isParallelEnabled,
      command);
    perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.WAIT_COMPILE);
    if (metrics != null) {
      metrics.decrementCounter(MetricsConstant.WAITING_COMPILE_OPS, 1);
    }

    if (compileLock == null) {
      throw createProcessorResponse(ErrorMsg.COMPILE_LOCK_TIMED_OUT.getErrorCode());
    }


    try {
      compile(command, true, deferClose);
    } catch (CommandProcessorResponse cpr) {
      try {
        releaseLocksAndCommitOrRollback(false);
      } catch (LockException e) {
        LOG.warn("Exception in releasing locks. " + org.apache.hadoop.util.StringUtils.stringifyException(e));
      }
      throw cpr;
    } finally {
      compileLock.unlock();
    }

    //Save compile-time PerfLogging for WebUI.
    //Execution-time Perf logs are done by either another thread's PerfLogger
    //or a reset PerfLogger.
    queryDisplay.setPerfLogStarts(QueryDisplay.Phase.COMPILATION, perfLogger.getStartTimes());
    queryDisplay.setPerfLogEnds(QueryDisplay.Phase.COMPILATION, perfLogger.getEndTimes());
  }
```



### compile方法

内部有两个方法实现功能

```java
private void compile(String command, boolean resetTaskIds, boolean deferClose) throws CommandProcessorResponse {
    .....
	tree = ParseUtils.parse(command, ctx); // 解析器，生成AST树
	.....
	sem.analyze(tree, ctx); // 编译器和优化器
    .....
}
```



### AST树是怎么生成的

参见代码，在ParseUtils类中

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



解析驱动类的代码

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
	// 词法语法解析对象
    HiveLexerX lexer = new HiveLexerX(new ANTLRNoCaseStringStream(command));
    // 得到Token
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
      // 要return的结果
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
	// 得到树
    ASTNode tree = (ASTNode) r.getTree();
    tree.setUnknownTokenBoundaries();
    return tree;
  }
```



Hive用到了Antlr框架解析词法，使用了四个.g结尾的配置文件，把Hive所有的关键字做解析，与Antlr相匹配，这里不过多介绍

所以Hive解析流程如下：

1. 根据定义好的文件，对词法语法做解析
2. 解析后生成的Token对象，生成最后的抽象语法树



### 优化器过程

1. 逻辑优化

   Hive会把优化配置为True的优化项放在一个List里面，之后一个个遍历，每个优化选项都有一个类来实现

2. 物理优化

   分成三种优化，MR优化，Tez优化，Spark优化

   其中MR优化的代码是空的

   因为前面编译过程中已经优化过了



### 执行器

在execute方法中实现，与runInternal方法平级

调用launchTsk方法实现功能

```java
 private TaskRunner launchTask(Task<? extends Serializable> tsk, String queryId, boolean noName,
      String jobname, int jobs, DriverContext cxt) throws HiveException {
    if (SessionState.get() != null) {
      SessionState.get().getHiveHistory().startTask(queryId, tsk, tsk.getClass().getName());
    }
    if (tsk.isMapRedTask() && !(tsk instanceof ConditionalTask)) {
      if (noName) {
        conf.set(MRJobConfig.JOB_NAME, jobname + " (" + tsk.getId() + ")");
      }
      conf.set(DagUtils.MAPREDUCE_WORKFLOW_NODE_NAME, tsk.getId());
      Utilities.setWorkflowAdjacencies(conf, plan);
      cxt.incCurJobNo(1);
      console.printInfo("Launching Job " + cxt.getCurJobNo() + " out of " + jobs);
    }
    tsk.initialize(queryState, plan, cxt, ctx.getOpContext());
    TaskRunner tskRun = new TaskRunner(tsk);

     // 添加一个运行的任务
    cxt.launching(tskRun);
    // Launch Task
    if (HiveConf.getBoolVar(conf, HiveConf.ConfVars.EXECPARALLEL) && tsk.canExecuteInParallel()) {
      // Launch it in the parallel mode, as a separate thread only for MR tasks
      if (LOG.isInfoEnabled()){
        LOG.info("Starting task [" + tsk + "] in parallel");
      }
      // 真正运行任务的方法，并行执行
      tskRun.start();
    } else {
      if (LOG.isInfoEnabled()){
        LOG.info("Starting task [" + tsk + "] in serial mode");
      }
      // 真正运行任务的方法，单线程执行，只能用于MR任务
      tskRun.runSequential();
    }
    return tskRun;
  }
```

最后调用到了ExecDriver这个类，这个类中设置了MR任务 Job客户端执行配置

```java
 @Override
  public int execute(DriverContext driverContext) {

    IOPrepareCache ioPrepareCache = IOPrepareCache.get();
    ioPrepareCache.clear();

    boolean success = true;

    Context ctx = driverContext.getCtx();
    boolean ctxCreated = false;
    Path emptyScratchDir;
    JobClient jc = null;

    if (driverContext.isShutdown()) {
      LOG.warn("Task was cancelled");
      return 5;
    }

    MapWork mWork = work.getMapWork();
    ReduceWork rWork = work.getReduceWork();

    try {
      if (ctx == null) {
        ctx = new Context(job);
        ctxCreated = true;
      }

      emptyScratchDir = ctx.getMRTmpPath();
      FileSystem fs = emptyScratchDir.getFileSystem(job);
      fs.mkdirs(emptyScratchDir);
    } catch (IOException e) {
      e.printStackTrace();
      console.printError("Error launching map-reduce job", "\n"
          + org.apache.hadoop.util.StringUtils.stringifyException(e));
      return 5;
    }

    HiveFileFormatUtils.prepareJobOutput(job);
    //See the javadoc on HiveOutputFormatImpl and HadoopShims.prepareJobOutput()
    job.setOutputFormat(HiveOutputFormatImpl.class);

    job.setMapRunnerClass(ExecMapRunner.class);
    job.setMapperClass(ExecMapper.class);

    job.setMapOutputKeyClass(HiveKey.class);
    job.setMapOutputValueClass(BytesWritable.class);

    try {
      String partitioner = HiveConf.getVar(job, ConfVars.HIVEPARTITIONER);
      job.setPartitionerClass(JavaUtils.loadClass(partitioner));
    } catch (ClassNotFoundException e) {
      throw new RuntimeException(e.getMessage(), e);
    }

    propagateSplitSettings(job, mWork);

    job.setNumReduceTasks(rWork != null ? rWork.getNumReduceTasks().intValue() : 0);
    job.setReducerClass(ExecReducer.class);

    // set input format information if necessary
    setInputAttributes(job);

    // Turn on speculative execution for reducers
    boolean useSpeculativeExecReducers = HiveConf.getBoolVar(job,
        HiveConf.ConfVars.HIVESPECULATIVEEXECREDUCERS);
    job.setBoolean(MRJobConfig.REDUCE_SPECULATIVE, useSpeculativeExecReducers);

    String inpFormat = HiveConf.getVar(job, HiveConf.ConfVars.HIVEINPUTFORMAT);

    if (mWork.isUseBucketizedHiveInputFormat()) {
      inpFormat = BucketizedHiveInputFormat.class.getName();
    }

    LOG.info("Using " + inpFormat);

    try {
      job.setInputFormat(JavaUtils.loadClass(inpFormat));
    } catch (ClassNotFoundException e) {
      throw new RuntimeException(e.getMessage(), e);
    }

    // No-Op - we don't really write anything here ..
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(Text.class);

    int returnVal = 0;
    boolean noName = StringUtils.isEmpty(job.get(MRJobConfig.JOB_NAME));

    if (noName) {
      // This is for a special case to ensure unit tests pass
      job.set(MRJobConfig.JOB_NAME, "JOB" + Utilities.randGen.nextInt());
    }

    try{
      MapredLocalWork localwork = mWork.getMapRedLocalWork();
      if (localwork != null && localwork.hasStagedAlias()) {
        if (!ShimLoader.getHadoopShims().isLocalMode(job)) {
          Path localPath = localwork.getTmpPath();
          Path hdfsPath = mWork.getTmpHDFSPath();

          FileSystem hdfs = hdfsPath.getFileSystem(job);
          FileSystem localFS = localPath.getFileSystem(job);
          FileStatus[] hashtableFiles = localFS.listStatus(localPath);
          int fileNumber = hashtableFiles.length;
          String[] fileNames = new String[fileNumber];

          for ( int i = 0; i < fileNumber; i++){
            fileNames[i] = hashtableFiles[i].getPath().getName();
          }

          //package and compress all the hashtable files to an archive file
          String stageId = this.getId();
          String archiveFileName = Utilities.generateTarFileName(stageId);
          localwork.setStageID(stageId);

          CompressionUtils.tar(localPath.toUri().getPath(), fileNames,archiveFileName);
          Path archivePath = Utilities.generateTarPath(localPath, stageId);
          LOG.info("Archive "+ hashtableFiles.length+" hash table files to " + archivePath);

          //upload archive file to hdfs
          Path hdfsFilePath =Utilities.generateTarPath(hdfsPath, stageId);
          short replication = (short) job.getInt("mapred.submit.replication", 10);
          hdfs.copyFromLocalFile(archivePath, hdfsFilePath);
          hdfs.setReplication(hdfsFilePath, replication);
          LOG.info("Upload 1 archive file  from" + archivePath + " to: " + hdfsFilePath);

          //add the archive file to distributed cache
          DistributedCache.createSymlink(job);
          DistributedCache.addCacheArchive(hdfsFilePath.toUri(), job);
          LOG.info("Add 1 archive file to distributed cache. Archive file: " + hdfsFilePath.toUri());
        }
      }
      work.configureJobConf(job);
      List<Path> inputPaths = Utilities.getInputPaths(job, mWork, emptyScratchDir, ctx, false);
      Utilities.setInputPaths(job, inputPaths);

      Utilities.setMapRedWork(job, work, ctx.getMRTmpPath());

      if (mWork.getSamplingType() > 0 && rWork != null && job.getNumReduceTasks() > 1) {
        try {
          handleSampling(ctx, mWork, job);
          job.setPartitionerClass(HiveTotalOrderPartitioner.class);
        } catch (IllegalStateException e) {
          console.printInfo("Not enough sampling data.. Rolling back to single reducer task");
          rWork.setNumReduceTasks(1);
          job.setNumReduceTasks(1);
        } catch (Exception e) {
          LOG.error("Sampling error", e);
          console.printError(e.toString(),
              "\n" + org.apache.hadoop.util.StringUtils.stringifyException(e));
          rWork.setNumReduceTasks(1);
          job.setNumReduceTasks(1);
        }
      }

      jc = new JobClient(job);
      // make this client wait if job tracker is not behaving well.
      Throttle.checkJobTracker(job, LOG);

      if (mWork.isGatheringStats() || (rWork != null && rWork.isGatheringStats())) {
        // initialize stats publishing table
        StatsPublisher statsPublisher;
        StatsFactory factory = StatsFactory.newFactory(job);
        if (factory != null) {
          statsPublisher = factory.getStatsPublisher();
          List<String> statsTmpDir = Utilities.getStatsTmpDirs(mWork, job);
          if (rWork != null) {
            statsTmpDir.addAll(Utilities.getStatsTmpDirs(rWork, job));
          }
          StatsCollectionContext sc = new StatsCollectionContext(job);
          sc.setStatsTmpDirs(statsTmpDir);
          if (!statsPublisher.init(sc)) { // creating stats table if not exists
            if (HiveConf.getBoolVar(job, HiveConf.ConfVars.HIVE_STATS_RELIABLE)) {
              throw
                new HiveException(ErrorMsg.STATSPUBLISHER_INITIALIZATION_ERROR.getErrorCodedMsg());
            }
          }
        }
      }

      Utilities.createTmpDirs(job, mWork);
      Utilities.createTmpDirs(job, rWork);

      SessionState ss = SessionState.get();
      // TODO: why is there a TezSession in MR ExecDriver?
      if (ss != null && HiveConf.getVar(job, ConfVars.HIVE_EXECUTION_ENGINE).equals("tez")) {
        // TODO: this is the only place that uses keepTmpDir. Why?
        TezSessionPoolManager.closeIfNotDefault(ss.getTezSession(), true);
      }

      HiveConfUtil.updateJobCredentialProviders(job);
      // Finally SUBMIT the JOB!
      if (driverContext.isShutdown()) {
        LOG.warn("Task was cancelled");
        return 5;
      }

      rj = jc.submitJob(job);

      if (driverContext.isShutdown()) {
        LOG.warn("Task was cancelled");
        killJob();
        return 5;
      }

      this.jobID = rj.getJobID();
      updateStatusInQueryDisplay();
      returnVal = jobExecHelper.progress(rj, jc, ctx);
      success = (returnVal == 0);
    } catch (Exception e) {
      e.printStackTrace();
      setException(e);
      String mesg = " with exception '" + Utilities.getNameMessage(e) + "'";
      if (rj != null) {
        mesg = "Ended Job = " + rj.getJobID() + mesg;
      } else {
        mesg = "Job Submission failed" + mesg;
      }

      // Has to use full name to make sure it does not conflict with
      // org.apache.commons.lang.StringUtils
      console.printError(mesg, "\n" + org.apache.hadoop.util.StringUtils.stringifyException(e));

      success = false;
      returnVal = 1;
    } finally {
      Utilities.clearWork(job);
      try {
        if (ctxCreated) {
          ctx.clear();
        }

        if (rj != null) {
          if (returnVal != 0) {
            killJob();
          }
          jobID = rj.getID().toString();
        }
        if (jc!=null) {
          jc.close();
        }
      } catch (Exception e) {
	LOG.warn("Failed while cleaning up ", e);
      } finally {
	HadoopJobExecHelper.runningJobs.remove(rj);
      }
    }

    // get the list of Dynamic partition paths
    try {
      if (rj != null) {
        if (mWork.getAliasToWork() != null) {
          for (Operator<? extends OperatorDesc> op : mWork.getAliasToWork().values()) {
            op.jobClose(job, success);
          }
        }
        if (rWork != null) {
          rWork.getReducer().jobClose(job, success);
        }
      }
    } catch (Exception e) {
      // jobClose needs to execute successfully otherwise fail task
      if (success) {
        setException(e);
        success = false;
        returnVal = 3;
        String mesg = "Job Commit failed with exception '" + Utilities.getNameMessage(e) + "'";
        console.printError(mesg, "\n" + org.apache.hadoop.util.StringUtils.stringifyException(e));
      }
    }

    return (returnVal);
  }
```







## 总结

流程如下

1. hive提交任务

2. CliDriver

   ```
   解析客户端" -e  -f"参数
   
   定义标准输入输出流
   
   按照分号切分SQL语句
   ```

3. Driver

   ```java
   HQL转AST -> ParseUtils.parse(command, ctx);
   
   AST转为TaskTree -> sem.analyze(tree, ctx);
   
   提交任务执行 -> TaskRunner.runSequential()
   ```



### HQL转AST

在PaserDriver类中

```
HQL语句转为Token

Token解析生成AST
```



### AST转为TaskTree

在SemanticAnalyzer类中

```
AST转为QueryBlock

QueryBlock转为OperatorTree

OperatorTree逻辑优化

生成TaskTree

TaskTree物理优化
```



### 提交任务执行

在ExecDriver类中

```
获取MR临时工作目录

定义Partitioner

定义Mapper和Reducer

实例化Job
```







Hive Debug模式（待验证）

hive server 2 服务端代码远程debug调试方式：Hive 1.2.1支持

1. idea 添加debug 远程调试，在Config中添加Remaote模式

2. 将idea 远程debug 参数添加到 vi hadoop_conf/hadoop-env.sh 中 HADOOP_CLIENT_OPTS 配置项中，如：

```shell
export HADOOP_CLIENT_OPTS="-Xmx5g -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9001 $HADOOP_CLIENT_OPTS"
```



beeline client 客户端代码远程debug调试方式：

1. 修改 vi hadoop_conf/hadoop-env.sh 中 HADOOP_CLIENT_OPTS 配置项，添加 -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005 参数，端口号可自己配置，如下：

```shell
export HADOOP_CLIENT_OPTS="-Xmx5g -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005 $HADOOP_CLIENT_OPTS"
```

2. idea中添加debug远程调试：


