---
title: "Trino Practice"
author: "爱吃芒果"
description:
date: "2025-07-09T22:27:56+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
categories:
---



直接从maven仓库下载对应版本的`presto-cli`作为客户端，记得下载`presto-cli-350-executable.jar`版本，否则可能不能直接运行

## 查询的接收、解析和提交

### 接收SQL查询请求

客户端sql请求作为HTTP POST请求`v1/statement` 被`io.prestosql.dispatcher.QueuedStatementResource#postStatement`处理，生成QueryId，然后将查询加入map中，返回响应报文给客户端。

```java
public Response postStatement(
        String statement,
        @Context HttpServletRequest servletRequest,
        @Context HttpHeaders httpHeaders,
        @Context UriInfo uriInfo)
{
    if (isNullOrEmpty(statement)) {
        throw badRequest(BAD_REQUEST, "SQL statement is empty");
    }

    String remoteAddress = servletRequest.getRemoteAddr();
    Optional<Identity> identity = Optional.ofNullable((Identity) servletRequest.getAttribute(AUTHENTICATED_IDENTITY));
    MultivaluedMap<String, String> headers = httpHeaders.getRequestHeaders();
		// 创建请求的session context
    SessionContext sessionContext = new HttpRequestSessionContext(headers, remoteAddress, identity, groupProvider);
  	// 创建Query对象，构造QueryId
    Query query = new Query(statement, sessionContext, dispatchManager);
    queries.put(query.getQueryId(), query);

    // let authentication filter know that identity lifecycle has been handed off
    servletRequest.setAttribute(AUTHENTICATED_IDENTITY, null);

    return createQueryResultsResponse(query.getQueryResults(query.getLastToken(), uriInfo), compressionEnabled);
}
```

之后客户端会发起第二次HTTP请求`queued/{queryId}/{slug}/{token}`，Presto此时才会真正将查询提交到DispatchManager::createQueryInternal来执行。



```java
// TODO: 这段异步处理的逻辑后面再看
@ResourceSecurity(PUBLIC)
@GET
@Path("queued/{queryId}/{slug}/{token}")
@Produces(APPLICATION_JSON)
public void getStatus(
        @PathParam("queryId") QueryId queryId,
        @PathParam("slug") String slug,
        @PathParam("token") long token,
        @QueryParam("maxWait") Duration maxWait,
        @Context UriInfo uriInfo,
        @Suspended AsyncResponse asyncResponse)
{
    Query query = getQuery(queryId, slug, token);

    // wait for query to be dispatched, up to the wait timeout
    ListenableFuture<?> futureStateChange = addTimeout(
            query.waitForDispatched(),
            () -> null,
            WAIT_ORDERING.min(MAX_WAIT_TIME, maxWait),
            timeoutExecutor);

    // when state changes, fetch the next result
    ListenableFuture<QueryResults> queryResultsFuture = Futures.transform(
            futureStateChange,
            ignored -> query.getQueryResults(token, uriInfo),
            responseExecutor);

    // transform to Response
    ListenableFuture<Response> response = Futures.transform(
            queryResultsFuture,
            queryResults -> createQueryResultsResponse(queryResults, compressionEnabled),
            directExecutor());
    bindAsyncResponse(asyncResponse, response, responseExecutor);
}
```



```java
/**
 * Creates and registers a dispatch query with the query tracker.  This method will never fail to register a query with the query
 * tracker.  If an error occurs while creating a dispatch query, a failed dispatch will be created and registered.
 */
private <C> void createQueryInternal(QueryId queryId, Slug slug, SessionContext sessionContext, String query, ResourceGroupManager<C> resourceGroupManager)
{
    Session session = null;
    PreparedQuery preparedQuery = null;
    try {
      	// 检查查询长度是否超过限制，超过报错
        if (query.length() > maxQueryLength) {
            int queryLength = query.length();
            query = query.substring(0, maxQueryLength);
            throw new PrestoException(QUERY_TEXT_TOO_LARGE, format("Query text length (%s) exceeds the maximum length (%s)", queryLength, maxQueryLength));
        }

        // 创建查询会话
        session = sessionSupplier.createSession(queryId, sessionContext);

        // 检查查询执行权限
        accessControl.checkCanExecuteQuery(sessionContext.getIdentity());

        // sql解析，生成抽象语法树
        preparedQuery = queryPreparer.prepareQuery(session, query);

        // select resource group
        Optional<String> queryType = getQueryType(preparedQuery.getStatement().getClass()).map(Enum::name);
        SelectionContext<C> selectionContext = resourceGroupManager.selectGroup(new SelectionCriteria(
                sessionContext.getIdentity().getPrincipal().isPresent(),
                sessionContext.getIdentity().getUser(),
                sessionContext.getIdentity().getGroups(),
                Optional.ofNullable(sessionContext.getSource()),
                sessionContext.getClientTags(),
                sessionContext.getResourceEstimates(),
                queryType));

        // apply system default session properties (does not override user set properties)
        session = sessionPropertyDefaults.newSessionWithDefaultProperties(session, queryType, selectionContext.getResourceGroupId());

        // mark existing transaction as active
        transactionManager.activateTransaction(session, isTransactionControlStatement(preparedQuery.getStatement()), accessControl);

        DispatchQuery dispatchQuery = dispatchQueryFactory.createDispatchQuery(
                session,
                query,
                preparedQuery,
                slug,
                selectionContext.getResourceGroupId());

        boolean queryAdded = queryCreated(dispatchQuery);
        if (queryAdded && !dispatchQuery.isDone()) {
            try {
                resourceGroupManager.submit(dispatchQuery, selectionContext, dispatchExecutor);
            }
            catch (Throwable e) {
                // dispatch query has already been registered, so just fail it directly
                dispatchQuery.fail(e);
            }
        }
    }
    catch (Throwable throwable) {
        // creation must never fail, so register a failed query in this case
        if (session == null) {
            session = Session.builder(new SessionPropertyManager())
                    .setQueryId(queryId)
                    .setIdentity(sessionContext.getIdentity())
                    .setSource(sessionContext.getSource())
                    .build();
        }
        Optional<String> preparedSql = Optional.ofNullable(preparedQuery).flatMap(PreparedQuery::getPrepareSql);
        DispatchQuery failedDispatchQuery = failedDispatchQueryFactory.createFailedDispatchQuery(session, query, preparedSql, Optional.empty(), throwable);
        queryCreated(failedDispatchQuery);
    }
}
```

