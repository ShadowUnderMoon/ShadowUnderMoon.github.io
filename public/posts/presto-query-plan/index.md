# Presto查询计划生成和优化


在`SqlQueryExecution.doPlanQuery`中，从原来的抽象语法树生成逻辑执行计划，然后优化执行计划，生成优化后的执行计划，最后将逻辑执行计划切分成多颗子树。

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



### 语义分析、生成执行计划，并优化执行计划

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

将逻辑执行计划拆分为多棵子树并生成subPlan的逻辑，这个过程用SimplePlanRewriter的实现类Fragmenter层层遍历上一步生成的PlanNode树，将其中的ExchangeNode[scope=REMOTE]替换为RemoteSourceNode，并且断开它与叶子节点的连接，这样一个PlanNode树就被划分成了两个PlanNode树，每棵树都是一个SubPlan。在查询执行的数据流转中，子树是父树的数据产出上游。

```java
public class SubPlan
{
    private final PlanFragment fragment;
    private final List<SubPlan> children;
}
```



```java
// io.prestosql.sql.planner.PlanFragmenter.Fragmenter#visitExchange
public PlanNode visitExchange(ExchangeNode exchange, RewriteContext<FragmentProperties> context)
{
    if (exchange.getScope() != REMOTE) {
        return context.defaultRewrite(exchange, context.get());
    }

    PartitioningScheme partitioningScheme = exchange.getPartitioningScheme();

    if (exchange.getType() == ExchangeNode.Type.GATHER) {
        context.get().setSingleNodeDistribution();
    }
    else if (exchange.getType() == ExchangeNode.Type.REPARTITION) {
        context.get().setDistribution(partitioningScheme.getPartitioning().getHandle(), metadata, session);
    }

    ImmutableList.Builder<SubPlan> builder = ImmutableList.builder();
    for (int sourceIndex = 0; sourceIndex < exchange.getSources().size(); sourceIndex++) {
        FragmentProperties childProperties = new FragmentProperties(partitioningScheme.translateOutputLayout(exchange.getInputs().get(sourceIndex)));
        builder.add(buildSubPlan(exchange.getSources().get(sourceIndex), childProperties, context));
    }

    List<SubPlan> children = builder.build();
    context.get().addChildren(children);

    List<PlanFragmentId> childrenIds = children.stream()
            .map(SubPlan::getFragment)
            .map(PlanFragment::getId)
            .collect(toImmutableList());

    return new RemoteSourceNode(exchange.getId(), childrenIds, exchange.getOutputSymbols(), exchange.getOrderingScheme(), exchange.getType());
}
```



```java
public RemoteSourceNode(
        @JsonProperty("id") PlanNodeId id,
        @JsonProperty("sourceFragmentIds") List<PlanFragmentId> sourceFragmentIds,
        @JsonProperty("outputs") List<Symbol> outputs,
        @JsonProperty("orderingScheme") Optional<OrderingScheme> orderingScheme,
        @JsonProperty("exchangeType") ExchangeNode.Type exchangeType)
{
    super(id);

    requireNonNull(outputs, "outputs is null");

    this.sourceFragmentIds = sourceFragmentIds;
    this.outputs = ImmutableList.copyOf(outputs);
    this.orderingScheme = requireNonNull(orderingScheme, "orderingScheme is null");
    this.exchangeType = requireNonNull(exchangeType, "exchangeType is null");
}
```

`RemoteSourceNode`保存了当前stage直接依赖的所有stage的PlanFragmentId。

