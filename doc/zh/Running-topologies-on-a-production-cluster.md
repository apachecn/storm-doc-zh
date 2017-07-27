---
title: 在生产集群上运行 Topology
layout: documentation
documentation: true
---
在生产集群上运行 Topology 类似于在 [本地模式](Local-mode.html) 下运行。以下是步骤：

1）定义 Topology （如果使用Java 定义，则使用 [TopologyBuilder](javadocs/org/apache/storm/topology/TopologyBuilder.html) ）

2）使用 [StormSubmitter](javadocs/org/apache/storm/StormSubmitter.html) 将 topology 提交到集群。 `StormSubmitter` 以 topology 的名称，topology 的配置和 topology 本身作为输入。例如：

```java
Config conf = new Config();
conf.setNumWorkers(20);
conf.setMaxSpoutPending(5000);
StormSubmitter.submitTopology("mytopology", conf, topology);
```

3）创建一个包含你的代码和代码的所有依赖项的 jar （除了 Storm - Storm jar 将被添加到 worker 节点上的 classpath 中）。

如果您使用 Maven， [Maven Assembly Plugin](http://maven.apache.org/plugins/maven-assembly-plugin/) 插件可以为您做包装。只需将其添加到您的 pom.xml 中即可：

```xml
  <plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
      <descriptorRefs>  
        <descriptorRef>jar-with-dependencies</descriptorRef>
      </descriptorRefs>
      <archive>
        <manifest>
          <mainClass>com.path.to.main.Class</mainClass>
        </manifest>
      </archive>
    </configuration>
  </plugin>
```

然后运行 mvn assembly:assembly 来获取适当打包的 jar 。确保您 [排除了](http://maven.apache.org/plugins/maven-assembly-plugin/examples/single/including-and-excluding-artifacts.html) Storm jar，因为群集已经在类路径上有 Storm。

4）使用 `storm` 客户端将 topology 提交到集群，指定您的 jar 的路径，要运行的类名以及将使用的任何参数：

`storm jar path/to/allmycode.jar org.me.MyTopology arg1 arg2 arg3`

`storm jar` 将 jar 提交到集群并配置 `StormSubmitter` 该类与正确的集群进行通信。在这个例子中，上传 jar 后 `storm jar` ， `org.me.MyTopology` 使用参数 "arg1", "arg2", and "arg3" 调用 main 函数。

您可以找到如何配置 `storm` 客户端与 Storm 集群进行交流，以 [设置开发环境](Setting-up-development-environment.html)。

### 常用配置

您可以根据 topology 设置各种配置。您可以在 [这里](javadocs/org/apache/storm/Config.html) 找到您可以设置的所有配置的列表。以 "TOPOLOGY" 为前缀的可以在 topology 特定的基础上被覆盖（其他的是集群配置，不能被覆盖）。以下是为 topology 设置的一些常见的：

1. **Config.TOPOLOGY_WORKERS**: 设置用于执行 topology 的工作进程数。例如，如果将其设置为25，则集群中将有25个 Java 进程执行所有任务。如果 topology 中的所有组件都具有150个并行度，则每个工作进程将在其中运行6个任务作为线程。
2. **Config.TOPOLOGY_ACKER_EXECUTORS**: 这将设置跟踪元组树的执行程序的数量，并检测出 spout 元组何时完全处理。 Ackers 是 Storm 可靠性模型的组成部分，您可以在 [保证消息处理](Guaranteeing-message-processing.html) 中阅读更多信息。通过不设置此变量或将其设置为 null ， Storm 将将执行器的数量设置为等于为此 topology 配置的工作人员数。如果这个变量设置为0，那么 Storm 会立即从元器件脱落出来，使其可靠性降低。
3. **Config.TOPOLOGY_MAX_SPOUT_PENDING**: 这将一次设置单个 spout 任务中可以挂起的 spout 元组的最大数量（挂起意味着元组尚未被确认或失败）。强烈建议您设置此配置以防止队列爆炸。
4. **Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS**: 这是一个 spout 元组在被认为失败之前必须完全完成的最长时间。此值默认为30秒，这对大多数拓扑结构都是足够的。有关Storm的可靠性模型如何工作的更多信息，请参阅 [保证消息处理](Guaranteeing-message-processing.html)。
5. **Config.TOPOLOGY_SERIALIZATIONS**: 您可以使用此配置向 Storm 注册更多序列化程序，以便您可以在元组内使用自定义类型。

### Killing 一个 topology

要 kill 一个 topology, 只需运行:

`storm kill {stormname}`

提供与 `storm kill` 提交 topology 时使用的名称相同的名称。

Storm不会立即杀死 topology。相反，它会停用所有的端口，以使它们不再发出任何元组，然后 Storm 会在销毁 所有workers 之前等待 Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS 秒。这给了 topology 足够的时间来完成它被杀死时处理的任何元组。


Storm won't kill the topology immediately. Instead, it deactivates all the spouts so that they don't emit any more tuples, and then Storm waits Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS seconds before destroying all the workers. This gives the topology enough time to complete any tuples it was processing when it got killed.

### Updating 一个正在运行的 topology

要更新正在运行的 topology ，目前唯一的选项是终止当前 topology 并重新提交新的 topology 。一个计划的功能是实现一个 `storm swap` 交换正在运行的 topology 结构的命令，以确保最短的停机时间，并且两个 topology 不会同时处理元组。

### Monitoring topologies

监控 topology 的最佳位置是使用 Storm UI。 Storm UI 提供有关每个运行 topology 的每个组件的吞吐量和延迟性能的任务和精细统计信息中发生的错误的信息。

您还可以查看群集机器上的工作日志。