`sessionSpplier.createSession`创建了查询会话，设置了各种状态和信息。

```java
public Session createSession(QueryId queryId, SessionContext context)
{
    Identity identity = context.getIdentity();
    accessControl.checkCanSetUser(identity.getPrincipal(), identity.getUser());

    // authenticated identity is not present for HTTP or if authentication is not setup
    context.getAuthenticatedIdentity().ifPresent(authenticatedIdentity -> {
        // only check impersonation if authenticated user is not the same as the explicitly set user
        if (!authenticatedIdentity.getUser().equals(identity.getUser())) {
            accessControl.checkCanImpersonateUser(authenticatedIdentity, identity.getUser());
        }
    });

    SessionBuilder sessionBuilder = Session.builder(sessionPropertyManager)
            .setQueryId(queryId)
            .setIdentity(identity)
            .setSource(context.getSource())
            .setPath(new SqlPath(path))
            .setRemoteUserAddress(context.getRemoteUserAddress())
            .setUserAgent(context.getUserAgent())
            .setClientInfo(context.getClientInfo())
            .setClientTags(context.getClientTags())
            .setClientCapabilities(context.getClientCapabilities())
            .setTraceToken(context.getTraceToken())
            .setResourceEstimates(context.getResourceEstimates());

    defaultCatalog.ifPresent(sessionBuilder::setCatalog);
    defaultSchema.ifPresent(sessionBuilder::setSchema);

    if (context.getCatalog() != null) {
        sessionBuilder.setCatalog(context.getCatalog());
    }

    if (context.getSchema() != null) {
        sessionBuilder.setSchema(context.getSchema());
    }

    if (context.getPath() != null) {
        sessionBuilder.setPath(new SqlPath(Optional.of(context.getPath())));
    }

    if (forcedSessionTimeZone.isPresent()) {
        sessionBuilder.setTimeZoneKey(forcedSessionTimeZone.get());
    }
    else if (context.getTimeZoneId() != null) {
        sessionBuilder.setTimeZoneKey(getTimeZoneKey(context.getTimeZoneId()));
    }

    if (context.getLanguage() != null) {
        sessionBuilder.setLocale(Locale.forLanguageTag(context.getLanguage()));
    }

    for (Entry<String, String> entry : context.getSystemProperties().entrySet()) {
        sessionBuilder.setSystemProperty(entry.getKey(), entry.getValue());
    }
    for (Entry<String, Map<String, String>> catalogProperties : context.getCatalogSessionProperties().entrySet()) {
        String catalog = catalogProperties.getKey();
        for (Entry<String, String> entry : catalogProperties.getValue().entrySet()) {
            sessionBuilder.setCatalogSessionProperty(catalog, entry.getKey(), entry.getValue());
        }
    }

    for (Entry<String, String> preparedStatement : context.getPreparedStatements().entrySet()) {
        sessionBuilder.addPreparedStatement(preparedStatement.getKey(), preparedStatement.getValue());
    }

    if (context.supportClientTransaction()) {
        sessionBuilder.setClientTransactionSupport();
    }

    Session session = sessionBuilder.build();
    if (context.getTransactionId().isPresent()) {
        session = session.beginTransactionId(context.getTransactionId().get(), transactionManager, accessControl);
    }

    return session;
}
```

### 词法与语法分析并生成抽象语法树

SqlParser使用Antlr4作为解析工具，通过词法和语法解析通过SQL字符串生成抽象语法树。抽象语法树是用一种树形结构表示SQL想要表述的语义，将一段SQL字符串结构化，以支持SQL执行引擎根据抽象语法树生成SQL执行计划。在Presto中，Node表示树的节点的抽象，根据语义不同，SQL抽象语法树中有多种不同类型的节点，都继承了Node节点。

抽象语法树在我看来只是将字符串转成结构更加严谨的树状结构，和sql的文本表述类似，除了将文本转成树状结构外，没有做额外的操作，比如FunctionCall并没有真的生成调用的函数，而只是描述。

```java
public class FunctionCall
        extends Expression
{
  	// 函数名
    private final QualifiedName name;
    private final Optional<Window> window;
    private final Optional<Expression> filter;
    private final Optional<OrderBy> orderBy;
    private final boolean distinct;
    private final Optional<NullTreatment> nullTreatment;
  	// 函数参数
    private final List<Expression> arguments;
```

### 创建并提交QueryExecution

`dispatchQueryFactory.createDispatchQuery`方法在执行中，会根据Statement的类型生成QueryExecution，对于DML操作(Data Manipulatioin language)生成SqlQueryExecution，对于DDL (Data Definition language)生成DataDefinitionExecution，然后进一步包装成LocalDispatchQuery，提交到resourceGroupManager等待运行。

```java
public void start()
{
    try (SetThreadName ignored = new SetThreadName("Query-%s", stateMachine.getQueryId())) {
        try {
            if (!stateMachine.transitionToPlanning()) {
                // query already started or finished
                return;
            }
						// 生成逻辑执行计划，此方法进一步调用了doPlanQuery
            PlanRoot plan = planQuery();
            // DynamicFilterService needs plan for query to be registered.
            // Query should be registered before dynamic filter suppliers are requested in distribution planning.
            registerDynamicFilteringQuery(plan);
          	// 生成数据源连接器的ConnectorSplitSource，创建SqlStageExecution(Stage)，指定StageScheduler
            planDistribution(plan);

            if (!stateMachine.transitionToStarting()) {
                // query already started or finished
                return;
            }

            // if query is not finished, start the scheduler, otherwise cancel it
            SqlQueryScheduler scheduler = queryScheduler.get();
						// 查询执行阶段的调度，根据执行计划将任务调度到Presto查询执行节点上
            if (!stateMachine.isDone()) {
                scheduler.start();
            }
        }
        catch (Throwable e) {
            fail(e);
            throwIfInstanceOf(e, Error.class);
        }
    }
}
```

## 执行计划的生成和优化

### 语义分析、生成执行计划

