---
    title: Scheduler（调度器）
layout: documentation
documentation: true
---

Storm 现在有 4 种内置的 schedulers（调度器）: [DefaultScheduler]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/scheduler/DefaultScheduler.clj), [IsolationScheduler]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/scheduler/IsolationScheduler.clj), [MultitenantScheduler]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/scheduler/multitenant/MultitenantScheduler.java), [ResourceAwareScheduler](Resource_Aware_Scheduler_overview.html). 

## Pluggable scheduler（可插拔的调度器）
你可以实现你自己的 scheduler（调度器）来替换掉默认的 scheduler（调度器），自定义分配executors  到 workers 的调度算法.
使用的时候，在storm.yaml 文件中将 "storm.scheduler" 配置属性设置成你的class类, 并且您的 scheduler（调度器）必须实现 [IScheduler]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/scheduler/IScheduler.java) 接口。

## Isolation Scheduler（隔离调度器）
solation scheduler（隔离调度器） 使得多个topologies（拓扑） 共享集群资源更加容易和安全.
isolation scheduler（隔离调度器） 允许你指定某些 topologies（拓扑）是 “isolated”（隔离的），
这意味着这些  solated topologies（隔离的拓扑）运行在集群的特定机器上，这些机器没有其他 topologies（拓扑）运行。
isolated topologies（隔离的拓扑） 具有高优先级，所以如果和 non-isolated  topologies（非隔离的拓扑）竞争资源，资源将会分配给 isolated topologies（隔离的拓扑），
如果必须给 isolated topologies（隔离的拓扑）分配资源，那么将会从 non-isolated topologies（非隔离的拓扑）中抽取资源。一旦所有 isolated topologies （隔离的拓扑）所需资源得到满足，
那么集群中剩下的机器将会被 non-isolated topologies（非隔离的拓扑）共享。

您可以通过将 "storm.scheduler" 设置为 "org.apache.storm.scheduler.IsolationScheduler" ， 这样 Nimbus 节点的 Scheduler 就配置为 Isolation Scheduler（隔离调度器）.
然后, 使用 "isolation.scheduler.machines" 配置来指定每个topology（拓扑） 分配多少台机器.
这个配置是从 topology name（拓扑名称）到分配给此 topology（拓扑）的隔离机器数量的 map 集合.
例如:

```
isolation.scheduler.machines: 
    "my-topology": 8
    "tiny-topology": 1
    "some-other-topology": 3
```

提交到集群中的topologies 如果没有出现在上述 map 集合中，那么将不会被 isolated 。
请注意：user不可以设置 isolation 属性,该配置只能通过集群的管理员分配（这是故意这样设计的）。

isolation scheduler（隔离调度器）通过在拓扑之间提供完全的隔离来解决多租户问题 - 避免 topologies（拓扑）之间的资源竞争问题.
最终的目的是 "productionized（生产黄精的）" topologies（拓扑）应该设置成 isolated , 测试或开发中的 topologies（拓扑）不应该设置成 isolated 属性.
集群上的剩余机器可以为 isolated topologies（隔离的拓扑）提供故障切换，也可以用来运行 non-isolated topologies（非隔离的拓扑）.

