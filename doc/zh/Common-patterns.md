---
title: 常见 Topology 模式
layout: documentation
documentation: true
---

这一页列出了Strom topologies（拓扑）中的各种常见模式。

1. Batching
2. BasicBolt
3. In-memory caching + fields grouping combo (内存缓存+字段分组组合)
4. Streaming top N
5. TimeCacheMap for efficiently keeping a cache of things that have been recently updated(最近刚刚更新的 TimeCacheMap ，可以有效的存储缓存数据)
6. CoordinatedBolt and KeyedFairBolt for Distributed RPC(分布式RPC)


### Batching

通常情况下为了效率或者其他原因，你想成批处理一组元组而不是单独处理一组元组。例如，您希望对数据库进行批处理更新，或者进行某种类型的流聚合。

如果您希望在数据处理中具有可靠性，那么正确的方法是在实例等待 bolt 进行处理时保留实例变量中的元组。完成批处理操作后，再确认所持有的所有元组。

如果bolt 发出元组，那么您可能会想使用多锚来确保可靠性。这要看具体的应用程序而定。有关可靠性如何工作的详细信息，请参见[保证消息处理](Guaranteeing-message-processing.html)。

### BasicBolt

许多bolts 遵循类似的阅读输入元组模式，基于输入元组发射零个或多个元组，然后在执行方法结束时会立即确认输入元组。与此模式匹配的Bolts 类似功能和过滤器。这是一个通用模式，storm会为你暴露一个称为[IBasicBolt](javadocs/org/apache/storm/topology/IBasicBolt.html) 接口 ，自动化模式。更多信息见[保证消息处理](Guaranteeing-message-processing.html)。

### In-memory caching + fields grouping combo(在内存中缓存+字段分组组合)

在Storm bolts 中使用内存缓存很常见。当把它与fields grouping 相结合时，缓存会变得特别强大。例如，假设你有一个bolt ，用于把短网址（如bit.ly，t.co，等）扩展为长的网址。你可以通过保存短URL的LRU，来缓存长URL的扩展 避免做同样的HTTP请求 从而提高性能。假设组件“URLS”发出短URLS，组件“expand ”将短URL扩展为长URL并在内部保留缓存。考虑下面两段代码之间的区别：

```java
builder.setBolt("expand", new ExpandUrl(), parallelism)
  .shuffleGrouping(1);
```

```java
builder.setBolt("expand", new ExpandUrl(), parallelism)
  .fieldsGrouping("urls", new Fields("url"));
```

第二种方法将有更有效的缓存，因为相同的URL将始终指向相同的任务。这避免了任务中缓存的重复，使得短URL更可能命中缓存。

### Streaming top N

一个常见的连续计算Storm 是通过“streaming top N ”的某种排序来实现。假设有一个bolt ，它会发射这种形式的元组["value", "count"] ，并且您希望有一个基于顶部N元组的bolt来计数 。要做到这一点，最简单的方法是有一个在流上执行全局组的bolt，并在内存中保存一个top N items列表。

显然，这种方法不能在较大的流中应用，因为整个流必须完成一个任务。一个更好的计算方法是在流的分区上并行地执行上面的多个N‘s，然后合并上面的N‘s 个来得到全局的顶部N：

```java
builder.setBolt("rank", new RankObjects(), parallelism)
  .fieldsGrouping("objects", new Fields("value"));
builder.setBolt("merge", new MergeObjects())
  .globalGrouping("rank");
```

这种模式之所以有效，是因为第一个bolt 所做的字段分组，使您在语义上需要正确的区分。你可以在storm-starter[这里]({{page.git-blob-base}}/examples/storm-starter/src/jvm/org/apache/storm/starter/RollingTopWords.java) 中 看到这个案例。

然而如果你想处理已知的数据倾斜问题，可以使用partialkeygrouping代替fieldsgrouping 。他将分配两个downstream bolts 来分布式负载以替代个使用单独的一个。

```java
builder.setBolt("count", new CountObjects(), parallelism)
  .partialKeyGrouping("objects", new Fields("value"));
builder.setBolt("rank" new AggregateCountsAndRank(), parallelism)
  .fieldsGrouping("count", new Fields("key"))
builder.setBolt("merge", new MergeRanksObjects())
  .globalGrouping("rank");
``` 

topology 需要一个额外的处理层来聚合来自上游螺栓的部分计数，但现在只处理聚合值，所以bolt 不受数据倾斜引起的负载的影响。您可以在storm-starter[这里]({{page.git-blob-base}}/examples/storm-starter/src/jvm/org/apache/storm/starter/SkewedRollingTopWords.java)  中看到这种模式的示例。

### TimeCacheMap for efficiently keeping a cache of things that have been recently updated （最近刚刚更新的 TimeCacheMap ，可以有效的存储缓存数据）


有时你想在最近活动的项目中保留一个缓存，并且让那些已经有一段时间没用的的条目自动过期。timecachemap就是比较试用这个场景的数据结构，他提供了一种方式，可以在当你需要让条目过期时添加回调函数。

### CoordinatedBolt and KeyedFairBolt for Distributed RPC（分布式RPC coordinatedbolt和keyedfairbolt）

当在storm顶部构建分布式RPC应用程序时，通常需要两种常见模式。这些都是由在Storm codebase 中的“standard library”  的[CoordinatedBolt](javadocs/org/apache/storm/task/CoordinatedBolt.html)和[KeyedFairBolt](javadocs/org/apache/storm/task/KeyedFairBolt.html) 封装的代码 。

当你的bolt接收到任何给定请求的所有元组时，`CoordinatedBolt`封装了包含你的逻辑以及算法的bolt 。 它大量使用direct streams 来做到这一点。

`CoordinatedBolt`也封装了包含逻辑的bolt ，并确保您的topology 同时处理多个DRPC调用，而不是一次一次连续的执行。

有关详细信息，请参阅[分布式 RPC](Distributed-RPC.html)。