```java
private PlanRoot doPlanQuery()
{
    // planNodeId是一个从0递增的int值
    PlanNodeIdAllocator idAllocator = new PlanNodeIdAllocator();
    LogicalPlanner logicalPlanner = new LogicalPlanner(stateMachine.getSession(),
            planOptimizers,
            idAllocator,
            metadata,
            typeOperators,
            new TypeAnalyzer(sqlParser, metadata),
            statsCalculator,
            costCalculator,
            stateMachine.getWarningCollector());
  	// 语义分析(Analysis)，生成执行计划
  	// 优化执行计划，生成优化后的执行计划
    Plan plan = logicalPlanner.plan(analysis);
    queryPlan.set(plan);

    // 将逻辑执行计划分成多棵子树
    SubPlan fragmentedPlan = planFragmenter.createSubPlans(stateMachine.getSession(), plan, false, stateMachine.getWarningCollector());

    // extract inputs
    List<Input> inputs = new InputExtractor(metadata, stateMachine.getSession()).extractInputs(fragmentedPlan);
    stateMachine.setInputs(inputs);

    stateMachine.setOutput(analysis.getTarget());

    boolean explainAnalyze = analysis.getStatement() instanceof Explain && ((Explain) analysis.getStatement()).isAnalyze();
    return new PlanRoot(fragmentedPlan, !explainAnalyze);
}
```

LogicalPlanner.plan的职责如下：

- 语义分析：遍历SQL抽象语法树，将抽象语法树中表达的含义拆解为多个map结构，以便后续生成执行计划时，不再频繁遍历SQL抽象语法树。同时获取了表和字段的元数据，生成了对应的ConnectorTableHandle、ColumnHandle等与数据源连接器相关的对象实例，为了之后拿来即用打下基础。在此过程中生成的所有对象，都维护在一个实例化的Analysis对象，Analysis对象可以理解为一个Context对象
- 生成执行计划：生成以PlanNode为节点的逻辑执行计划，它也是类似于抽象语法树的树型结构，树节点和根的类型都是PlanNode。
- 优化执行计划，生成优化后的执行计划：用预定义的几百个优化器迭代优化之前生成的PlanNode树，并返回优化后的PlanNode树

```java
public Plan plan(Analysis analysis, Stage stage, boolean collectPlanStatistics)
{
  	// 语义分析，生成执行计划
    PlanNode root = planStatement(analysis, analysis.getStatement());

    planSanityChecker.validateIntermediatePlan(root, session, metadata, typeOperators, typeAnalyzer, symbolAllocator.getTypes(), warningCollector);

  	// 优化执行计划，生成优化后的执行计划
    if (stage.ordinal() >= OPTIMIZED.ordinal()) {
        for (PlanOptimizer optimizer : planOptimizers) {
            root = optimizer.optimize(root, session, symbolAllocator.getTypes(), symbolAllocator, idAllocator, warningCollector);
            requireNonNull(root, format("%s returned a null plan", optimizer.getClass().getName()));
        }
    }

    if (stage.ordinal() >= OPTIMIZED_AND_VALIDATED.ordinal()) {
        // make sure we produce a valid plan after optimizations run. This is mainly to catch programming errors
        planSanityChecker.validateFinalPlan(root, session, metadata, typeOperators, typeAnalyzer, symbolAllocator.getTypes(), warningCollector);
    }

    TypeProvider types = symbolAllocator.getTypes();

    StatsAndCosts statsAndCosts = StatsAndCosts.empty();
    if (collectPlanStatistics) {
        StatsProvider statsProvider = new CachingStatsProvider(statsCalculator, session, types);
        CostProvider costProvider = new CachingCostProvider(costCalculator, statsProvider, Optional.empty(), session, types);
        statsAndCosts = StatsAndCosts.create(root, statsProvider, costProvider);
    }
    return new Plan(root, types, statsAndCosts);
}
```

### 将逻辑执行计划树拆分为多棵子树

将逻辑执行计划拆分为多棵子树并生成subPlan的逻辑，这个过程用SimplePlanRewriter的实现类Fragmenter层层遍历上一步生成的PlanNode树，将其中的ExchangeNode[scope=REMOTE]替换为RemoteSourceNode，并且断开它与叶子节点的连接，这样一个PlanNode树就被划分成了两个PlanNode树，一个父树（对应创建一个PlanFragment）和一个子树（又称为SubPlan，对应创建一个PlanFragment）。在查询执行的数据流转中，子树是父树的数据产出上游。

```java
public class SubPlan
{
    private final PlanFragment fragment;
    private final List<SubPlan> children;
}
```

## 执行计划的调度

### 创建SqlStageExecution

在Presto的执行模型中，SQL的执行被划分为如下几个层次：

- 查询：用户提交一个SQL，触发Presto的一次查询，在代码中对应一个QueryInfo。每个查询都有一个字符串形式的QueryId
- 查询执行阶段：Presto生成查询的执行计划时，根据是否需要做跨查询执行节点的数据交换来划分PlanFragment。调度执行计划时，每个PlanFragment对应一个查询执行阶段，在代码中对应一个StageInfo，其中有StageId，StageId的形式为QueryId + 从0自增id。查询执行阶段之间有数据依赖关系，即不能并行执行，存在执行上的顺序关系，需要注意的是，StageId越小，这个查询执行阶段的执行顺序越靠后。Presto的查询执行阶段类似于Spark的查询执行阶段的概念，他们的不同是Presto不像Spark批式处理那样，需要前面的查询执行阶段执行完再执行后面的查询执行阶段，Presto采用的是流水线（Pipeline）处理机制。
- 任务（Task）：任务是Presto分布式任务的执行单元，每个查询执行阶段可以有多个任务，这些任务可以并行执行，同一个查询执行阶段中的所有任务的执行逻辑完全相同。一个查询执行阶段的任务个数就是此查询执行阶段的并发度。在Presto的任务调度代码中，可以看到任务的个数是根据查询执行阶段的数据分布方式（Source，Fixed，Single）以及查询执行节点的个数来决定的。

