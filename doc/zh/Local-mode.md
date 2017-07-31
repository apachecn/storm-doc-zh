---
title: 本地模式
layout: documentation
documentation: true
---
本地模式模拟了一个单进程的Storm集群,对开发和测试topologies非常有用. 在本地模式下运行topologies与 [集群模式](Running-topologies-on-a-production-cluster.html)相似.

要创建单进程集群,只需使用'LocalCluster'类. 例如:

```java
import org.apache.storm.LocalCluster;

LocalCluster cluster = new LocalCluster();
```

然后,您可以使用`LocalCluster`对象的`submitTopology`方法提交topologies,就像 [StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html)的一样, `submitTopology` 设置名称,topology的参数和topology对象. 你可以使用 `killTopology` 方法杀掉 topology,参数就是topology的名称
通过简单的调用下面方法来关闭本地集群:

```java
cluster.shutdown();
```

### 本地模式的通用配置
您可以看到完整的配置列表 [config列表](javadocs/org/apache/storm/Config.html).

1. **Config.TOPOLOGY_MAX_TASK_PARALLELISM**: 此项配置为单个组件创建的线程数上限. 通常,生产环境下的topology具有大量的并行性（数百个线程）.而当在本地模式下测试topology时,这显然是不合理的. 该配置允许您轻松控制并行性.
2. **Config.TOPOLOGY_DEBUG**: 当此项设置为true时,Storm将在何spout或bolt每次发出一个tuple时记录一条消息. 这对调试非常有用.