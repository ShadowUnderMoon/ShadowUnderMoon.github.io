# Presto任务执行流程


```java
// TaskResouce /v1/task/{taskId}
public Response createOrUpdateTask(@PathParam("taskId") TaskId taskId, TaskUpdateRequest taskUpdateRequest, @Context UriInfo uriInfo)
{
    requireNonNull(taskUpdateRequest, "taskUpdateRequest is null");

    Session session = taskUpdateRequest.getSession().toSession(sessionPropertyManager, taskUpdateRequest.getExtraCredentials());
    TaskInfo taskInfo = taskManager.updateTask(session,
            taskId,
            taskUpdateRequest.getFragment(),
            taskUpdateRequest.getSources(),
            taskUpdateRequest.getOutputIds(),
            taskUpdateRequest.getTotalPartitions());

    if (shouldSummarize(uriInfo)) {
        taskInfo = taskInfo.summarize();
    }

    return Response.ok().entity(taskInfo).build();
}
```




