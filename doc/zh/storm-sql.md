---
title: Storm SQL integration
layout: documentation
documentation: true
---

Storm SQL 使用户在 Storm 中的流数据上运行 SQL 查询. SQL 接口不仅可以加快流数据分析的开发周期, 同时还创造了一个机遇, 统一如 [Apache Hive](///hive.apache.org) 和实时流数据分析之类的批量数据处理.

在很高的级别, StromSQL 把 SQL 编译为 [Trident](Trident-API-Overview.html) 拓扑并在 Strom 集群中执行. 本文档提供了作为一个末端用户如何使用 StormSQL 的相关信息. 对于想更深入了解 StormSQL 的设计和实现的朋友请参考[这个](storm-sql-internal.html) 页面.

Storm SQL 是一个 `试验性` 的功能, 因此其内部逻辑和支持的特性可能在将来会有变化. 但是小的改动不会影响用户体验. 在引入 UX 更改时，我们会提醒和通知用户.

## 使用

运行 `storm sql` 命令把 SQL 语句编译为 Trident topology, 并且提交到 Storm 集群. 

```
$ bin/storm sql <sql-file> <topo-name>
```

`sql-file` 文件中包含需要被执行的 SQL 语句的列表, `topo-name` 是 topology 的名称.

当用户把 `topo-name` 设置为 `--explain` 的时候, StormSQL 激活 `explain mode` 以显示查询计划而不是提交拓扑. 详细的解释请参见 `显示查询计划(explain mode)` 一节.

## 支持的特性

当前版本支持以下特性:

* 读出和流入外部数据源
* 过滤 tuples
* 投影
* 用户自定义函数 (标量)

特意不支持聚合和连接. 当 Storm SQL 要支持本地 `Streaming SQL` 时, 将会介绍这些特性.

## 指定外部数据源

在 StormSQL中, 数据表现为外部表. 用户可以使用语句 `CREATE EXTERNAL TABLE` 指定数据源. `CREATE EXTERNAL TABLE` 语法与 [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)中的非常接近.

```
CREATE EXTERNAL TABLE table_name field_list
    [ STORED AS
      INPUTFORMAT input_format_classname
      OUTPUTFORMAT output_format_classname
    ]
    LOCATION location
    [ PARALLELISM parallelism ]
    [ TBLPROPERTIES tbl_properties ]
    [ AS select_stmt ]
```

各种属性的详细解释参考 [Hive Data Definition Language](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL).

`PARALLELISM` 是 StormSQL 特有的关键词, 用于描述输入数据源的并行度. 等同于为 Trident Spout 设置并行度.

默认值是 1, 这个选项对于输出数据源没有任何影响. (如果需要的话, 以后可能会改变. 正常情况下应当避免重新分区).

例如, 下面的语句指定了一个 Kafka Spout 和 sink: 

```
CREATE EXTERNAL TABLE FOO (ID INT PRIMARY KEY) LOCATION 'kafka://localhost:2181/brokers?topic=test' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
```

## 植入外部数据源

用户通过实现 `ISqlTridentDataSource` 接口并且利用 Java 的 service loader 机制注册他们, 以植入外部数据源. 外部数据源将根据表的 URI模式 进行选择. 更多细节请参考 `storm-sql-kafka`.

## 指定 User Defined Function (UDF)

用户可以使用 `CREATE FUNCTION` 语句来定义 user defined function (标量 或者 聚合). 
例如, 下面的语句使用`org.apache.storm.sql.TestUtils$MyPlus` 类定义了一个名为 `MYPLUS` 的函数.

```
CREATE FUNCTION MYPLUS AS 'org.apache.storm.sql.TestUtils$MyPlus'
```

Storm SQL 通过检查用了什么方法来决定这个函数作为一个 标量 还是 聚合.
如果类中定义了 `evaluate` 方法, Storm SQL 将这个函数作为 `scalar`.

标量函数类的示例:

```
  public class MyPlus {
    public static Integer evaluate(Integer x, Integer y) {
      return x + y;
    }
  }

```

## 例子: 过滤 Kafka 流

假设有一个 Kafka stream 代表订单交易. 每个 stream 中的消息包含订单的 id, 产品的单价, 产品数量. 目标是过滤重要交易的订单(译注:总价格大于50的订单)，并将这些订单插入到另一个 Kafka stream 用于进一步分析.

用户可以在 SQL 文件中指定下列 SQL 语句:

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders'
CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.org.apache.storm.kafka.ByteBufferSerializer"}}'
INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

第一个语句定义一个表 `ORDER` 代表输入流. `LOCATION` 从句指定 ZkHost (`localhost:2181`), broker的路径(`/brokers`), 和 topic名称(`orders`).

类似的, 第二个语句指定了表 `LARGE_ORDERS` 代表一个输出流. `TBLPROPERTIES` 从句指定了一个 [KafkaProducer](http://kafka.apache.org/documentation.html#producerconfigs) 的配置, 这个从句是 Kafka sink 表必须的.

第三个语句是一个定义拓扑的 `SELECT` 语句: 它指示 StormSQL 过滤 `ORDERS` 表中的所有订单, 计算各订单总价并将匹配的记录插入 `LARGE_ORDER` 指定的 Kafka流 中.

要想运行这个例子, 用户需要在 classpath 中包含数据源 (本例中 `storm-sql-kafka`)和它的所有依赖. 当运行 `storm sql` 的时候 Storm SQL 的依赖会自动处理. 用户可以在提交的步骤中包含数据源依赖, 如下所示:

```
$ bin/storm sql order_filtering.sql order_filtering --artifacts "org.apache.storm:storm-sql-kafka:2.0.0-SNAPSHOT,org.apache.storm:storm-kafka:2.0.0-SNAPSHOT,org.apache.kafka:kafka_2.10:0.8.2.2^org.slf4j:slf4j-log4j12,org.apache.kafka:kafka-clients:0.8.2.2"
```

上面的命令提交 SQL 语句到 StormSQL. 如果用户使用了不同版本的 Storm 或者 Kafka, 需要替换每个 artifacts 的版本.

现在, 应该能在 Storm UI 中看到 `order_filtering` 拓扑.

## 显示查询计划(explain mode)

就像 SQL 语句上的 `explain`, StormSQL 在运行 Storm SQL 执行器时提供 `explain mode`. 在分析模式下, StormSQL 分析每一个查询语句(仅DML)并显示执行计划而不是提交拓扑.

为了运行 `explain mode`, 需要设置拓扑名称为 `--explain` 并像用和提交相同的方式执行 `storm sql` 命令.

例如, 当以分析模式运行上面的例子的时:
 
```
$ bin/storm sql order_filtering.sql --explain --artifacts "org.apache.storm:storm-sql-kafka:2.0.0-SNAPSHOT,org.apache.storm:storm-kafka:2.0.0-SNAPSHOT,org.apache.kafka:kafka_2.10:0.8.2.2\!org.slf4j:slf4j-log4j12,org.apache.kafka:kafka-clients:0.8.2.2"
```

StormSQL 输出打印如下:

```
===========================================================
query>
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.storm.kafka.ByteBufferSerializer"}}'
-----------------------------------------------------------
16:53:43.951 [main] INFO  o.a.s.s.r.DataSourcesRegistry - Registering scheme kafka with org.apache.storm.sql.kafka.KafkaDataSourcesProvider@4d1bf319
No plan presented on DDL
===========================================================
===========================================================
query>
CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.storm.kafka.ByteBufferSerializer"}}'
-----------------------------------------------------------
No plan presented on DDL
===========================================================
===========================================================
query>
INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
-----------------------------------------------------------
plan>
LogicalTableModify(table=[[LARGE_ORDERS]], operation=[INSERT], updateColumnList=[[]], flattened=[true]), id = 8
  LogicalProject(ID=[$0], TOTAL=[*($1, $2)]), id = 7
    LogicalFilter(condition=[>(*($1, $2), 50)]), id = 6
      EnumerableTableScan(table=[[ORDERS]]), id = 5

===========================================================

```

## 局限

- Windowing 尚未实现.
- 不支持聚合和连接（待到 `流SQL` 成熟）
