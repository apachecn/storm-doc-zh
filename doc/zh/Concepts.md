---
title: 概念
layout: documentation
documentation: true
---

本页列出了Storm的主要概念，以及可以获取到更多信息的资源链接，概念如下：

1. 拓扑
2. 流
3. Spouts
4. Bolts
5. 流分组
6. 可靠性
7. Tasks
8. Workers

### 拓扑

实时应用程序的逻辑被封装在 Storm 拓扑中。 Storm 拓扑类似于 MapReduce 作业。 一个关键的区别是 MapReduce 作业最终会完成，而拓扑永远运行（除非 kill 掉它）。 
一个拓扑是 Spout 和 Bolt 通过流分组连接起来的有向无环图。 这些概念会在下面的段落中具体描述。

**相关资料:**

* [TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html): 使用这个类来构建拓扑
* [如何在生产集群上运行拓扑](Running-topologies-on-a-production-cluster.html)
* [如何使用local模式](Local-mode.html): 学习如何用 local 模式开发和测试

### 流

流是 Storm 中的核心概念。一个流是一个无界的、以分布式方式并行创建和处理的 Tuple 序列。 流以一个 schema 来定义，这个 schema 命名了流中
的元组中的字段。默认情况下 Tuple 可以是 integers, longs, shorts, bytes, strings, doubles, floats, booleans, and byte arrays 
等数据类型。也可以自定义序列化器，以在 Tuple 中使用自定义的类型。

每一个流在声明的时候会有一个赋予一个ID。由于单流的 Spout 和 Bolt 比较常见, [OutputFieldsDeclarer]
(javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html) 有更便捷的方法定义一个单流而不用指定ID。这种情况下流
会被赋予一个默认的ID，"default"。


**相关资料:**

* [Tuple](javadocs/org/apache/storm/tuple/Tuple.html): 流由一系列连续的 Tuple 组成
* [OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html): 用于定义流和它的 schema
* [Serialization](Serialization.html): 动态类型的 Tuple 和自定义序列化的相关信息

### Spouts

Spout 是一个拓扑中的流的源头。 通常 Spout 会从外部数据源（如 Kestel 队列，或者 Twitter API）读取 Tuple 然后把他们发射到拓扑。Spout 
可以是 __可靠的__ 或 __不可靠的__。可靠的 Spout 在 Tuple 处理失败的时候能够重放， 不可靠的 Spout 一旦把一个 Tuple 发射出去就撒手不管了。

Spout 可以发射多个流。可以使用[OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html) 类的 
declareStream 方法定义多个流，然后在[SpoutOutputCollector](javadocs/org/apache/storm/spout/SpoutOutputCollector.html)
对象的emit方法中指定要发送到的目标流。


Spout 中的最重要的方法是 `nextTuple`。 `nextTuple` 要么向拓扑中发射一个新的 Tuple，要么在没有 Tuple 需要发射的情况下直接返回。 对于任何 Spout 实现，`nextTuple` 
方法都必须非阻塞的, 因为 Spout 的所有方法都是在同一个线程中调用。

Spout的另外几个重要的方法是 `ack` 和 `fail` 。 这些方法在 Storm 检测到 Spout 发送出去的 Tuple 被成功处理或者处理失败的时候调用。`ack`和` fail`只会在可靠的 Spout 中调用。 
更多相关信息，请参见 
[the 
Javadoc](javadocs/org/apache/storm/spout/ISpout.html) 。

**相关资料:**

* [IRichSpout](javadocs/org/apache/storm/topology/IRichSpout.html): 创建Spout时必须实现的接口
* [Guaranteeing message processing](Guaranteeing-message-processing.html)

### Bolts

拓扑中所有的业务处理都在 Bolt 中完成。Bolt 可以通过过滤，函数，聚合，关联，与数据库交互等等手段来做任何事情。

Bolt 可以做简单流转换。复杂的流转换一般需要多个步骤因此也就要多个 Bolt 协同工作。 如，转换一个 tweets 流为一个 trending 
images流需要两个步骤：一个 Bolt 做每个图片的滚动计数同时一个或者多个 Bolt 输出 Top X 的图片 (可以使用更具弹性的方式，用3个 Bolt 而不是先前的2个 Bolt，来完成这个特殊的流转换)。 

Bolt 可以发射一个或者多个流。使用[OutputFieldsDeclarer](javadocs/org/apache/storm/topology/OutputFieldsDeclarer.html)
的`declareStream`方法定义多个并，并且在 [OutputCollector]
(javadocs/org/apache/storm/task/OutputCollector.html)的`emit`中指定需要发射到的目标流。

当定义一个 Bolt 的输入流, 一定要订阅另一个组件的特定的流。如果想订阅另一个组件的所有流，必须分别单独订阅每一个流。 [InputDeclarer]
(javadocs/org/apache/storm/topology/InputDeclarer.html) 有订阅使用默认id定义的流的语法糖。`declarer.shuffleGrouping
("1")` 订阅组件 "1" 的默认流，等价于 `declarer.shuffleGrouping("1", DEFAULT_STREAM_ID)`。

Bolt 中最重要的方法是`execute` 方法，当有一个新 Tuple 输入的时候会进入这个方法。Bolt 使用[OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html) 对象发射新 Tuple。Bolt
必须在每一个 Tuple 处理完以后调用`OutputCollector`上的`ack`方法，来告诉Strom的拓扑 Tuple 已经处理完成 (最终可以确定
原始的 Spout Tuple). 对于处理输入 Tuple 的一般情形：发射基于这个 Tuple 的零或者多个 Tuple 并且 ack 这个 Tuple, Storm 提供了
[IBasicBolt](javadocs/org/apache/storm/topology/IBasicBolt.html) 接口可以自动调用ack方法。