```java
private void planDistribution(PlanRoot plan)
{
  	// 遍历执行计划PlanNode树，找到所有的TableScanNode（也就是连接器对应的PlanNode），获取到他们的ConnectorSplit
    DistributedExecutionPlanner distributedPlanner = new DistributedExecutionPlanner(splitManager, metadata, dynamicFilterService);
    StageExecutionPlan outputStageExecutionPlan = distributedPlanner.plan(plan.getRoot(), stateMachine.getSession());

    // ensure split sources are closed
    stateMachine.addStateChangeListener(state -> {
        if (state.isDone()) {
            closeSplitSources(outputStageExecutionPlan);
        }
    });

    // 如果查询被取消了，跳过创建调度器
    if (stateMachine.isDone()) {
        return;
    }

    // record output field
    stateMachine.setColumns(outputStageExecutionPlan.getFieldNames(), outputStageExecutionPlan.getFragment().getTypes());

  	// 创建最后一个查询执行阶段的Outputbuffer，这个OutputBuffer用于给Presto SQL客户端输出查询的最终计算结果
    PartitioningHandle partitioningHandle = plan.getRoot().getFragment().getPartitioningScheme().getPartitioning().getHandle();
    OutputBuffers rootOutputBuffers = createInitialEmptyOutputBuffers(partitioningHandle)
            .withBuffer(OUTPUT_BUFFER_ID, BROADCAST_PARTITION_ID)
            .withNoMoreBufferIds();

    // 创建SqlStageExecution，并将其封装在SqlQueryScheduler里面返回
  	// 这里只是创建Stage，但是不会去调度执行它
    SqlQueryScheduler scheduler = createSqlQueryScheduler(
            stateMachine,
            outputStageExecutionPlan,
            nodePartitioningManager,
            nodeScheduler,
            remoteTaskFactory,
            stateMachine.getSession(),
            plan.isSummarizeTaskInfos(),
            scheduleSplitBatchSize,
            queryExecutor,
            schedulerExecutor,
            failureDetector,
            rootOutputBuffers,
            nodeTaskMap,
            executionPolicy,
            schedulerStats,
            dynamicFilterService);

    queryScheduler.set(scheduler);

    // if query was canceled during scheduler creation, abort the scheduler
    // directly since the callback may have already fired
    if (stateMachine.isDone()) {
        scheduler.abort();
        queryScheduler.set(null);
    }
}
```

从数据源连接器中获取到所有的分片。分片是Presto中分块组织数据的方式，Presto连接器会将待处理的所有数据划分为若干分片让Presto读取，而这些分片也会被安排到多个Presto查询执行节点上来处理以实现分布式高性能计算。分布式OLAP引擎几乎全都有分片的抽象设计，例如Spark、Flink等。

```java
public class ConnectorAwareSplitSource
        implements SplitSource
{
    private final CatalogName catalogName;
    private final ConnectorSplitSource source;
}
public class FixedSplitSource
        implements ConnectorSplitSource
{
    private final List<ConnectorSplit> splits;
    private int offset;
}
```



```java
public class TpcdsSplitManager
        implements ConnectorSplitManager
  public ConnectorSplitSource getSplits(ConnectorTransactionHandle transaction, ConnectorSession session, ConnectorTableLayoutHandle layout, SplitSchedulingStrategy splitSchedulingStrategy)
  {
      Set<Node> nodes = nodeManager.getRequiredWorkerNodes();
      checkState(!nodes.isEmpty(), "No TPCDS nodes available");

      int totalParts = nodes.size() * splitsPerNode;
      int partNumber = 0;

      // Split the data using split and skew by the number of nodes available.
      ImmutableList.Builder<ConnectorSplit> splits = ImmutableList.builder();
      for (Node node : nodes) {
          for (int i = 0; i < splitsPerNode; i++) {
              splits.add(new TpcdsSplit(partNumber, totalParts, ImmutableList.of(node.getHostAndPort()), noSexism));
              partNumber++;
          }
      }
      return new FixedSplitSource(splits.build());
  }
```



```java
public interface ConnectorSplit
{
  	// 这个信息定义了分片是否可以非分片所在的节点访问到，对于计算存储分离的情况，这里需要返回true
    boolean isRemotelyAccessible();
		// 这个信息定义了分片可以从哪些节点访问（这些节点，并不需要是Presto集群的节点，例如对于计算存储分离的情况，大概率Presto的节点与数据源分片所在的节点不是相同的
    List<HostAddress> getAddresses();
		// 这里允许连接器设置一些自己的信息
    Object getInfo();
}
```

注意，这一步只是生成数据源分片，既不会将分片安排到某个Presto查询执行节点上，也不会真正使用分片读取连接器的数据。

```java
private SqlQueryScheduler(
        QueryStateMachine queryStateMachine,
        StageExecutionPlan plan,
        NodePartitioningManager nodePartitioningManager,
        NodeScheduler nodeScheduler,
        RemoteTaskFactory remoteTaskFactory,
        Session session,
        boolean summarizeTaskInfo,
        int splitBatchSize,
        ExecutorService queryExecutor,
        ScheduledExecutorService schedulerExecutor,
        FailureDetector failureDetector,
        OutputBuffers rootOutputBuffers,
        NodeTaskMap nodeTaskMap,
        ExecutionPolicy executionPolicy,
        SplitSchedulerStats schedulerStats,
        DynamicFilterService dynamicFilterService)
{
    this.queryStateMachine = requireNonNull(queryStateMachine, "queryStateMachine is null");
    this.executionPolicy = requireNonNull(executionPolicy, "schedulerPolicyFactory is null");
    this.schedulerStats = requireNonNull(schedulerStats, "schedulerStats is null");
    this.summarizeTaskInfo = summarizeTaskInfo;
    this.dynamicFilterService = requireNonNull(dynamicFilterService, "dynamicFilterService is null");

    // todo come up with a better way to build this, or eliminate this map
    ImmutableMap.Builder<StageId, StageScheduler> stageSchedulers = ImmutableMap.builder();
    ImmutableMap.Builder<StageId, StageLinkage> stageLinkages = ImmutableMap.builder();

    // Only fetch a distribution once per query to assure all stages see the same machine assignments
    Map<PartitioningHandle, NodePartitionMap> partitioningCache = new HashMap<>();

    OutputBufferId rootBufferId = Iterables.getOnlyElement(rootOutputBuffers.getBuffers().keySet());
    List<SqlStageExecution> stages = createStages(
            (fragmentId, tasks, noMoreExchangeLocations) -> updateQueryOutputLocations(queryStateMachine, rootBufferId, tasks, noMoreExchangeLocations),
            new AtomicInteger(),
            plan.withBucketToPartition(Optional.of(new int[1])),
            nodeScheduler,
            remoteTaskFactory,
            session,
            splitBatchSize,
            partitioningHandle -> partitioningCache.computeIfAbsent(partitioningHandle, handle -> nodePartitioningManager.getNodePartitioningMap(session, handle)),
            nodePartitioningManager,
            queryExecutor,
            schedulerExecutor,
            failureDetector,
            nodeTaskMap,
            stageSchedulers,
            stageLinkages);

    SqlStageExecution rootStage = stages.get(0);
    rootStage.setOutputBuffers(rootOutputBuffers);
    this.rootStageId = rootStage.getStageId();

    this.stages = stages.stream()
            .collect(toImmutableMap(SqlStageExecution::getStageId, identity()));

    this.stageSchedulers = stageSchedulers.build();
    this.stageLinkages = stageLinkages.build();

    this.executor = queryExecutor;
}
```

