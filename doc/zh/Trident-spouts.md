---
title: Trident Spouts
layout: documentation
documentation: true
---
# Trident spouts

像在 vanilla Storm API 中，spouts 是 Trident topology 中的 source of streams （streams 的来源）。在 vanilla Storm spouts 之上，Trident 为更复杂的 spouts 提供了额外的 API 。

您的 how you source your data streams （数据流的来源）和 how you update state (e.g. databases) based on those data streams （基于这些数据流更新状态（例如数据库））之间存在着一个不可分割的联系。请参阅 [Trident state doc](Trident-state.html) ，以了解此信息 - 了解此联系对于了解可用的 spout 选项是至关重要的。

Regular Storm spouts 将是 Trident topology 中的 non-transactional spouts 。要使用 regular Storm IRichSpout，请在 TridentTopology 中创建如下 stream :

```java
TridentTopology topology = new TridentTopology();
topology.newStream("myspoutid", new MyRichSpout());
```

Trident topology 中的所有 spouts 都需要为该流提供 unique identifier （唯一的标识符） - 该 identifier 在集群上运行的所有 topologies 中必须是唯一的。 Trident 将使用此 identifier 来存储 Zookeeper 中 spout 消耗的 metadata （元数据），包括 txid 和与 spout 相关联的任何 metadata （元数据）。

您可以通过以下配置选项配置 spout metadata 的 Zookeeper 存储:

1. `transactional.zookeeper.servers`: Zookeeper hostnames 列表
2. `transactional.zookeeper.port`: Zookeeper 集群的端口
3. `transactional.zookeeper.root`: Zookeeper 中存储 Metadata 的根目录。 Metadata 将存储在路径 <root path>/<spout id>

## Pipelining

默认情况下，Trident 一次处理 single batch （一个批次），等待批次成功或失败，然后再尝试处理 another batch （另一批次）。您可以通过 pipelining the batches 获得 significantly higher throughput （明显更高的吞吐量）并降低每个 batch （批处理）的延迟。您可以使用 "topology.max.spout.pending" 属性同时处理要处理的 maximum amount of batches （最大批处理量）。

即使在同时处理 multiple batches （多个批次）的同时，Trident 也会在 topology 中将任何 state updates 按照 batches 的顺序排序。例如，假设您正在对数据库进行全局计数聚合。这个想法是，当您更新数据库中 batch 1 的计数时，您仍然可以计算 batch 2 到 10 的部分计数。 Trident 将不会移动到 batch 2 的 state updates （状态更新），直到状态更新为 batch 1 已成功。这对于实现 exactly-once （一次且仅一次）处理语义非常重要，如 [Trident state doc](Trident-state.html) 中的大纲。

## Trident spout types

以下是可用的以下 spout API:

1. [ITridentSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/ITridentSpout.java): 最通用的 API ，可以支持 transactional 或 opaque transactional semantics （语义）。一般来说，您将直接使用此 API 的一种 partitioned （分区）风格，而不是使用此 API 。
2. [IBatchSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/IBatchSpout.java): 一次发送 batches of tuples 的 non-transactional spout 。
3. [IPartitionedTridentSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/IPartitionedTridentSpout.java): 从 partitioned data source （分区数据源）读取的 transactional spout （像 Kafka 服务器集群）
4. [IOpaquePartitionedTridentSpout]({{page.git-blob-base}}/storm-core/src/jvm/org/apache/storm/trident/spout/IOpaquePartitionedTridentSpout.java): 一个从 partitioned data source （分区数据源）读取的 opaque transactional spout 。

而且，像本教程开头所提到的，你也可以使用常规的 IRichSpout 。
 

