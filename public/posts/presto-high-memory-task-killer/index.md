# Presto worker高负载查杀任务


为了尽量避免worker发生OOM从而短暂不可用，Presto提供了在内存高负载下查杀任务的机制，代码很简单，在`com.facebook.presto.memory.HighMemoryTaskKiller`中。

## 如何判断内存高负载

Presto通过监听gc事件，通过fullgc的内存回收情况以及发生频率来判断worker是否处于高负载状态，并提供了两种策略：

- FREE_MEMORY_ON_FREQUENT_FULL_GC 两次fullgc间隔小于1s且两次fullgc回收的内存均没有超过xmx的1%
- FREE_MEMORY_ON_FULL_GC 当前内存占用超过90%且fullgc回收的内存小于xmx的1%

以上策略中的参数都可以配置

```java
private boolean shouldTriggerTaskKiller(GarbageCollectionNotificationInfo info)
{
    boolean triggerTaskKiller = false;
    DataSize beforeGcDataSize = info.getBeforeGcTotal();
    DataSize afterGcDataSize = info.getAfterGcTotal();

    if (taskKillerStrategy == FREE_MEMORY_ON_FREQUENT_FULL_GC) {
        long currentGarbageCollectedBytes = beforeGcDataSize.toBytes() - afterGcDataSize.toBytes();
        Duration currentFullGCTimestamp = new Duration(ticker.read(), TimeUnit.NANOSECONDS);

        if (isFrequentFullGC(lastFullGCTimestamp, currentFullGCTimestamp) && !hasFullGCFreedEnoughBytes(currentGarbageCollectedBytes)) {
            triggerTaskKiller = true;
        }

        lastFullGCTimestamp = currentFullGCTimestamp;
        lastFullGCCollectedBytes = currentGarbageCollectedBytes;
    }
    else if (taskKillerStrategy == FREE_MEMORY_ON_FULL_GC) {
        if (isLowMemory() && beforeGcDataSize.toBytes() - afterGcDataSize.toBytes() < reclaimMemoryThreshold) {
            triggerTaskKiller = true;
        }
    }
    log.debug("Task Killer Trigger: " + triggerTaskKiller + ", Before Full GC Head Size: " + beforeGcDataSize.toBytes() + " After Full GC Heap Size: " + afterGcDataSize.toBytes());

    return triggerTaskKiller;
}
```



```java
private boolean isLowMemory()
{
    MemoryUsage memoryUsage = memoryMXBean.getHeapMemoryUsage();

    if (memoryUsage.getUsed() > heapMemoryThreshold) {
        return true;
    }

    return false;
}
```

感觉这里没有必要用`memoryMXBean`重新获取当前内存占用并判断是否高负载，完全可以使用`afterGcDataSize.toBytes()`代替。

## 高负载后如何恢复

Presto选择杀掉当前worker上占用内存最高的查询，这里的内存占用是说它上报的内存占用，而非实际的内存占用，实际的内存占用除非打dump否则很难获得。

```java
private void onGCNotification(Notification notification)
{
    if (GC_NOTIFICATION_TYPE.equals(notification.getType())) {
        GarbageCollectionNotificationInfo info = new GarbageCollectionNotificationInfo((CompositeData) notification.getUserData());
        if (info.isMajorGc()) {
            if (shouldTriggerTaskKiller(info)) {
                //Kill task consuming most memory
                List<SqlTask> activeTasks = getActiveTasks();
                ListMultimap<QueryId, SqlTask> activeQueriesToTasksMap = activeTasks.stream()
                        .collect(toImmutableListMultimap(task -> task.getQueryContext().getQueryId(), Function.identity()));

                Optional<QueryId> queryId = getMaxMemoryConsumingQuery(activeQueriesToTasksMap);

                if (queryId.isPresent()) {
                    List<SqlTask> activeTasksToKill = activeQueriesToTasksMap.get(queryId.get());
                    for (SqlTask sqlTask : activeTasksToKill) {
                        TaskStats taskStats = sqlTask.getTaskInfo().getStats();
                        sqlTask.failed(new PrestoException(EXCEEDED_HEAP_MEMORY_LIMIT, format("Worker heap memory limit exceeded: User Memory: %d, System Memory: %d, Revocable Memory: %d", taskStats.getUserMemoryReservationInBytes(), taskStats.getSystemMemoryReservationInBytes(), taskStats.getRevocableMemoryReservationInBytes())));
                    }
                }
            }
        }
    }
}
```








