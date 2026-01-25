# Presto执行计划的调度


在Presto的执行模型中，SQL的执行被划分为如下几个层次：

- 查询：用户提交一个SQL，触发Presto的一次查询，在代码中对应一个QueryInfo。每个查询都有一个字符串形式的QueryId
- 查询执行阶段：Presto生成查询的执行计划时，根据是否需要做跨查询执行节点的数据交换来划分PlanFragment。调度执行计划时，每个PlanFragment对应一个查询执行阶段，在代码中对应一个StageInfo，其中有StageId，StageId的形式为QueryId + 从0自增id。查询执行阶段之间有数据依赖关系，即不能并行执行，存在执行上的顺序关系，需要注意的是，StageId越小，这个查询执行阶段的执行顺序越靠后。Presto的查询执行阶段类似于Spark的查询执行阶段的概念，他们的不同是Presto不像Spark批式处理那样，需要前面的查询执行阶段执行完再执行后面的查询执行阶段，Presto采用的是流水线（Pipeline）处理机制。
- 任务（Task）：任务是Presto分布式任务的执行单元，每个查询执行阶段可以有多个任务，这些任务可以并行执行，同一个查询执行阶段中的所有任务的执行逻辑完全相同。一个查询执行阶段的任务个数就是此查询执行阶段的并发度。在Presto的任务调度代码中，可以看到任务的个数是根据查询执行阶段的数据分布方式（Source，Fixed，Single）以及查询执行节点的个数来决定的。

```java
// SqlQueryExecution.java
private void planDistribution(PlanRoot plan)
{
  	// 遍历执行计划PlanNode树，找到所有的TableScanNode（也就是连接器对应的PlanNode），获取到他们的SplitSource
    DistributedExecutionPlanner distributedPlanner = new DistributedExecutionPlanner(splitManager, metadata, dynamicFilterService);
    StageExecutionPlan outputStageExecutionPlan = distributedPlanner.plan(plan.getRoot(), stateMachine.getSession());

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
}
```



### 获取数据源分片

`SqlQueryExecution.planDistribution`首先从数据源连接器中获取到所有的分片数据源。分片是Presto中分块组织数据的方式，Presto连接器会将待处理的所有数据划分为若干分片让Presto读取，而这些分片也会被安排到多个Presto查询执行节点上来处理以实现分布式高性能计算。分布式OLAP引擎几乎全都有分片的抽象设计，例如Spark、Flink等。

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

以Tpcds数据源的实现为例，通过调用`ConnectorSplitManager.getSplits`可以获得ConnectorSplitSource，里面保存了所有的分片，也就是ConnectorSplit，最终被封装到StageExecutionPlan中。注意，这一步只是生成数据源分片，既不会将分片安排到某个Presto查询执行节点上，也不会真正使用分片读取连接器的数据。

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

### 创建SqlQueryScheduler

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

createSqlQueryScheduler会为执行计划的每个PlanFragment创建对应的SqlStageExecution。每个SqlStageExecution根据不同的数据分区类型（PartitioningHandle）可能对应不同的StageScheduler实现`io.prestosql.execution.scheduler.SqlQueryScheduler#createStages`。

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

StageScheduler的职责是绑定查询执行节点与上游数据源分片的关系，创建任务并调度到查询执行节点上。如果stage使用数据源连接器从存储系统拉取数据，这些stage的任务调度使用的是SourcePartitionedScheduler。

从数据源连接器那里获取一批分片，并准备调度这些分片。Presto的默认配置为每批最多调度1000个分片。FixedSplitSourc预先准备好所有的分片，Presto框架的SplitSource::getNextBlock每次会根据需要获取一批分片，FixedSplitSource根据需要的分片个数来返回。几乎所有的连接器都是用的FixedSplitSource，只有少数几个连接器（如Hive）实现了自己的ConnectorSplitSource。根据SplitPlacementPolicy为这一批分片挑选对应的节点，建立一个map，key是节点， value是分片列表。

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

}
```

生成任务并调度到Presto查询执行节点上，任务的调度需要先绑定节点和分片的关系，再绑定分片和任务的关系，再将任务调度到查询执行节点上。分片选择了哪些查询执行节点，那么查询执行节点就会创建任务，分片选择了多少查询执行节点，就会有多少个查询执行节点创建任务，这会影响查询执行阶段的并发度。创建任务即创建某个RemoteTask的实例化对象，Presto默认实现和使用的是HttpRemoteTask。

### FixedCountScheduler

```java
public class FixedCountScheduler
        implements StageScheduler
{
    public FixedCountScheduler(SqlStageExecution stage, List<InternalNode> partitionToNode)
    {
        requireNonNull(stage, "stage is null");
        this.taskScheduler = stage::scheduleTask;
        this.partitionToNode = requireNonNull(partitionToNode, "partitionToNode is null");
    }
  
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

`FixedCountScheduler`为每个分区分配了对应的节点，后续task会被发送到对应的节点处理对应的分区，`schedule`方法遍历所有的分区，调用taskScheduler生成对应的remoteTask，这里的taskScheduler实际上是`SqlStageExecution.scheduleTask`。

### 生成RemoteTask

查询执行阶段的数据来源有两种，一种是数据源连接器，一种是上游查询执行阶段的任务可以输出到OutputBuffer的数据。对于下游的查询执行阶段来说，上游查询执行阶段的任务可以称为数据上游任务（upstream source task）。这些数据上游任务是通过SqlStageExecution::addExchangeLocations注册到下游的SqlStageExecution中的，让下游查询执行阶段知道去哪里取数据。无论是哪一种数据源，Presto都统一抽象为ConnectorSplit，当上游查询执行阶段作为数据源时，Presto把它看做是一种特殊的连接器。它的catalog name = $remote，其实就是一个假的目录，ConnectorSplit的实现类是RemoteSplit。

```java
// SqlStageExecution.java
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

任务绑定分配到当前节点的分片之后，Presto会调用HttpRemoteTask.start，将任务分发到查询执行节点上

### 启动HttpRemoteTask

```java
// HttpRemoteTask
public void start()
{
    try (SetThreadName ignored = new SetThreadName("HttpRemoteTask-%s", taskId)) {
        // to start we just need to trigger an update
        scheduleUpdate();

        dynamicFiltersFetcher.start();
        taskStatusFetcher.start();
        taskInfoFetcher.start();
    }
}
```



构造了一个携带PlanFgragment和分片信息的HTTP Post请求，请求对应Presto查询执行节点的URI：`/v1/task/{taskId}`，Presto查询执行节点在收到请求后，会解析PlanFragment中的执行计划并创建SqlTaskExecution，开始执行任务。


