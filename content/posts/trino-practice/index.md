---
title: "Presto的计算流程"
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
    - "Presto"
categories:
    - "Presto"
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

#### 数据域分析

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

 另外，CTE（Common Table Expression）特性使得查询中可以使用一些自定义的视图（表），通过namedQueries可以记录这些信息，注意它用来验证“表名字符串”是否合法，不是用来验证列信息的。

数据域的核心功能是识别语法结构中的字符串，判断它是否是一个合法的列引用（ColumnReference）信息。该过程主要发生在表达式分析器ExpressionAnalyzer的visitIdentifier方法和visitDereferenceExpression方法中，他们分别用来判断字符串和带"."分隔符的字符串是否合法。识别动作的入口是Scope.resolveField函数，Field的名称包括列名以及底层关系名或者关系别名。Field的名称包括列名以及底层关系名或者关系别名。

#### 数据源信息读取

对于查询类的SQL语句来说，元数据读取主要发生在Table节点，Presto引擎需获取以下信息：

- TableHandle：通过连接器获取表的数据结构，它的内容和底层数据源有关。Presto仅定义了一个抽象的ConnectorTablehandle接口，里面的内容为空。使用场景比如PageSourceProvder.createPageSource是Presto建模的数据读取接口，里面就有TableHandle作为入参，所以它需要包含数据表读取所需的内容。
- ColumnHandle：通过连接器获取当前表的列信息，和TableHandle一样它也是一个抽象的定义
- 元数据信息：包括ConnectorTableMetadata和ColumnMetadata，两者分别是表和列的元数据信息，主要包括名称、类型、注释、属性。列的类型信息（Type)是最底层的类型，它是Presto定义的类型信息，Field的类型信息就是从元数据获取的。

#### 语义理解

语义分析需要正确地理解表达式含义，处理有歧义的逻辑并提供归一化的数据结构，尽量为下游的执行计划模块屏蔽语法细节。Presto的列名和RowType的子字段其实代表不同的语义，但是复用了相同的语法结构DereferenceExpression。

DereferenceExpression节点的语义理解过程如下，Presto中的Dereference表示类似x.y.z这样的表达式，包含至少一个"."分隔符，参考g4文件的定义，base=primaryExpression'.'fieldName=identifier，其中suffix是标识符类型，prefix可以是任意表达式类型。

对于x.y.z这样的DereferenceExpression结构，可以表示为如下几种情况：

1. 数据库为x，表为y，列为z的列名（如果一个语法结构指向一个列名，那么它是一个ColumnReference）
2. x表y列的z字段，其中y是RowType类型（类似于Hive的Struct结构）
3. x列的y列的z字段，x列和y列都是RowType类型

因为存在上述多种情况，所以第一步尝试获取该结构的QualifiedName，此时DeferenceExpression需要满足每一段都是标识符，这样才可能是一个列引用。针对QualifiedName非空的情况使用数据域的tryRosolveField方法识别当前字段，有如下3种可能：

1. 列引用识别成功，命中刚刚提到的情况1
2. 未能识别到列引用，此时调用数据域的isColumnReference判断表达式的所有Prefix前缀子集是否能识别为列引用，即情况2、3，识别成功，则进入下面主逻辑，调用process递归处理base部分的语法结构。
3. 识别失败，说明当前DereferenceExpression非法，抛出异常

