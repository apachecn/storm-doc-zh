---
title: Configuration
layout: documentation
documentation: true
---

Storm 具有各种各样的配置，用于调整 nimbus, supervisors 和运行 topologies（拓扑）的行为.
一些配置是系统配置, 不能在拓扑基础上对拓扑进行修改, 而每个拓扑可以修改其他配置.

每个配置都有一个定义在 Storm 代码库 [defaults.yaml]({{page.git-blob-base}}/conf/defaults.yaml) 文件中的 default value（默认值）.
您可以通过在 Nimbus 按 supervious 的 classpath 中定义一个 storm.yaml 文件以覆盖那些配置.
最后, 您可以定义一个指定的 topology 配置, 它在使用 [StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html) 时随着你的 topology 一起提交.
然而, 指定的 topology 配置只能够覆盖前缀 "TOPOLOGY" 开始的配置.

Storm 0.7.0 及以上版本允许您以 per-bolt/per-spout 为基础进行配置覆盖.
可以通过这种方式覆盖的唯一配置是:

1. "topology.debug"
2. "topology.max.spout.pending"
3. "topology.max.task.parallelism"
4. "topology.kryo.register": 这与其他工作有点不同, 因为 serializations（序列化将）可用于拓扑中的所有组件. 更多细节请参阅 [序列化](Serialization.html). 

Java API 允许您以两种方式指定组件特定的配置:

1. *Internally（内部的）:* 在任何 spout 或 bolt 中覆盖 `getComponentConfiguration` 方法, 并返回组件指定的的配置 map.
2. *Externally（外部的）:* `TopSpringBuilder` 中的 `setSpout` 和 `setBolt` 返回一个可以用来覆盖组件配置的 `addConfiguration` 和`addConfigurations` 方法的对象.

配置值的优先顺序是 defaults.yaml < storm.yaml < topology 指定的配置 < internal（内部）组件指定的配置 < external（外部）组件指定的配置. 


**Resources:**

* [Config](javadocs/org/apache/storm/Config.html): 所有配置的列表以及用于创建拓扑特定配置的辅助 class（类）
* [defaults.yaml]({{page.git-blob-base}}/conf/defaults.yaml): 所有配置的默认值
* [部署 Storm 集群](Setting-up-a-Storm-cluster.html): 介绍如何创建和配置 Storm 集群
* [在生产集上运行 topologies（拓扑）](Running-topologies-on-a-production-cluster.html): 在集群上运行拓扑时列出了有用的配置
* [Local（本地）模式](Local-mode.html): 列出使用 local model（本地模式）时的有用配置
