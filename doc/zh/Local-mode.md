---
title: 本地模式
layout: documentation
documentation: true
---
本地模式是一种在本地进程中模拟Storm集群的工作模式,对开发和测试 topologies（拓扑） 非常有用.本地模式运行 topologies（拓扑）和在[集群上](Running-topologies-on-a-production-cluster.html)运行 topologies 一样。

创建一个进程内的集群，只需要使用 LocalCluster 类. 例如:

```java
import org.apache.storm.LocalCluster;

LocalCluster cluster = new LocalCluster();
```

然后,您可以使用 `LocalCluster` 对象的 `submitTopology` 方法提交topologies（拓扑）,和 [StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html)中的一些方法相似, `submitTopology` 以 拓扑名称，topology configuration，和 topology 对象作为参数. 你可以使用 `killTopology` 方法 kill a topology,killTopology 方法接受 topology name 为参数
使用以下语句关闭本地模式集群：

```java
cluster.shutdown();
```

### 本地模式的常用配置
您可以在这里看到所有的配置 [config列表](javadocs/org/apache/storm/Config.html).

1. **Config.TOPOLOGY_MAX_TASK_PARALLELISM**: 该配置项设置了单个组件（bolt/spout）的线程数上限。生产环境上的 topology 往往含有很高的并行度（数百个线程），导致在本地模式下测试 topology（拓扑）时会有较大的负载。这个配置项可以让你很容易地控制并行度。
2. **Config.TOPOLOGY_DEBUG**: 此配置项设置为 true 时，spout 或者 bolt 每一次发送 tuple 的时候，Storm都会打印日志。这个功能对于调试很有用。