```java
  @Override
  protected Type visitDereferenceExpression(DereferenceExpression node, StackableAstVisitorContext<Context> context)
  {
    	// 尝试转换成QualifiedName，这种情况下要求每个部分都是表示符
      QualifiedName qualifiedName = DereferenceExpression.getQualifiedName(node);

      // 如果这个Dereference结构看起来像是带有库表限定的列名，先尝试匹配到列
      if (qualifiedName != null) {
          Scope scope = context.getContext().getScope();
        	// 尝试识别该字段
          Optional<ResolvedField> resolvedField = scope.tryResolveField(node, qualifiedName);
          if (resolvedField.isPresent()) {
              return handleResolvedField(node, resolvedField.get(), context);
          }
        	// 是一个非法的结构
          if (!scope.isColumnReference(qualifiedName)) {
              throw missingAttributeException(node, qualifiedName);
          }
      }
			
    	// 前缀存在列引用，进入递归解析，递归分析base结构
      Type baseType = process(node.getBase(), context);
      if (!(baseType instanceof RowType)) {
          throw semanticException(TYPE_MISMATCH, node.getBase(), "Expression %s is not of type ROW", node.getBase());
      }
			
    	// baseType一定是RowType
      RowType rowType = (RowType) baseType;
    	// 根据fieldName确定子列名称
      String fieldName = node.getField().getValue();

      Type rowFieldType = null;
    	// 验证fieldNmae是否真的存在
      for (RowType.Field rowField : rowType.getFields()) {
          if (fieldName.equalsIgnoreCase(rowField.getName().orElse(null))) {
              rowFieldType = rowField.getType();
              break;
          }
      }

      if (rowFieldType == null) {
          throw missingAttributeException(node, qualifiedName);
      }

      return setExpressionType(node, rowFieldType);
  }
```

### 生成初始逻辑执行计划

语法分析生成了抽象语法树，语义分析通过查询元数据和进一步解析生成了一个Analysis元数据对象。语义分析、执行计划是两套不同的体系，理论上是相互独立、各自演进。执行计划本质上是一种中间表达式（Intermidiate Representation, IR)，一种偏底层的描述语言，还是一种树状结构。IR使得优化器能够对SQL更好地进行优化。PlanNode类是所有执行计划节点的父类，它定义了IR结构的通用方法。生成初始逻辑执行计划就是遍历AST（抽象语法树），结合Analysis元数据生成IR的过程。优化器则是按照某些优化规则修改执行计划树的过程。

初始逻辑执行计划整体上在做如下两件事情。

- Statement结构转换：本质上是AST到IR结构的转变，结合语义分析阶段的Analysis结构，Presto通过访问者模式自底向上把AST结构转换成不同PlanNode结构。
- Expression改写：对Expression结构进行重写，将变量名替换成Symbol唯一标识符，同时保留元数据信息，将ResolvedFunction信息编码到函数名称中。

这不是独立发生的两件事情，Statement结构的转换会伴随着Expression的改写，只不过Expression的处理方式比较特别。

#### Statement结构转换

AST的核心骨架是Statement结构，SQL语句最外层是Query，通过追踪LogicalPlanner的实现可以定位到createRelationPlan函数，它也是基于访问者模式来生成初始执行计划的，这里依赖两个核心类来完成执行计划的生成。

- RelationPlanner: 一个访问者，它专门处理关系结构之间的运算，比如关联JOIN、集合Union、底层表、表抽样等关系类型
- QueryPlanner：对单个关系进行的变换操作，它包括过滤、投影变换、聚合、窗口函数、排序、分页等操作，核心就是把QuerySpecification结构包含的操作转换成执行计划节点

RelationPlanner和QueryPlanner之间是相互嵌套的，遇到Relation就是调用RelationPlanner，遇到Query就会使用QueryPlanner。

#### Expression改写

Presto引擎的核心模块之间存在耦合的情况，虽然AST和IR有着本质的区别，但是具体到Expression结构（Expression的子类），由于它表达的是一种逻辑运算，所以表达式在语法树和执行计划之间是复用的，语法分析生成的Expression结构一直被沿用到执行阶段。

IR使用Symbol作为全局唯一标识符来代替AST里面的变量名。AST中的变量名有很多表示方法，可能会出现重复，比如a + 1 as a，虽然AST把SQL结构化成语法分析树，但AST的表达式只有人能够理解，OLAP引擎还不能理解表达式之间变量的引用关系，如果t.col_a或者col_a指代的是哪一列，OLAP引擎无法解决。

为了复用AST的Expresssion结构，Presto为Symbol引入了一个SymbolReference表达式，它表示的是一个内部符号（Symbol），以取代AST里面的Identifier等变量名。内部符号本质就是字符串，它是唯一标识符，全局唯一。Symbol的生成逻辑位于SymbolAllocator中。