在 Bolt 中启动新线程执行异步处理是非常好的。 [OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html) 是线程安全的，并且可以在任何时机调用。

**相关资料:**

* [IRichBolt](javadocs/org/apache/storm/topology/IRichBolt.html): Bolt 的通用接口
* [IBasicBolt](javadocs/org/apache/storm/topology/IBasicBolt.html): 比较便捷的定义一个过滤或者一般功能的 Bolt 的接口
* [OutputCollector](javadocs/org/apache/storm/task/OutputCollector.html): Bolts使用这个类的实例提交 Tuple 到他们的输出流
* [Guaranteeing message processing](Guaranteeing-message-processing.html)

### 流分组

拓扑定义的一部分，指定 Bolt 把哪个流作为自己的输入。 一个流分组定义了流如何在 Bolt 的 task 之间分区。

一共有8个内置的 Stream Grouping。可以通过实现接口[CustomStreamGrouping](javadocs/org/apache/storm/grouping/CustomStreamGrouping.html)
接口来自定义流分区。

1. **Shuffle grouping**: Tuple 随机的分发到 Bolt Task，每个 Bolt 获取等量的 Tuple。
2. **Fields grouping**: 流通过分组指定的字段来分区。例如流通过 ”user-id” 字段分区，具有相同 user-id 的 Tuple 会发送到同一个task，具有不同 user-id
的 Tuple 可能会流入到不同的 task。
3. **Partial Key grouping**: 流通过grouping中指定的 field 来分组，与 Fields 
Grouping 相似。但是对于2个下行流 Bolt 来说是负载均衡的，可以在输入数据不平均的情况下提供更好的优化。
以下地址[This paper](https://melmeric.files.wordpress
.com/2014/11/the-power-of-both-choices-practical-load-balancing-for-distributed-stream-processing-engines.pdf) 
更好的解释了它是如何工作的及它的优势。
4. **All grouping**: 流在所有的 Bolt Tasks之间复制。小心使用。
5. **Global grouping**: 整个流会进入一个其中的一个 Bolt task。特别指出，它会进入 id 最小的 task。
6. **None grouping**: 这个分组模式下，你不需要关心流如何分组。当前，None grouping 和 Shuffle grouping 等价。最终, 使用 None Grouping 的下行 Bolt 
会在他们订阅消息的上游 Bolt 的相同的线程中运行。
(when possible)。
7. **Direct grouping**: 这是一种特殊的分组方式. 一个流用这个方式分组意味着由这个 Tuple 的 __生产者__ 来决定哪个任务来接收它。 直接分组只能被用于定义为直接流的流上。 被发射到直接流的 tuple 
必须使用 [emitDirect](javadocs/org/apache/storm/task/OutputCollector.html#emitDirect(int, int, java.util.List) 方法来发射。
Bolt 可以使用[TopologyContext](javadocs/org/apache/storm/task/TopologyContext.html) 或者通过保持对[OutputCollector]
(javadocs/org/apache/storm/task/OutputCollector.html)中的`emit` 方法的输出的跟踪来获取它的所有消费者的 ID (返回 Tuple 被发送到的目标 task的id)。
8. **Local or shuffle grouping**: 如果目标 Bolt 有多个 task 在同一个 woker 进程中，Tuple 会将消息打散分发到同进程内的任务。否则，和 shuffle goruping 一样。

**相关资料:**

* [TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html): 使用这个类来定义一个拓扑
* [InputDeclarer](javadocs/org/apache/storm/topology/InputDeclarer.html): 
当在`TopologyBuilder`上调用`setBolt`方法的时候返回这个对象，用于定义一个 Bolt 的输入流以及这些流如何分组
### 可靠性

Storm 保障每一个 Spout 的 Tuple 都会被拓扑完全处理。 它通过每一个 Spout 的 Tuple 触发生成 Tuple 跟踪树，通过确定 Tuple 
树何时被成功处理完成来实现可靠性。每一个拓扑都有一个关联的“message 
timeout”。如果Storm检测到一个 Spout Tuple 没有在这个超时时间内被处理完成，则判定这个 Tuple 失败，并且重新发送。

要利用这个可靠性的功能, 必须告诉 Storm 何时在 Tuple 树中创建一个新边界，并且需要在 Tuple 被完全处理完成的时候通知 Storm。 以上操作在 Bolt用于发射 Tuple 的 [OutputCollector]对象的
(javadocs/org/apache/storm/task/OutputCollector.html) 对象来完成这个操作。 锚点在 
`emit` 方法中完成, 使用`ack`方法来声明自己已经成功完成了一个 Tuple 的处理。

更详细的解释 [Guaranteeing message processing](Guaranteeing-message-processing.html)。

### Tasks

每个 Spout 或者 Bolt 都以跨集群的多个 Task 方式执行。 每个 Task 对应一个 execution 的线程，流分组定义如何从一个 Task 发送 Tuple 到另一个 Task。 
可以在方法`setSpout`或者`setBolt`中为每个 Spout 或者 Bolt 设置并行度，
### Workers

一个拓扑可以在一个或者跨多个 worker 执行。每个 Worker 进程是一个物理的 JVM，执行各个拓扑的所有 Tasks 的一个子集。
例如，如果一个拓扑的并行度是300，共有50个 Worker 在运行，每个 Worker 会分配到6个 Task（作为 Worker 进程中的线程）。
Storm 会尽量把所有 Task 均匀的分配到所有的 Worker 上。

**相关资料:**

* [Config.TOPOLOGY_WORKERS](javadocs/org/apache/storm/Config.html#TOPOLOGY_WORKERS): 这个配置项设置用于运行拓扑的 worker 数量。