createSqlQueryScheduler会为执行计划的每一个PlanFragment创建一个SqlStageExecution。每个SqlStageExecution对应一个StageScheduler，不同数据分区类型（PartitioningHandle）的查询执行计划阶段对应不同的StageScheduler。后面在调度查询执行阶段的任务时，主要依赖的是这个StageScheduler的具体实现。

### 调度并分发执行计划

```java
private void schedule()
{
    try (SetThreadName ignored = new SetThreadName("Query-%s", queryStateMachine.getQueryId())) {
        Set<StageId> completedStages = new HashSet<>();
      	// 根据执行策略确定查询执行阶段的调度顺序和调度时机，默认是AllAtOnceExecutionPolicy
      	// 会按照查询执行阶段执行的上下游关系依次调度查询执行阶段，生成任务并全部分发给Presto查询执行节点
      	// 另一种策略是PhasedExeuctionPolicy
        ExecutionSchedule executionSchedule = executionPolicy.createExecutionSchedule(stages.values());
        while (!executionSchedule.isFinished()) {
            List<ListenableFuture<?>> blockedStages = new ArrayList<>();
            for (SqlStageExecution stage : executionSchedule.getStagesToSchedule()) {
                stage.beginScheduling();

                // 拿到与当前查询执行阶段对应的StageScheduler
              	// 绑定Presto查询执行节点与上游数据源分片的关系，创建任务并调度到Presto查询执行节点上
                ScheduleResult result = stageSchedulers.get(stage.getStageId())
                        .schedule();

                // modify parent and children based on the results of the scheduling
                if (result.isFinished()) {
                    stage.schedulingComplete();
                }
                else if (!result.getBlocked().isDone()) {
                    blockedStages.add(result.getBlocked());
                }
              	// 将上一步在当前查询执行阶段上刚创建的任务注册到下游查询执行阶段的soruceTask阶段里，
              	// 这样下游查询执行阶段的任务就知道他们要去哪些上游任务拉取数据
                stageLinkages.get(stage.getStageId())
                        .processScheduleResults(stage.getState(), result.getNewTasks());
                schedulerStats.getSplitsScheduledPerIteration().add(result.getSplitsScheduled());
                if (result.getBlockedReason().isPresent()) {
                    switch (result.getBlockedReason().get()) {
                        case WRITER_SCALING:
                            // no-op
                            break;
                        case WAITING_FOR_SOURCE:
                            schedulerStats.getWaitingForSource().update(1);
                            break;
                        case SPLIT_QUEUES_FULL:
                            schedulerStats.getSplitQueuesFull().update(1);
                            break;
                        case MIXED_SPLIT_QUEUES_FULL_AND_WAITING_FOR_SOURCE:
                        case NO_ACTIVE_DRIVER_GROUP:
                            break;
                        default:
                            throw new UnsupportedOperationException("Unknown blocked reason: " + result.getBlockedReason().get());
                    }
                }
            }

            // make sure to update stage linkage at least once per loop to catch async state changes (e.g., partial cancel)
            for (SqlStageExecution stage : stages.values()) {
                if (!completedStages.contains(stage.getStageId()) && stage.getState().isDone()) {
                    stageLinkages.get(stage.getStageId())
                            .processScheduleResults(stage.getState(), ImmutableSet.of());
                    completedStages.add(stage.getStageId());
                }
            }

            // wait for a state change and then schedule again
            if (!blockedStages.isEmpty()) {
                try (TimeStat.BlockTimer timer = schedulerStats.getSleepTime().time()) {
                    tryGetFutureValue(whenAnyComplete(blockedStages), 1, SECONDS);
                }
                for (ListenableFuture<?> blockedStage : blockedStages) {
                    blockedStage.cancel(true);
                }
            }
        }

        for (SqlStageExecution stage : stages.values()) {
            StageState state = stage.getState();
            if (state != SCHEDULED && state != RUNNING && state != FLUSHING && !state.isDone()) {
                throw new PrestoException(GENERIC_INTERNAL_ERROR, format("Scheduling is complete, but stage %s is in state %s", stage.getStageId(), state));
            }
        }
    }
    catch (Throwable t) {
        queryStateMachine.transitionToFailed(t);
        throw t;
    }
    finally {
        RuntimeException closeError = new RuntimeException();
        for (StageScheduler scheduler : stageSchedulers.values()) {
            try {
                scheduler.close();
            }
            catch (Throwable t) {
                queryStateMachine.transitionToFailed(t);
                // Self-suppression not permitted
                if (closeError != t) {
                    closeError.addSuppressed(t);
                }
            }
        }
        if (closeError.getSuppressed().length > 0) {
            throw closeError;
        }
    }
}
```

#### 创建当前查询执行阶段对应的StageScheduler

不同数据分区类型（PartitioningHandle）的PlanFragment的查询执行阶段对应不同类型的StageScheduler，后面在调度查询执行阶段时，主要依赖的是这样StageScheduler的实现。SqlQueryScheduler中通过Map<StageId, StageScheduler> stageSchedulers这样一个Map数据结构维护了当前查询所有查询执行阶段的StageScheduler。

```java
public interface StageScheduler
        extends Closeable
{
    /**
     * Schedules as much work as possible without blocking.
     * The schedule results is a hint to the query scheduler if and
     * when the stage scheduler should be invoked again.  It is
     * important to note that this is only a hint and the query
     * scheduler may call the schedule method at any time.
     */
    ScheduleResult schedule();

    @Override
    default void close() {}
}
```

到目前为止，StageScheduler有4个实现类，分别对应了4种不同的查询执行阶段调度方式，最常用到的是SourcePartitionedScheduler和FixedCountScheduler。

#### 调度分片和任务

StageScheduler的职责是绑定查询执行节点与上游数据源分片的关系，创建任务并调度到查询执行节点上。

对于stage使用数据源连接器，从存储系统拉取数据，这些stage的任务调度使用SourcePartitionedScheduler。

从数据源连接器那里获取一批分片，并准备调度这些分片。Presto的默认配置为每批最多调度1000个分片。FixedSplitSourc预先准备好所有的分片，Presto框架的SplitSource::getNextBlock每次会根据需要获取一批分片，FixedSplitSource根据需要的分片个数来返回。几乎所有的连接器都是用的FixedSplitSource，只有少数几个连接器（如Hive）实现了自己的ConnectorSplitSource。