```java
public class Symbol
        implements Comparable<Symbol>
{
    private final String name;

    public static Symbol from(Expression expression)
    {
      	// 必须是SymbolReference类型的表达式
        checkArgument(expression instanceof SymbolReference, "Unexpected expression: %s", expression);
        return new Symbol(((SymbolReference) expression).getName());
    }

    @JsonCreator
    public Symbol(String name)
    {
        requireNonNull(name, "name is null");
        this.name = name;
    }

    @JsonValue
  	// 生成Symbol，name是唯一的
    public String getName()
    {
        return name;
    }
		// Symbol转SymbolReference
    public SymbolReference toSymbolReference()
    {
        return new SymbolReference(name);
    }

// Symbol的wrapper，复用Expression体系
public class SymbolReference
        extends Expression
{
    private final String name;

    public SymbolReference(String name)
    {
        super(Optional.empty());
        this.name = name;
    }  
```

这里执行计划会用到如下两种元数据信息：

- 表达式的类型信息：即Presto的Type，通过一个TypeAnalyzer工具类在执行计划期间重新进行语义分析来获取Expression类型信息
- 函数元数据信息ResolvedFunction，在语义分析阶段通过函数解析获取，参考ExpressionAnalyzer.visitFunctionCall方法，后续常量折叠等优化器需要用到它。该信息直接通过序列化+ZSTD（一种压缩算法名称）压缩+BASE32编码的方式集成到函数名称的字符串中。

#### 使用PlanNode表达逻辑执行计划

PlanNode是一个抽象类，包含以下核心函数：

- getSources：用于遍历执行计划，它会返回当前节点的子节点（从数据流的角度看，子节点是数据的上游）
- replaceChildren：替换子节点，在访问者模式中递归生成新的执行计划
- getOutputSymbols：向父节点提供的字段集合，类型是Symbol
- accept：访问者模式的通用接口，执行计划节点的访问者父类是PlanVisitor

```java
public abstract class PlanNode
{
    private final PlanNodeId id;

    protected PlanNode(PlanNodeId id)
    {
        requireNonNull(id, "id is null");
        this.id = id;
    }

    @JsonProperty("id")
    public PlanNodeId getId()
    {
        return id;
    }

    public abstract List<PlanNode> getSources();

    public abstract List<Symbol> getOutputSymbols();

    public abstract PlanNode replaceChildren(List<PlanNode> newChildren);

    public <R, C> R accept(PlanVisitor<R, C> visitor, C context)
    {
        return visitor.visitPlan(this, context);
    }
}
```

**TableScanNode**

数据读取节点，一个TableScanNdoe代表一个底层数据源的读取操作，核心是如下3个变量：

- TableHandle：描述了如何读取一张表，里面包含了这个数据的位置信息，以及下推后的元数据信息，比如limit下推、聚合下推等信息
- outputSymbols：当前节点提供的输出列，父类的重载方法getOutputSymbols会返回这个变量
- assignments：记录了数据源的列信息到Symbol的映射关系。ColumnHandle建模了一个数据列，它可能存储额外的信息，比如ProjectionPushDown优化会裁剪复合列的读取，它仅读取复合结构中所需的子列，Hive连接器把这个信息记录在HiveColumnHandle中。注意这里的Symbol并不对应某个表达式结构，因为这里已经是Symbol血缘关系的最上游，其他Symbol都是通过重写表达式生成的。

**FilterNode**

数据过滤节点，FilterNode对应了WHERE语句的数据过滤结构，它是一个表达式，记录在predicate变量中。source变量存储了上游的节点引用。结合前面提到的Symbol替换操作和上游节点的getOutputSymbols函数可以知道，在生成FilterNode的时候会把predicate表达式中的变量替换成上游的Symbol，这样表达式结构就从AST结构转换成IR结构了，引擎也就知道这个Symbol是从哪里来的，所以Symbol本质上是一种血缘信息。

**ProjectNode**

投影变换节点，投影变换节点是很常见的节点，可以对应SELECT语句中的变换操作，在引擎中也承担了很多非用户指定的转换操作，这些转换也是表达式结构。它的核心结构是Assignments，它使用了一个哈希表来记录与投影变换表达式对应的输出列（同样是Symbol类型）

