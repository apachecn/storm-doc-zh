---
title: Scheduler（调度器）
layout: documentation
documentation: true
---

Storm 现在有 4 种内置的 schedulers（调度器）: [DefaultScheduler]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/scheduler/DefaultScheduler.clj), [IsolationScheduler]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/scheduler/IsolationScheduler.clj), [MultitenantScheduler]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/scheduler/multitenant/MultitenantScheduler.java), [ResourceAwareScheduler](Resource_Aware_Scheduler_overview.html). 

## Pluggable scheduler（可插拔的调度器）
您可以实现自己的 scheduler（调度器）来替换 default scheduler（默认调度器）以将 executors（执行器）分配给 workers.
您在 storm.yaml 配置文件中使用 "storm.scheduler" 配置来配置您的 class, 并且您的 scheduler（调度器）必须实现 [IScheduler]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/scheduler/IScheduler.java) 接口。

## Isolation Scheduler（隔离调度器）
该 isolation scheduler（隔离调度器）使得在许多 topologies（拓扑）中共享集群变得容易和安全.
isolation scheduler（隔离调度器）序允许您指定哪些 topologies（拓扑）应该是 "isolated（隔离的）", 这意味着它们在集群中的一组专用机器上运行, 而不会运行其他 topologies（拓扑）.
这些在集群上被隔离的 topologies（拓扑）被赋予优先级, 因此如果与非隔离拓扑结构存在竞争, 资源将被分配给隔离拓扑, 如果需要为非隔离拓扑获取资源, 资源将被从非隔离拓扑中移除.
一旦分配了所有隔离拓扑, 群集中的其余机器将在所有非隔离拓扑中共享.

您可以通过将 "storm.scheduler" 设置为 "org.apache.storm.scheduler.IsolationScheduler" 以在 Nimbus 配置中配置 Isolation Scheduler（隔离调度器）.
然后, 使用 "isolation.scheduler.machines" 配置来指定每个拓扑应该获得多少台机器.
此配置是从拓扑名称到分配给此拓扑的隔离机器数量的映射.
例如:

```
isolation.scheduler.machines: 
    "my-topology": 8
    "tiny-topology": 1
    "some-other-topology": 3
```

提交给未列出的集群的任何拓扑将不会被隔离.
请注意, Storm 用户无法影响其隔离设置 - 这只是允许集群的管理员来操作（这是故意如此设计的）

该 isolation scheduler（隔离调度器）通过在拓扑之间提供完全的隔离来解决多租户问题 - 避免拓扑之间的资源争用.
意图是 "productionized（生产的）" 拓扑应该列在隔离配置中, 测试或开发中的拓扑不应该.
集群上的剩余机器可以为隔离拓扑提供故障切换和运行非隔离拓扑的双重角色.