根据SplitPlacementPolicy为这一批分片挑选对应的节点，建立一个map，key是节点， value是分片列表。

生成任务并调度到Presto查询执行节点上，任务的调度需要先绑定节点和分片的关系，再绑定分片和任务的关系，再将任务调度到查询执行节点上。分片选择了哪些查询执行节点，那么查询执行节点就会创建任务，分片选择了多少查询执行节点，就会有多少个查询执行节点创建任务，这会影响查询执行阶段的并发度。创建任务即创建某个RemoteTask的实例化对象，Presto默认实现和使用的是HttpRemoteTask。任务绑定分配到当前节点的分片之后，Presto会调用HttpRemoteTask.start，将任务分发到查询执行节点上。SqlStageExecution::scheduleTask详细描述了分发过程，构造了一个携带PlanFgragment和分片信息的HTTP Post请求，请求对应Presto查询执行节点的URI：`/v1/task/{taskId}`，Presto查询执行节点在收到请求后，会解析PlanFragment中的执行计划并创建SqlTaskExecution，开始执行任务。

```java
private synchronized RemoteTask scheduleTask(InternalNode node, TaskId taskId, Multimap<PlanNodeId, Split> sourceSplits, OptionalInt totalPartitions)
{
    checkArgument(!allTasks.contains(taskId), "A task with id %s already exists", taskId);

    ImmutableMultimap.Builder<PlanNodeId, Split> initialSplits = ImmutableMultimap.builder();
  	// 添加来自上游的数据源Connector的Split
    initialSplits.putAll(sourceSplits);
		// 添加来自上游查询执行阶段的任务的数据输出，注册为RemoteSplit
    sourceTasks.forEach((planNodeId, task) -> {
        TaskStatus status = task.getTaskStatus();
        if (status.getState() != TaskState.FINISHED) {
            initialSplits.put(planNodeId, createRemoteSplitFor(taskId, status.getSelf()));
        }
    });

    OutputBuffers outputBuffers = this.outputBuffers.get();
    checkState(outputBuffers != null, "Initial output buffers must be set before a task can be scheduled");

  	// 创建HttpRemoteTask
    RemoteTask task = remoteTaskFactory.createRemoteTask(
            stateMachine.getSession(),
            taskId,
            node,
            stateMachine.getFragment(),
            initialSplits.build(),
            totalPartitions,
            outputBuffers,
            nodeTaskMap.createPartitionedSplitCountTracker(node, taskId),
            summarizeTaskInfo);

    completeSources.forEach(task::noMoreSplits);
	
  	// 将刚创建的TaskId添加到当前查询执行阶段的TaskId列表中
    allTasks.add(taskId);
  	// 将刚创建的任务添加到当前查询执行阶段的节点与任务映射的map中
    tasks.computeIfAbsent(node, key -> newConcurrentHashSet()).add(task);
    nodeTaskMap.addTask(node, task);

    task.addStateChangeListener(new StageTaskListener());
    task.addFinalTaskInfoListener(this::updateFinalTaskInfo);

    if (!stateMachine.getState().isDone()) {
      	// 向Presto查询执行节点发请求，将刚创建的任务调度起来，开始执行
        task.start();
    }
    else {
        // stage finished while we were scheduling this task
        task.abort();
    }

    return task;
}
```

如果stage的数据源只有上游查询执行阶段的任务输出，使用FixedCountScheduler在选中的节点上调度任务。这些任务在Presto查询执行节点上执行时，将从上游查询执行阶段的任务OutputBuffer拉取数据计算结果。

```java
public class FixedCountScheduler
        implements StageScheduler
{
    public interface TaskScheduler
    {
        Optional<RemoteTask> scheduleTask(InternalNode node, int partition, OptionalInt totalPartitions);
    }
		// taskScheduler就是SqlStageExecution::schedulerTask方法
    private final TaskScheduler taskScheduler;
  	// 任务将调度到下面的节点上（Presto查询执行节点，每个节点对应一个任务）
    private final List<InternalNode> partitionToNode;
  
    @Override
    public ScheduleResult schedule()
    {
        OptionalInt totalPartitions = OptionalInt.of(partitionToNode.size());
        List<RemoteTask> newTasks = IntStream.range(0, partitionToNode.size())
                .mapToObj(partition -> taskScheduler.scheduleTask(partitionToNode.get(partition), partition, totalPartitions))
                .filter(Optional::isPresent)
                .map(Optional::get)
                .collect(toImmutableList());

        return new ScheduleResult(true, newTasks, 0);
    }
}
```

#### 关联ExchangeLocation

查询执行阶段的数据来源有两种，一种是数据源连接器，一种是上游查询执行阶段的任务可以输出到OutputBuffer的数据。对于下游的查询执行阶段来说，上游查询执行阶段的任务可以称为数据上游任务（upstream source task）。这些数据上游任务是通过SqlStageExecution::addExchangeLocations注册到下游的SqlStageExecution中的，让下游查询执行阶段知道去哪里取数据。无论是哪一种数据源，Presto都统一抽象为ConnectorSplit，当上游查询执行阶段作为数据源时，Presto把它看做是一种特殊的连接器。它的catalog name = $remote，其实就是一个假的目录，ConnectorSplit的实现类是RemoteSplit。

## 执行计划的执行

### 算子Operator

```java
public interface Operator
        extends AutoCloseable
{
    ListenableFuture<?> NOT_BLOCKED = Futures.immediateFuture(null);

    OperatorContext getOperatorContext();

    /**
     * Returns a future that will be completed when the operator becomes
     * unblocked.  If the operator is not blocked, this method should return
     * {@code NOT_BLOCKED}.
     */
    default ListenableFuture<?> isBlocked()
    {
        return NOT_BLOCKED;
    }

    /**
     * Returns true if and only if this operator can accept an input page.
     */
    boolean needsInput();

    /**
     * Adds an input page to the operator.  This method will only be called if
     * {@code needsInput()} returns true.
     */
    void addInput(Page page);

    /**
     * Gets an output page from the operator.  If no output data is currently
     * available, return null.
     */
    Page getOutput();

    /**
     * After calling this method operator should revoke all reserved revocable memory.
     * As soon as memory is revoked returned future should be marked as done.
     * <p>
     * Spawned threads cannot modify OperatorContext because it's not thread safe.
     * For this purpose implement {@link #finishMemoryRevoke()}
     * <p>
     * Since memory revoking signal is delivered asynchronously to the Operator, implementation
     * must gracefully handle the case when there no longer is any revocable memory allocated.
     * <p>
     * After this method is called on Operator the Driver is disallowed to call any
     * processing methods on it (isBlocked/needsInput/addInput/getOutput) until
     * {@link #finishMemoryRevoke()} is called.
     */
    default ListenableFuture<?> startMemoryRevoke()
    {
        return NOT_BLOCKED;
    }

    /**
     * Clean up and release resources after completed memory revoking. Called by driver
     * once future returned by startMemoryRevoke is completed.
     */
    default void finishMemoryRevoke()
    {
    }

    /**
     * Notifies the operator that no more pages will be added and the
     * operator should finish processing and flush results. This method
     * will not be called if the Task is already failed or canceled.
     */
    void finish();

    /**
     * Is this operator completely finished processing and no more
     * output pages will be produced.
     */
    boolean isFinished();

    /**
     * This method will always be called before releasing the Operator reference.
     */
    @Override
    default void close()
            throws Exception
    {
    }
}
```