**ExchangeNode**

数据交换节点，ExchangeNode用来描述数据交换的相关参数，它的地位很特殊，仅存在于逻辑执行计划阶段，后续分布式执行计划生成的时候，Fragmenter会依据此节点切分成多个查询执行阶段（Stage）。

**AggregateNode**

聚合操作节点，它专门处理当前查询结构的所有聚合操作

## 执行计划优化的目的、基本原理和基础算法

### 执行计划优化的基本原理

SQL作为一项图灵奖级别的发明，其重要意义不仅在于发明了一种可以用作数据查询的语言，更重要的是发明了关系代数（relation algebra)这一工具，使得计算机理解和处理查询的语义更加方便。SQL查询语言的优化也是基于关系代数这一模型进行的。

所谓关系代数，是SQL从语句到执行计划的一种中间表示，首先它不是单纯的AST，而是一种经过进一步处理得到的中间表示，可以类比一般编程语言的中间语言（Intermidate representation），SQL优化的本质是对关系代数的优化。

总结使用关系代数进行查询优化的要点：

- SQL查询可以表示为关系代数
- 关系代数作为一种树形结构，实质上也可以表示查询的物理实现方案
- 关系代数可以进行局部的等价变换，变换前后返回的结果不变，但执行的成本不同
- 通过寻找执行成本最低的关系代数表示，就可以将一个SQL查询优化成更为高效的方案

此外，很重要的一点是：实现关系代数的化简和优化，要依赖数据系统的物理性质，如存储设备的特性（顺序读性能、随机读性能、吞吐量）、存储内容的格式和排列（列式存储、行式存储、对某列进行分片）、包含的元数据和预计算结果（是否存在索引或物化视图）、聚合和计算单元的特性（单线程、并发计算、分布式计算、特殊加速硬件）。

综上所述，对SQL查询进行优化，既要在原先逻辑算子的基础上进行变换，又要考虑物理实现的特性，这就是很多查询系统存在逻辑方案和物理方案区别的原因。在优化时，往往存在一个从逻辑方案到物理方案进行变换的阶段。

事实上，从逻辑方案到物理方案的变换也可以划归为一种关系代数的优化问题，因为其本质上仍然是按照一定的规则将原模型中的局部等价地变换成一种可以物理执行的模型或算子。

### 执行计划优化的基本算法

现行主流的执行计划优化算法分为两类，基于规则的优化（RBO）和基于成本的优化（RBO）。

基于规则的优化的思路很简单，根据经验事先定义一些规则，比如谓词下推、topN优化等，将原先的执行计划转换成新的执行计划。

基于规则的优化算法在实际使用中仍然面对很多问题：

- 变换规则的选择问题：哪些规则应该被应用？以什么顺序被使用？
- 变换效果评估的问题：经过变换的查询性能是否会变好？各种可能得方案那个更优？

现阶段主流的方法都是基于成本估算的方法。也就是说，给定某一关系代数代表的执行方案，对这一方案的执行成本进行估算，最终选择估算成本最低的方案。虽然被称为基于成本的估算，但这类算法仍然需要结合规则进行方案探索。也就是说，基于成本的方法其实是通过不断的应用规则进行变换得到新的执行方案，然后对于方案的成本优劣进行最终选择。

Volcano Optimizer（火山优化器）是一种基于成本的优化算法，其目的是基于一些假设和工程算法的实现，在获得成本较优的执行计划的同时，可以通过剪枝和缓存中间结果（动态规划）的方法降低计算消耗。

**成本最优假设**

成本最优假设是理解Volcano Optimizer实现的要点之一。这一假设认为，在最优的方案当中，取局部的结果来看其方案也是最优的。成本最优假设利用了贪心算法的思想，在计算的过程中，如果一个方案是由几个局部区域组合而成的，那么在计算总成本时，我们只需要考虑每个局部目前已知的最优方案和成本即可。对于成本最优假设的更直观的描述是，如果关系代数局部的某个输入的计算成本上升，那么这一子树的整体成本趋向于上升，反之会下降。上述假设对于大部分关系代数算子都是有效的，但是并非百分之百准确。

**动态规划算法与等价集合**

