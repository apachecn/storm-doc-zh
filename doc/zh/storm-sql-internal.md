---
title: Storm SQL 内部实现
layout: documentation
documentation: true
---

本页描述了 Storm SQL 的设计和实现.

## 概览

SQL是一个很好使用但又复杂的标准. 包括 Drill，Hive，Phoenix 和 Spark 在内的几个项目都在其 SQL 层面上投入了大量资金. StormSQL 的主要设计目标之一是利用这些项目的现有资源. StormSQL 利用[Apache Calcite](///calcite.apache.org) 来实现 SQL 标准. StormSQL 专注于将 SQL 语句编译成Storm / Trident 拓扑，以便它们可以在 Storm 集群中执行.

图1描述了在 StormSQL 中执行 SQL 查询的工作流程. 首先，用户提供了一系列 SQL 语句. StormSQL 解析 SQL 语句并将其转换为 Calcite 逻辑计划. 逻辑计划由一系列 SQL 逻辑运算符组成，描述如何执行查询而不考虑底层执行引擎. 逻辑运算符的一些示例包括 `TableScan`, `Filter`, `Projection` 和`GroupBy`.

<div align="center">
<img title="Workflow of StormSQL" src="images/storm-sql-internal-workflow.png" style="max-width: 80rem"/>

<p>图 1: StormSQL 工作流.</p>
</div>

The next step is to compile the logical execution plan down to a physical execution plan. A physical plan consists of physical operators that describes how to execute the SQL query in *StormSQL*. Physical operators such as `Filter`, `Projection`, and `GroupBy` are directly mapped to operations in Trident topologies. StormSQL also compiles expressions in the SQL statements into Java code blocks and plugs them into the Trident functions which will be compiled once and executed in runtime.

Finally, StormSQL submits created Trident topology with empty packaged JAR to the Storm cluster. Storm schedules and executes the Trident topology in the same way of it executes other Storm topologies.

The follow code blocks show an example query that filters and projects results from a Kafka stream.

```
CREATE EXTERNAL TABLE ORDERS (ID INT PRIMARY KEY, UNIT_PRICE INT, QUANTITY INT) LOCATION 'kafka://localhost:2181/brokers?topic=orders' ...

CREATE EXTERNAL TABLE LARGE_ORDERS (ID INT PRIMARY KEY, TOTAL INT) LOCATION 'kafka://localhost:2181/brokers?topic=large_orders' ...

INSERT INTO LARGE_ORDERS SELECT ID, UNIT_PRICE * QUANTITY AS TOTAL FROM ORDERS WHERE UNIT_PRICE * QUANTITY > 50
```

The first two SQL statements define the inputs and outputs of external data. Figure 2 describes the processes of how StormSQL takes the last `SELECT` query and compiles it down to Trident topology.

<div align="center">
<img title="Compiling the example query to Trident topology" src="images/storm-sql-internal-example.png" style="max-width: 80rem"/>

<p>Figure 2: Compiling the example query to Trident topology.</p>
</div>


## Constraints of querying streaming tables

There are several constraints when querying tables that represent a real-time data stream:

* The `ORDER BY` clause cannot be applied to a stream.
* There is at least one monotonic field in the `GROUP BY` clauses to allow StormSQL bounds the size of the buffer.

For more information please refer to http://calcite.apache.org/docs/stream.html.

## Dependency

Storm takes care about necessary dependencies of Storm SQL except the data source JAR which is used by `EXTERNAL TABLE`. 
You can use `--jars` or `--artifacts` option to `storm sql` so that data source JAR can be included to Storm SQL Runner and also Trident Topology runtime classpath.
(Use `--artifacts` if your data source JARs are available in Maven repository since it handles transitive dependencies.)

Please refer [Storm SQL integration](storm-sql.html) page to how to do it.