- TableScanOperator：用于读取数据源连接器的数据
- AggregationOperator：用于聚合计算，内部可以指定一个或多个聚合函数，如sum，avg等
- TaskOutputOperator：查询执行阶段之间的任务做数据交换用的，上游查询执行阶段的任务通过此算子将计算结果输出到当前任务所在节点的OutputBuffer
- ExchangeOperator: 用于查询执行阶段之间进行任务的数据交换，下游查询执行阶段的ExchangeClient从上游OutputBuffer获取数据

### Page、Block、Slice

Slice表示一个值（Single Value），Block表示一列，类似于Parquet中的列（Column），Page表示多行记录。但是他们是以多列多个block的方式组织在一起，类似Parquet中的Row Group，这种组织方式，不同行相同列的字段都顺序相临，对应数据更容易按列读取与计算。

### TaskExecutor/Driver

TaskExecutor是Presto任务的执行池，它以单例的方式在Presto查询执行节点上启动，内部维护了一个Java线程池-ExecutorService用于提交运行任务，所以无论某个Presto查询执行节点上有多少个任务在运行，TaskExecutor都只有一个。

Driver是任务的算子链执行的驱动器，由它来推动数据穿梭于算子。在具体的实现上，火山模型是自顶而下的拉取（Pull）数据，Presto Driver和火山模型不一样，它是自底而上的推送（Push）数据。

```java
private ListenableFuture<?> processInternal(OperationTimer operationTimer)
for (int i = 0; i < activeOperators.size() - 1 && !driverContext.isDone(); i++) {
    Operator current = activeOperators.get(i);
    Operator next = activeOperators.get(i + 1);

    // skip blocked operator
    if (getBlockedFuture(current).isPresent()) {
        continue;
    }

    // 如果当前算子未完成任务，且下一个算子未被阻塞且需要输入
    if (!current.isFinished() && getBlockedFuture(next).isEmpty() && next.needsInput()) {
        // 从当前算子中获取计算输出的Page
        Page page = current.getOutput();
        current.getOperatorContext().recordGetOutput(operationTimer, page);

        // 如果能拿到一个Page，将其给到下一个算子
        if (page != null && page.getPositionCount() != 0) {
            next.addInput(page);
            next.getOperatorContext().recordAddInput(operationTimer, page);
            movedPage = true;
        }

        if (current instanceof SourceOperator) {
            movedPage = true;
        }
    }

    // if current operator is finished...
    if (current.isFinished()) {
        // let next operator know there will be no more data
        next.finish();
        next.getOperatorContext().recordFinish(operationTimer);
    }
}  
```

### 分批返回查询计算结果给SQL客户端

Presto采用的是流水线的处理方式，数据在Presto的计算过程中是持续流动的，是分批执行和返回的，在某个任务内不需要前面的算子计算完所有数据再输出结果给后面的算子，在某个查询内也不需要前面的查询执行阶段的所有任务都计算完所有数据再输出结果给后面的查询执行阶段。因此，从SQL客户端来看，Presto也支持分批返回查询计算结果给SQL客户端。Presto这种流水线的机制，与Flink非常类似，他们都不像Spark批式处理那样，需要前面的查询执行阶段执行完再执行后米娜的查询执行阶段。

如果你用过MYSQL这类关系型数据库，一定听说过游标（cursor），也用过各种编程语言的JDBC驱动的getNext，通过这样的方式来每次获取SQL执行结果的一部分数据。Presto也提供了类似的机制，它会给SQL客户端一个QueryResult，其中包含一个nextUri。对于某个查询，每次请求QueryResult，都会得到一个新的nextUri，它的作用类似于游标。

- StatementClientV1::advance
- ExecutingStatementResource::getQueryResults
- Query::getNextResult

## 执行计划生成的设计实现

### 从SQL到抽象语法树

所有提供SQL查询的数据库、查询引擎都需要语法分析能力，它把非结构化的SQL字符串转换成一系列的词法符号（Token），然后通过特定规则解析词法符号，返回一个语法分析树（ParseTree），最终得到一个结构化的树状结构，这就是抽象语法树（AST）。

访问者模式的优点：

- 避免修改元素类，遵循开闭原则
- 分离操作逻辑
- 支持新的操作拓展
- 多态分派

不同的Node类通过accept方法适配访问者模式，入参是当前访问者对象以及一个上下文对象。当前的节点会接受访问这，并调用访问者的特定方法，这里以抽象语法树的节点为例，Node类调用visitNode方法，Statement类调用visitStatement方法

```java
public abstract class Node
{
  protected <R, C> R accept(AstVisitor<R, C> visitor, C context)
    {
        return visitor.visitNode(this, context);
    }
}

public abstract class Statement
        extends Node
{
    @Override
    public <R, C> R accept(AstVisitor<R, C> visitor, C context)
    {
        return visitor.visitStatement(this, context);
    }
}
```

对一个抽象语法树使用访问者模式的时候，无论根节点是什么类型，仅需要使用node.accept(visitor)语句，节点会帮我们导航到对应的访问者方法。