由于引入了成本最优假设，在优化过程中就可以对任意子树目前已知的最优方案和最优成本进行缓存。在此后的计算过程中，如果需要利用这一子树，可以直接使用之前缓存的结果，这里应用了动态规则算法的思想。要实现这一算法，只需要建立缓存结果到子树双向映射，某一刻子树所有可能的变换方案组成的集合称为等价集合（Equivalent Set），等价集合将会维护自身元素当中具有最优成本的方案。

**自底向上和自顶向下**

自底向上的动态规划算法最为直观：当我们试图计算节点A的最优方案时，其子树上的每个节点对应的等价结合和最优方案都已经计算完成了，我们只需要在A节点上不断寻找可以应用的规则，并利用已经计算好的子树成本计算出母树的成本，就可以得到最优方案。然而这一方案存在以下难以解决的问题：

- 不方便应用剪枝技巧：在查询中可能会遇到在父节点的某一种方案成本很高，后续完全无需考虑的情况，尽管如果，需要被利用的子计算都已经完成，这部分计算不可避免。
- 难以实现启发式计算和限制计算层数：由于程序要不断递归到最后才能得到比较好的方案，因此即使计算量比较大也无法提前获取到一个可行的方案并停止运行。

因此，Volcano Optimizer采用了自顶向下的计算方法，在计算开始，每棵子树先按照原先的样子计算成本并作为初始结果。在不断应用规则的过程中，如果出现一种新的结构被加入当前的等价集合中，且这种等价集合具有更优的成本，这时需要向上冒泡到所有依赖这一子集合的父亲等价集合，更新集合里每个元素的成本并得到新的最优成本和方案。值得注意的是，在向上冒泡的过程中需要遍历父亲集合内的每一个方案，这是因为不同方案对于投入成本变化的敏感性不同，不能假设之前的最优方案仍然是最优的。自顶向上的方案尽管解决了一些问题，但是也带来了对关系代数节点操作繁琐、要不断维护父子等价集合的关系等问题，实现相对复杂。推荐Calcite。

### 执行计划优化的设计实现

Presto的执行计划包括初始逻辑执行计划（LogicalPlanner.planStatement）、优化后的逻辑执行计划（PlanOptimizer.optimize）、分布式执行计划（PlanFragmenter.createSubPlans）和物理执行计划（LocalExecutionPlanner.plan）4种。

逻辑执行计划：对SQL的语法分析树进行翻译，转换中间表达式IR，这个时候就是一个初始逻辑执行计划，它比声明式语言更丰富、更底层，比如GROUP BY语句会被翻译为（ProjectNode/GroupIdNode + AggregationNode），这种中间结构使得优化器能够很好地对它进行转换。

优化器：初始逻辑执行计划可以做进一步的优化，比如谓词下推、TopN转换等，这个时候初始逻辑执行计划变成优化后的逻辑执行计划。Presto有迭代式优化器和非迭代式优化器两种，主要是RBO规则，它们根据PlanOptimizers中定义的顺序来执行。这里其实还有一个基于CBO的ReorderJoin优化器，它使用查并集（Disjoint Set）结构计算出等价的JOIN结构，然后运用动态规划算法计算出成本最低的JOIN顺序，Presto没有在全局的维度维护多个执行计划，这一点和业界的CBO算法不一样。

分布式执行计划和物理执行计划：因为逻辑执行计划最终是为了转成物理执行计划，而优化器在优化的过程中也会借助底层的特性，所以在优化的过程中，从逻辑执行计划到物理执行计划是一个渐变的过程，两者之间没有明显的边界。优化后的逻辑计划其实是介于物理执行计划和逻辑执行计划之间的。Fragmenter最后把优化后的逻辑执行计划进行分片转换成分布式执行计划，每个分片就是一个查询执行阶段，这个Fragment结构在查询执行节点（Worker）上稍加转换就变成了最终的物理计划。

