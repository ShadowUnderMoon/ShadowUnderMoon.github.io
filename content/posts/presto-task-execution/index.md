---
title: "Presto任务执行流程"
author: "爱吃芒果"
description:
date: "2026-01-25T21:37:20+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
- Presto
- Trino
categories:
- Presto
- Trino
---

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



