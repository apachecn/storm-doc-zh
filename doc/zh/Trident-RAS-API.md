---
title: Trident RAS API
layout: documentation
documentation: true
---

## Trident RAS API

Trident RAS（ Resource Aware Scheduler （资源感知调度程序））API 提供了一种机制, 允许用户指定 Trident topology 的 resource consumption （资源消耗）.  API 看起来与基本的 RAS API 完全相同, 只是在 Trident Streams 上调用, 而不是 Bolts 和 Spouts . 

为了避免文档中的 duplication （重复）和 inconsistency （不一致）,  resource setting （资源设置）的目的和效果在这里不再描述, 而是在 [Resource Aware Scheduler Overview](Resource_Aware_Scheduler_overview.html) 中可以找到,

### Use

首先, 例如:

```java
    TridentTopology topo = new TridentTopology();
    topo.setResourceDefaults(new DefaultResourceDeclarer();
                                                          .setMemoryLoad(128)
                                                          .setCPULoad(20));
    TridentState wordCounts =
        topology
            .newStream("words", feeder)
            .parallelismHint(5)
            .setCPULoad(20)
            .setMemoryLoad(512,256)
            .each( new Fields("sentence"),  new Split(), new Fields("word"))
            .setCPULoad(10)
            .setMemoryLoad(512)
            .each(new Fields("word"), new BangAdder(), new Fields("word!"))
            .parallelismHint(10)
            .setCPULoad(50)
            .setMemoryLoad(1024)
            .each(new Fields("word!"), new QMarkAdder(), new Fields("word!?"))
            .groupBy(new Fields("word!"))
            .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))
            .setCPULoad(100)
            .setMemoryLoad(2048);
```

可以为每个操作设置 Resources （资源）（除了 grouping （分组）, shuffling （混洗）, partitioning （分区））. 将 Trident combined 成 single Bolts 的操作将将其资源相加. 

每个 Bolt 都被给予 **至少** 默认资源, 无论用户设置如何. 

在上述情况下, 我们最终得到

- 一个 spout 和 spout coordinator （spout 协调器）, 每个 CPU 负载为 20% , 内存负载为 512MiB , off-heap （堆栈）为 256MiB . 
- 组合的 `Split` 和 `BangAdder` 以及 `QMarkAdder` 的 on-heap （堆栈）中, 具有80% cpu 负载（10%+ 50%+ 20%）和 1664MiB（1024 + 512 + 128）的内存负载的 bolt , 使用 DefaultResourceDeclarer 中包含的默认资源
- 具有 100% cpu 加载和 2048MiB 堆内存负载的 bolt , 默认值为 off-heap （非堆）

任何 operation （操作）后都可以调用 Resource declarations （资源声明）. 没有明确资源的操作将获得默认值. 如果您选择仅为某些操作设置资源, 则必须声明默认值, 否则拓扑提交将失败. 
资源声明具有与并行提示相同的 *boundaries* . 他们不会进行任何分组, 洗牌或任何其他类型的重新分配. 
每个操作都会声明资源, 但在边界内组合. 