在Presto中所有优化器都需要实现PlanOptimizer接口，优化器主要分为迭代式优化器和非迭代式优化器（相对于迭代优化器而言，实际它不是显式分类）。**迭代式优化器**指多个功能相似的优化规则（Rule）组成一个规则组，每个规则专注一种逻辑，每个规则的逻辑相对简单，规则组内的所有优化规则会被尝试应用到当前的执行计划，如果执行计划有更新，会不断循环应用规则组的规则，直到执行计划不再改变。**非迭代式优化器**是Presto项目早期使用较多的优化器，这类优化器都需要实现一个完整的逻辑执行计划树访问者（visitor）并通过层层遍历完成优化逻辑。它擅长通过自顶向下或自底向上的方式来遍历逻辑执行计划树，从而完成一些逻辑复杂，需要状态的优化操作。很多经典的优化器（比如AddExchanges）就是基于访问者模式实现了插入全局数据交换节点（ExchangeNode）功能的。

#### 迭代式优化器

每个优化规则都有特定的目标节点，可能还有一些附加的触发条件，判断当前执行计划节点是否能应用某个优化规则，这类需求称为模式匹配，它提供一下几个功能：

- 指定作用的目标节点
- 指定节点需要满足的条件，这是一个逻辑表达式，可以由多个语句组成一个逻辑与结构
- 捕获匹配到的节点，方便优化器引用它们

```java
public class PushPartialAggregationThroughExchange
        implements Rule<AggregationNode>
{
    private final Metadata metadata;

    public PushPartialAggregationThroughExchange(Metadata metadata)
    {
        this.metadata = requireNonNull(metadata, "metadata is null");
    }

    private static final Capture<ExchangeNode> EXCHANGE_NODE = Capture.newCapture();

  	// pattern，目标节点为AggregationNode，且子节点为ExchangeNode，且数据交换节点不能有排序要求
  	// 因为在排序情况下聚合函数无法进行预聚合优化，只能在单个任务中完成聚合，captureAs捕获命中的exchange节点
    private static final Pattern<AggregationNode> PATTERN = aggregation()
            .with(source().matching(
                    exchange()
                            .matching(node -> node.getOrderingScheme().isEmpty())
                            .capturedAs(EXCHANGE_NODE)));

    @Override
    public Pattern<AggregationNode> getPattern()
    {
        return PATTERN;
    }

    @Override
    public Result apply(AggregationNode aggregationNode, Captures captures, Context context)
    {
        ExchangeNode exchangeNode = captures.get(EXCHANGE_NODE);

        boolean decomposable = aggregationNode.isDecomposable(metadata);
```

Memo原本在Cascacd优化算法中是用来存储每个Group节点下面的等价执行计划片段，因为Presto始终只有一个执行计划，所有Memo的作用变成维护一个可变执行计划，同时Memo可以自动识别需要删除的节点，因为执行计划只有替换和插入操作，不会进行显式删除，所以需要一种识别机制来删除已废弃的节点。通过调用Memo.insertRecursive进行初始化，执行计划的每个节点都被映射成一个分组，也就是Memo.Group结构，原本PlanNode.getSources返回的下游节点，在这里全部替换成虚拟的GroupReference节点，通过它的id可以定位到一个分组，分组里存储着实际的执行计划节点，通过这种方式，节点之间的关系就解耦了，所以Memo的本质是：

- 把一个执行计划变成一个Map结构，键是分组的id，值是对应的Group节点，里面存储了实际的执行计划节点
- 把执行计划节点的下游节点替换成GroupReference，这个分组是不变的，但是分组里面的计划节点可以发生变化

IterativeOptimizer初始化时接收多个功能相似的优化规则作为入参并构建一个RuleIndex结构，它的key是优化规则中Pattern匹配的目标节点，这样在遍历执行计划的时候可以更高效地过滤无关规则。optimize逻辑将待优化的执行计划转换成一个Memo结构，然后对执行计划进行自顶向下的遍历，由以下3中explore函数负责特定范围的遍历逻辑。

- exploreGroup：完成当前节点所在子树的遍历，它由exploreNode和exploreChildren组成，外加一些条件判断语句
- exploreNode：对当前节点应用规则组的所有优化规则
- exploreChildren：对当前节点的所有子节点进行遍历