```java
private Node invokeParser(String name, String sql, Function<SqlBaseParser, ParserRuleContext> parseFunction, ParsingOptions parsingOptions)
{
    try {
      	// 从sql字符串构建字节流，忽略大小写
        SqlBaseLexer lexer = new SqlBaseLexer(new CaseInsensitiveStream(CharStreams.fromString(sql)));
        CommonTokenStream tokenStream = new CommonTokenStream(lexer);
        SqlBaseParser parser = new SqlBaseParser(tokenStream);
        initializer.accept(lexer, parser);

        // Override the default error strategy to not attempt inserting or deleting a token.
        // Otherwise, it messes up error reporting
        parser.setErrorHandler(new DefaultErrorStrategy()
        {
            @Override
            public Token recoverInline(Parser recognizer)
                    throws RecognitionException
            {
                if (nextTokensContext == null) {
                    throw new InputMismatchException(recognizer);
                }
                else {
                    throw new InputMismatchException(recognizer, nextTokensState, nextTokensContext);
                }
            }
        });
				// PostProcessor用来显示捕捉某种语法错误并给出精确的错误提示
      	// 使用观察者模式，在语法分析的过程中触发回调
        parser.addParseListener(new PostProcessor(Arrays.asList(parser.getRuleNames()), parser));

        lexer.removeErrorListeners();
        lexer.addErrorListener(LEXER_ERROR_LISTENER);

        parser.removeErrorListeners();
        parser.addErrorListener(PARSER_ERROR_HANDLER);

        ParserRuleContext tree;
        try {
            // first, try parsing with potentially faster SLL mode
            parser.getInterpreter().setPredictionMode(PredictionMode.SLL);
  					// 生成语法分析树对象
            tree = parseFunction.apply(parser);
        }
        catch (ParseCancellationException ex) {
            // if we fail, parse with LL mode
            tokenStream.seek(0); // rewind input stream
            parser.reset();

            parser.getInterpreter().setPredictionMode(PredictionMode.LL);
            tree = parseFunction.apply(parser);
        }
				// 使用访问者模式生成以Node为父类的抽象语法树结构
        return new AstBuilder(parsingOptions).visit(tree);
    }
    catch (StackOverflowError e) {
        throw new ParsingException(name + " is too large (stack overflow while parsing)");
    }
}
```

通过访问者模式的一系列转换，Antlr的语法分析树结构终于被转换成Presto内部的抽象语法树结构。语法树节点不是扁平的结果，里面还存在很多抽象类。

- Statement：定义了最上层的SQL语句，Presto最常用的就是OLAP查询能力，即DQL语句（数据查询语言，对应Query），还有其他类型的SQL语句，如INSERT, EXPALIN, CREATE
- Relation：表示关系代数中的关系类型，可以是一张表或视图，一个查询，也可以是任意复杂的关联操作
- QeuryBody：表示DQL类型语法树的SQL结构，可以是集合操作组成的复杂SQL，也可以是基础的QuerySpecification结构，此外还支持特殊的Table查询和Values常量，一个查询本身也是一种关系，所以它归在了Relation下
- Expression：表示表达式的结构。表达式是比较特殊的一类结构，它是一种计算逻辑（或者说是计算表达式），由操作符号和操作数组成，比如a+1 , a>b

QueryBody是查询的主体结构，QuerySpecification指定了最常见的查询主体，Relation则是关系，表示数据的来源，最后几乎所有元素都会依赖表达式，从列名引用、表名、WHERE语句中的过滤谓词等都在表达式的范畴内。

### 语义分析

经过语法分析流程，一个SQL从最初的字符串转换成Presto引擎内部的抽象语法树结构。到了语义分析阶段，会结合元数据对抽象语法树做进一步的分析，主要是结合上下文验证SQL的正确性以及记录抽象语法树相关元数据。值得注意的是Presto在语义分析阶段没有生成新的树结构，分析所得到的元数据信息存放在一个Analysis类中，通过多个Map结构记录每个语法树节点（Key）的元数据信息（Value）。后续流程通过查找Analysis结构来获取对应的元数据。

数据源信息读取：包括表、视图、物化视图的表信息TableHandle，列信息ColumnHandle以及对应的表元数据TableMetadata，列元数据ColumnMetadata。查询这些元数据一方面是为了将其应用于后续的执行计划生成，另一个方面用于语义检查。

语义检查：有很多SQL规则是需要上下文信息共同维护的，包括对多节点的内容进行联合比对。比如SELECT语句的非聚合函数列是否和GROUP BY语句匹配，该约束在语义分析中由AggregationAnalyzer进行检查，这些是语法分析做不了的。

语义理解：包括消除歧义以及语法糖展开等操作

- a.b，指的是a表的b列还是Row Type类型a列的b字段，语法规则可能会产生二义性，语义分析阶段需要解决这些问题
- select * from xxx，将*展开成所有列

元数据分析：所有下游需要的信息都会在语义分析阶段记录下来，记录在Analysis对象中

- 数据域（scope）分析：每个关系节点当前可用的列信息，通过自底向上的方式计算出来
- 聚合函数分析：收集每个QuerySpecification结构的聚合函数
- 类型分析：对表达式结构Expression的每个节点进行自底向上的类型推导，其中包含隐式转换信息的记录
- 函数解析：表达式的内容包含函数调用和操作符，本质上都可以看做是函数调用，引擎需要分析与抽象语法树对应的函数结构FunctionCall是否存在，参数类型是否匹配等

语义分析阶段有两个核心的分析器StatementAnalyzer和ExpressionAnalyzer，一个用于分析Statement对象，一个用于分析Expression表达式对象。上文提到的类型分析、函数解析就是在ExpressionAnalyzer中完成的。他们的关注点不一样，这也是语法树节点为什么会有不同的抽象类。

**数据域分析**

Scope是StatementAnalyzer访问者的返回值，它表示当前抽象语法树节点能被引用的列信息，这里称为数据域。它的用途是记录可用的列信息，在语义分析中用来验证列名字符串是否存在于数据域中（合法性校验）。

- 列信息用Field表示，它不仅包含了列名Name，类型信息Type，还包含了所属关系的别名relationAlias，比如Join节点，可以引用两个子关系的列，这个时候需要加上子表的名称才能构成唯一标识符。
- RealtionType表示当前节点的关系，底层是Field列表，它封装了所有对关系的操作，是一个工具类
- Scope建模了更加复杂的情况，对于关联、子查询等特殊场景（如semi-join/lateral join），除了当前的关系，还能引用外层的关系。通过parent字段（父级数据域）可以建模这种情况，验证字段合法性的时候可以查找父级数据域来判断字段是否存在。

```sql
--- schema 为 tpch.sf1
SELECT
	(SELECT name FROM region r WHERE regionkey = n.regionkey)
		AS region name,
	n.name AS nation_name
FROM nation n
```

region和nation这两个关系分别有自己的数据域，但是region所在域的parent指向了nation域，所以在子查询内不仅可以使用region表的列，还能使用外层nation表的列。

