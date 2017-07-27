---
title: 中文文档
layout: documentation
documentation: true
---


> #### NOTE（注意）

> 在最新版本中, class packages 已经从 "backtype.storm" 改变成 "org.apache.storm" 了, 所以使用旧版本编译的 topology 代码不会像在 Storm 1.0.0 上那样运行了. 通过以下配置提供向后的兼容性

> `client.jartransformer.class: "org.apache.storm.hack.StormShadeTransformer"`

> 如果要运行使用较旧版本 Storm 编译的代码, 则需要在 Storm 安装中添加上述配置. 该配置应该添加到您用于提交 topologies（拓扑）的机器中.

> 更多细节, 请参阅 [https://issues.apache.org/jira/browse/STORM-1202](https://issues.apache.org/jira/browse/STORM-1202). 


### Storm 基础

* [Javadoc](javadocs/index.html)
* [概念](Concepts.html)
* [调度器](Storm-Scheduler.html)
* [配置](Configuration.html)
* [保证消息处理](Guaranteeing-message-processing.html)
* [Daemon（守护进程）容错](Daemon-Fault-Tolerance.html)
* [命令行 client（客户端）](Command-line-client.html)
* [REST API](STORM-UI-REST-API.html)
* [理解 Storm topology 的 parallelism（并行度）](Understanding-the-parallelism-of-a-Storm-topology.html)
* [FAQ](FAQ.html)

### Layers on Top of Storm

#### Storm Trident

Trident 是 Storm 的另一个 interface（接口）.
它提供了 exactly-once（仅且一次）处理, "transactional（事务性的）" datastore persistence（数据存储持久化）, 以及一些常见的 stream analytics operations（流式分析操作）.

* [Trident 教程](Trident-tutorial.html)     -- 基础的概念和预排工作
* [Trident API 概述](Trident-API-Overview.html) -- 针对 transforming（转换）和 orchestrating 数据的操作
* [Trident State（状态）](Trident-state.html)        -- exactly-once（仅且一次）处理以及 fast（快速的）, persistent aggregation（持久化的聚合）
* [Trident spouts](Trident-spouts.html)       -- transactional（事务性的）和 non-transactional（非事务性的）数据引入
* [Trident RAS API](Trident-RAS-API.html)     -- 与 Trident 一起使用 Resource Aware Scheduler .

#### Storm SQL

该 Storm SQL 的集成可以让用户在 Storm 的 streaming data（流式数据）上来运行 SQL 查询.

NOTE（注意）: Storm SQL 是一个 `experimental（实验性的）` 功能, 所以 Storm SQL 的结构和所支持的功能在以后可能会发生变化.
但是小的变化不会影响用户体验. 在引入 UX 更改时, 我们会及时通知用户.

* [Storm SQL 概述](storm-sql.html)
* [Storm SQL 示例](storm-sql-example.html)
* [Storm SQL 文献](storm-sql-reference.html)
* [Storm SQL 结构](storm-sql-internal.html)

#### Flux

* [Flux Data Driven Topology Builder](flux.html)

### Storm 安装和部署

* [安装一个 Storm 集群](Setting-up-a-Storm-cluster.html)
* [Local mode（本地模式）](Local-mode.html)
* [问题排查](Troubleshooting.html)
* [在生产 cluster（集群）上运行 topologies（拓扑）](Running-topologies-on-a-production-cluster.html)
* [构建 Storm](Maven.html) with Maven
* [安装 Secure（安全的）Cluster（集群）](SECURITY.html)
* [CGroup 的实施](cgroups_in_storm.html)
* [Pacemaker 针对大集群减低在 zookeeper 上的负载](Pacemaker.html)
* [Resource Aware Scheduler（资源意识调度器）](Resource_Aware_Scheduler_overview.html)
* [Daemon Metrics/Monitoring（守护进程的度量/监控）](storm-metrics-profiling-internal-actions.html)
* [Windows 平台的用户指南](windows-users-guide.html)

### Storm 中级

* [Serialization（序列化）](Serialization.html)
* [Common patterns（常见模式）](Common-patterns.html)
* [Clojure DSL](Clojure-DSL.html)
* [与 Storm 一起使用非 JVM 的语言](Using-non-JVM-languages-with-Storm.html)
* [分布式的 RPC](Distributed-RPC.html)
* [Transactional topologies（事务性的拓扑）](Transactional-topologies.html)
* [Hooks（钩子）](Hooks.html)
* [Metrics（度量）](Metrics.html)
* [State Checkpointing](State-checkpointing.html)
* [Windowing（窗口操作）](Windowing.html)
* [Joining Streams](Joins.html)
* [Blobstore(Distcahce)](distcache-blobstore.html)

### Storm 调试
* [Dynamic Log Level Settings](dynamic-log-level-settings.html)
* [Searching Worker Logs](Logs.html)
* [Worker Profiling](dynamic-worker-profiling.html)
* [Event Logging](Eventlogging.html)

### Storm 与外部系统, 以及其它库的集成
* [Apache Kafka 集成](storm-kafka.html), [新的 Kafka Consumer（消费者）集成](storm-kafka-client.html)
* [Apache HBase 集成](storm-hbase.html)
* [Apache HDFS 集成](storm-hdfs.html)
* [Apache Hive 集成](storm-hive.html)
* [Apache Solr 集成](storm-solr.html)
* [Apache Cassandra 集成](storm-cassandra.html)
* [JDBC 集成](storm-jdbc.html)
* [JMS 集成](storm-jms.html)
* [Redis 集成](storm-redis.html)
* [Event Hubs 集成](storm-eventhubs.html)
* [Elasticsearch 集成](storm-elasticsearch.html)
* [MQTT 集成](storm-mqtt.html)
* [Mongodb 集成](storm-mongodb.html)
* [OpenTSDB 集成](storm-opentsdb.html)
* [Kinesis 集成](storm-kinesis.html)
* [Druid 集成](storm-druid.html)
* [Kestrel 集成](Kestrel-and-Storm.html)

#### Container, Resource Management System Integration

* [YARN 集成](https://github.com/yahoo/storm-yarn), [通过 Slider 集成 YARN](http://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.3.2/bk_yarn_resource_mgt/content/ref-7d103a48-7c2e-4b7b-aab5-62c739a32ee0.1.html)
* [Mesos 集成](https://github.com/mesos/storm)
* [Docker 集成](https://hub.docker.com/_/storm/)
* [Kubernetes 集成](https://github.com/kubernetes/kubernetes/tree/master/examples/storm)

### Storm 高级

* [为 Storm 定义非 JVM 语言的 DSL](Defining-a-non-jvm-language-dsl-for-storm.html)
* [多语言协议](Multilang-protocol.html)（如何为其它语言提供支持）
* [实现文档](Implementation-docs.html)