optimize重载了父类的方法，new Memo命令把执行计划转换成Memo结构，调用getRootGroup获取根节点进行exploreGroup遍历操作。注意所有exploreXXX函数都会返回布尔值变量来标识是否更新了执行计划。如果有进展，exploreGroup内部的exploreNode+exploreChildren会重新执行一次。执行计划树的PlanNode本身是不可变的，如果当前节点有变更，需要递归更新所有父节点，而优化器会频繁变更执行计划树，所以通过Memo来支持可变执行计划。

```java
public class IterativeOptimizer
        implements PlanOptimizer
{
    private final RuleStatsRecorder stats;
    private final StatsCalculator statsCalculator;
    private final CostCalculator costCalculator;
    private final List<PlanOptimizer> legacyRules;
    private final RuleIndex ruleIndex;
    private final Predicate<Session> useLegacyRules;

    public IterativeOptimizer(RuleStatsRecorder stats, StatsCalculator statsCalculator, CostCalculator costCalculator, Set<Rule<?>> rules)
    {
        this(stats, statsCalculator, costCalculator, session -> false, ImmutableList.of(), rules);
    }

    public IterativeOptimizer(RuleStatsRecorder stats, StatsCalculator statsCalculator, CostCalculator costCalculator, Predicate<Session> useLegacyRules, List<PlanOptimizer> legacyRules, Set<Rule<?>> newRules)
    {
        this.stats = requireNonNull(stats, "stats is null");
        this.statsCalculator = requireNonNull(statsCalculator, "statsCalculator is null");
        this.costCalculator = requireNonNull(costCalculator, "costCalculator is null");
        this.useLegacyRules = requireNonNull(useLegacyRules, "useLegacyRules is null");
        this.legacyRules = ImmutableList.copyOf(legacyRules);
      	// 初始化，构造ruleIndex
        this.ruleIndex = RuleIndex.builder()
                .register(newRules)
                .build();

        stats.registerAll(newRules);
    }

  	// 优化逻辑，重写PlanOptimizer中的方法
    @Override
    public PlanNode optimize(PlanNode plan, Session session, TypeProvider types, SymbolAllocator symbolAllocator, PlanNodeIdAllocator idAllocator, WarningCollector warningCollector)
    {
        // only disable new rules if we have legacy rules to fall back to
        if (useLegacyRules.test(session) && !legacyRules.isEmpty()) {
            for (PlanOptimizer optimizer : legacyRules) {
                plan = optimizer.optimize(plan, session, symbolAllocator.getTypes(), symbolAllocator, idAllocator, warningCollector);
            }

            return plan;
        }
				// 转换成Memo
        Memo memo = new Memo(idAllocator, plan);
        Lookup lookup = Lookup.from(planNode -> Stream.of(memo.resolve(planNode)));
				// 超时控制
        Duration timeout = SystemSessionProperties.getOptimizerTimeout(session);
        Context context = new Context(memo, lookup, idAllocator, symbolAllocator, System.nanoTime(), timeout.toMillis(), session, warningCollector);
        exploreGroup(memo.getRootGroup(), context);
				// 从Memo结构转换成执行计划
        return memo.extract();
    }

    private boolean exploreGroup(int group, Context context)
    {
				// progress变量跟踪当前Group或者子节点是否由于应用了优化规则而发生变换
      	// 对当前节点进行优化
        boolean progress = exploreNode(group, context);
				// 递归遍历子节点进行优化
        while (exploreChildren(group, context)) {
            progress = true;
						如果子节点发生变更，再次尝试对当前节点进行优化，也许能应用新的规则
            if (!exploreNode(group, context)) {
                // 当前节点无更新，退出
                break;
            }
        }

        return progress;
    }

    private boolean exploreNode(int group, Context context)
    {
      	// 根据GroupID获取真正的执行计划节点
        PlanNode node = context.memo.getNode(group);

        boolean done = false;
        boolean progress = false;

        while (!done) {
            context.checkTimeoutNotExhausted();

            done = true;
          	// getCandicates可以快速过滤无关的优化规则
            Iterator<Rule<?>> possiblyMatchingRules = ruleIndex.getCandidates(node).iterator();
            while (possiblyMatchingRules.hasNext()) {
                Rule<?> rule = possiblyMatchingRules.next();

                if (!rule.isEnabled(context.session)) {
                    continue;
                }
								// 对当前节点应用一个优化规则
                Rule.Result result = transform(node, rule, context);
								// 成功应用了规则
                if (result.getTransformedPlan().isPresent()) {
                  	// 用优化后的结果替换当前节点
                    node = context.memo.replace(group, result.getTransformedPlan().get(), rule.getClass().getName());

                    done = false;
                    progress = true;
                }
            }
        }

        return progress;
    }

    private <T> Rule.Result transform(PlanNode node, Rule<T> rule, Context context)
    {
        Capture<T> nodeCapture = newCapture();
        Pattern<T> pattern = rule.getPattern().capturedAs(nodeCapture);
      	// 调用match函数进行模式匹配
        Iterator<Match> matches = pattern.match(node, context.lookup).iterator();
        while (matches.hasNext()) {
            Match match = matches.next();
            long duration;
            Rule.Result result;
            try {
                long start = System.nanoTime();
              	// 应用单条优化规则
                result = rule.apply(match.capture(nodeCapture), match.captures(), ruleContext(context));
                duration = System.nanoTime() - start;
            }
            catch (RuntimeException e) {
                stats.recordFailure(rule);
                throw e;
            }
            stats.record(rule, duration, !result.isEmpty());

            if (result.getTransformedPlan().isPresent()) {
                return result;
            }
        }

        return Rule.Result.empty();
    }

    private boolean exploreChildren(int group, Context context)
    {
        boolean progress = false;

        PlanNode expression = context.memo.getNode(group);
        for (PlanNode child : expression.getSources()) {
            checkState(child instanceof GroupReference, "Expected child to be a group reference. Found: " + child.getClass().getName());

            if (exploreGroup(((GroupReference) child).getGroupId(), context)) {
              	// 记录执行计划是否发生变更
                progress = true;
            }
        }

        return progress;
    }
```



核心步骤在于单个执行计划节点的优化，即exploreNode函数，注意explore的入参都是GroupID，需要查找Memo获取真正的执行计划节点。这里有如下两个变量：

- done：用来表示是否还需要对当前节点应用优化规则，如果有优化规则被成功地应用到当前节点，则说明还可以继续尝试
- progress：只要有优化规则被成功应用，progress=true

transform函数会使用Pattern进行匹配，如果规则被成功应用，那么该函数返回的结果不为空，此时调用Memo.replace原执行计划节点替换成新的执行计划节点，transform函数执行分成两步：

- 获取当前规则的Pattern进行模式匹配，通过这一步确定是否应用当前规则
- 如果匹配成功，那么会调用Rule.apply函数执行优化规则，第一个入参是Pattern指定的目标节点，第二个入参是Captures对象，通过它可以找到Pattern中的某个执行计划节点

#### 非迭代式优化器

非迭代式优化器只需要实现PlanOptimizer接口，具体的优化方式不限，一般来说，这类优化器都需要通过访问者模式遍历执行计划树，适用于需要在执行计划节点之间建立密切联系的优化规则。这类优化器大多数都能通过自顶向下+自底向上的方式来计算当前子树的某些属性，当不满足属性或者满足特定条件的时候，会插入特定的执行计划节点或者修改当前节点的某些属性。

- AddExchanges: 添加查询分布式执行时的数据交换节点，对应的节点是ExchangeNode[scope=REMOTE]
- AddLocalExchanges：添加单个任务执行时内部的数据交换节点，对应的节点是ExchangeNode[scope=LOCAL]
- HashGenerationOptimizer：为需要分区的执行计划节点根据分区用到的列提前计算好哈希值，减少重复计算
- PredicatePushDown：自顶向下把过滤谓词尽量下推，使其靠近数据源读取的算子，此外动态过滤（dynamic filtering）特性也被当做一种下推操作在这个优化器中实现
- UnaliasSymbolReference：将多余的投影映射简化，因为初始逻辑执行计划和优化器都会引入一些类似col_a as col_b的投影变换，这些变换可能是多余的，优化后整个逻辑执行计划看起来更易懂、清晰。

