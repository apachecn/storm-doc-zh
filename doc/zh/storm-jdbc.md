---
title: Storm JDBC Integration
layout: documentation
documentation: true
---

Storm/Trident集成JDBC.该包中包含的核心bolts 和 trident states ，允许storm topology把storm tuples插入数据库表中或者执行数据库查询，并且丰富了storm topology 中的tuples.

**注**：在下面的示例中，我们使用com.google.common.collect.Lists和com.google.common.collect.Maps。

## Inserting into a database. 插入数据库.
该包的中bolt 和 trident state可以将数据插入到数据库表中绑定到单个表。

### ConnectionProvider
由不同的连接池机制实现的接口 `org.apache.storm.jdbc.common.ConnectionProvider`

java
public interface ConnectionProvider extends Serializable {
    /**
     * method must be idempotent.
     */
    void prepare();

    /**
     *
     * @return a DB connection over which the queries can be executed.
     */
    Connection getConnection();

    /**
     * called once when the system is shutting down, should be idempotent.
     */
    void cleanup();
}
```

即插即用，我们支持'org.apache.storm.jdbc.common.HikariCPConnectionProvider'，这是一个使用HikariCP的实现。

###JdbcMapper
使用JDBC在表中插入数据的主要API是org.apache.storm.jdbc.mapper.JdbcMapper 接口：

```java
public interface JdbcMapper  extends Serializable {
    List<Column> getColumns(ITuple tuple);
}
```

`getColumns()` 方法定义了storm tuple如何映射到数据库中表示一行的列的列表。

**返回的列表的顺序很重要。 查询中的占位符以与返回列表相同的顺序进行解析。**

例如，如果用户的插入查询是 `insert into user(user_id, user_name, create_date) values (?,?, now())` , `getColumns` 方法返回列表中的第一项将映射到第一位，第二位到第二位，依此类推。我们不会解析提供的查询，以列名称来尝试和解析占位符。没有对查询语法进行任何假设，允许这个连接器被一些非标准的sql框架（如仅支持upsert的Pheonix）使用。


### JdbcInsertBolt
使用 `JdbcInsertBolt`，你可以通过指定一个 `ConnectionProvider` 实例和将storm tuple转换为DB行的 `JdbcMapper` 实例来构造一个 `JdbcInsertBolt`实例。另外，您必须使用 `withTableName` 方法提供表名或使用 `withInsertQuery`插入查询。如果您指定了一个插入查询，那么您应该确保您的 `JdbcMapper`实例将按照插入查询中的顺序返回一列列表。您可以选择指定一个查询超时秒参数，指定插入查询可以执行的最大秒数。默认设置为topology.message.timeout.secs的值为-1将表示不设置任何查询超时。
您应该将查询超时值设置为<= topology.message.timeout.secs。


 ```java
Map hikariConfigMap = Maps.newHashMap();
hikariConfigMap.put("dataSourceClassName","com.mysql.jdbc.jdbc2.optional.MysqlDataSource");
hikariConfigMap.put("dataSource.url", "jdbc:mysql://localhost/test");
hikariConfigMap.put("dataSource.user","root");
hikariConfigMap.put("dataSource.password","password");
ConnectionProvider connectionProvider = new HikariCPConnectionProvider(hikariConfigMap);

String tableName = "user_details";
JdbcMapper simpleJdbcMapper = new SimpleJdbcMapper(tableName, connectionProvider);

JdbcInsertBolt userPersistanceBolt = new JdbcInsertBolt(connectionProvider, simpleJdbcMapper)
                                    .withTableName("user")
                                    .withQueryTimeoutSecs(30);
                                    Or
JdbcInsertBolt userPersistanceBolt = new JdbcInsertBolt(connectionProvider, simpleJdbcMapper)
                                    .withInsertQuery("insert into user values (?,?)")
                                    .withQueryTimeoutSecs(30);                                    
 ```

### SimpleJdbcMapper
`storm-jdbc` 包括一个通用的 `JdbcMapper` 实现，称为 `SimpleJdbcMapper` ，可以映射Storm元组到数据库行。 `SimpleJdbcMapper`假定storm tuple中有与列名相同名称的字段您要写入的数据库表。


要使用 `SimpleJdbcMapper`，你只需要告诉你要写入的tableName并提供一个connectionProvider实例。


以下代码创建一个 `SimpleJdbcMapper` 实例：

1.允许映射器将 storm tuple转换为映射到表test.user_details中的行的列的列表。
2.将使用提供的HikariCP配置来建立具有指定数据库配置的连接池自动找出您要写入的表的列名称和相应的数据类型。
请参阅https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby了解有关hikari配置属性的更多信息。


```java
Map hikariConfigMap = Maps.newHashMap();
hikariConfigMap.put("dataSourceClassName","com.mysql.jdbc.jdbc2.optional.MysqlDataSource");
hikariConfigMap.put("dataSource.url", "jdbc:mysql://localhost/test");
hikariConfigMap.put("dataSource.user","root");
hikariConfigMap.put("dataSource.password","password");
ConnectionProvider connectionProvider = new HikariCPConnectionProvider(hikariConfigMap);
String tableName = "user_details";
JdbcMapper simpleJdbcMapper = new SimpleJdbcMapper(tableName, connectionProvider);
```
在上面的示例中初始化的映射器假定storm tuple具有要插入数据的表的所有列的值，其 `getColumn`方法将按照Jdbc连接实例的 `connection.getMetaData().getColumns();`的顺序返回列。

**如果您为 `JdbcInsertBolt` 指定了自己的插入查询，则必须使用显式列显示方式初始化 `SimpleJdbcMapper` ，以使模式具有与插入查询相同顺序的列。**

例如，如果您的插入查询是 `Insert into user (user_id, user_name) values (?,?)` ，那么您的 `SimpleJdbcMapper` 应该使用以下语句进行初始化：
```java
List<Column> columnSchema = Lists.newArrayList(
    new Column("user_id", java.sql.Types.INTEGER),
    new Column("user_name", java.sql.Types.VARCHAR));
JdbcMapper simpleJdbcMapper = new SimpleJdbcMapper(columnSchema);
```

如果您的 storm tuple仅具有子集列的字段i.e。如果表中的某些列具有默认值，并且您只想为没有默认值的列插入值，则可以通过显示的指定columnschema初始化`SimpleJdbcMapper`。例如，如果你有一个user_details表`create table if not exists user_details (user_id integer, user_name varchar(100), dept_name varchar(100), create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP);`在此表中，create_time列具有默认值。 要确保只插入没有默认值的列你可以初始化`jdbcMapper` 如下：

```java
List<Column> columnSchema = Lists.newArrayList(
    new Column("user_id", java.sql.Types.INTEGER),
    new Column("user_name", java.sql.Types.VARCHAR),
    new Column("dept_name", java.sql.Types.VARCHAR));
JdbcMapper simpleJdbcMapper = new SimpleJdbcMapper(columnSchema);
```
### JdbcTridentState
我们还支持持久化trident state 通过使用trident topologies。要创建一个jdbc 持久化的tridentstate，您需要使用表名或插入查询、JdbcMapper实例和连接提供程序实例进行初始化。见下面的例子：

```java
JdbcState.Options options = new JdbcState.Options()
        .withConnectionProvider(connectionProvider)
        .withMapper(jdbcMapper)
        .withTableName("user_details")
        .withQueryTimeoutSecs(30);
JdbcStateFactory jdbcStateFactory = new JdbcStateFactory(options);
```
类似于 `JdbcInsertBolt`，你可以使用 `withInsertQuery`来指定一个自定义的插入查询，而不是指定一个表名。

## Lookup from Database 查询数据库
我们支持数据库的 `select` 查询以丰富拓扑中的storm tuples。使用JDBC执行数据库查询的主要API是 `org.apache.storm.jdbc.mapper.JdbcLookupMapper` 接口：

```java
    void declareOutputFields(OutputFieldsDeclarer declarer);
    List<Column> getColumns(ITuple tuple);
    List<Values> toTuple(ITuple input, List<Column> columns);
```

`declareOutputFields` 方法用于指明将作为处理storm tuple的输出tuple的一部分发出哪些字段。

`getColumns` 方法指定选择查询中的占位符列及其SQL类型和要使用的值。例如在上面提到的user_details表中，如果你正在执行一个查询'select user_name from user_details where user_id =？and create_time>？`getColumns`方法将需要一个storm输入tuple，并返回一个包含两个项目的列表。 `Column`类型的 `getValue()`方法的第一个实例将被用作 `user_id`的值进行查找， `Column`类型的 `getValue()`方法的第二个实例将被用作 `create_time`的值。
**注意：返回的列表中的顺序决定了占位符的价值。 换句话说，列表中的第一个项目映射在select查询中第一个'？'，第二个项目是第二个'?',依次类推。**

`toTuple` 方法将select查询的结果接收输入tuple和表示DB行的列的列表，并返回要发射的值的列表。

**请注意，它返回一个 `Values` 列表，而不仅仅是一个 `Values` 的实例。**
这允许将单个DB行映射到多个输出storm tuples。

###SimpleJdbcLookupMapper
`storm-jdbc`包括一个通用的`JdbcLookupMapper`实现，叫做 `SimpleJdbcLookupMapper`。

要使用 `SimpleJdbcMapper`，必须使用您的bolt输出的字段和查询语句中占位符的列的列表来初始化它。以下示例展示如何初始化一个 `SimpleJdbcLookupMapper`，它将' `user_id,user_name,create_date` 声明为输出字段， `user_id`作为查询语句中的占位符列。`SimpleJdbcMapper` 假定您的tuple中的字段名称等于占位符列名称，即在我们的示例中，`SimpleJdbcMapper` 将在输入tuple中查找一个字段 `use_id` ，并将其值用作查询语句中占位符的值。对于构造输出tuple，它首先在输入元组中查找`outputFields`中指定的字段，如果在输入元组中找不到，那么它会查看select query的输出行中与列名称相同的列。所以在下面的例子中，如果输入tuple有字段 `user_id, create_date` ，查询语句是`select user_name from user_details where user_id = ?`，对于每个输入tuple `SimpleJdbcLookupMapper.getColumns(tuple)`将返回`tuple.getValueByField("user_id")` 用作select查询的`?` 中的值。对于DB的每个输出行，`SimpleJdbcLookupMapper.toTuple()`将使用输入元组中的 `user_id, create_date`，因为从结果行只添加`user_name`，并将这3个字段作为单个输出元组返回。

```java
Fields outputFields = new Fields("user_id", "user_name", "create_date");
List<Column> queryParamColumns = Lists.newArrayList(new Column("user_id", Types.INTEGER));
this.jdbcLookupMapper = new SimpleJdbcLookupMapper(outputFields, queryParamColumns);
```

### JdbcLookupBolt
要使用 `JdbcLookupBolt`，使用 `ConnectionProvider`实例， `JdbcLookupMapper`实例和select查询来构造一个它的实例。你可以选择指定一个查询超时秒参数，指定select查询可以采用的最大秒数。默认值为topology.message.timeout.secs的值。 您应该将此值设置为<= topology.message.timeout.secs。

```java
String selectSql = "select user_name from user_details where user_id = ?";
SimpleJdbcLookupMapper lookupMapper = new SimpleJdbcLookupMapper(outputFields, queryParamColumns)
JdbcLookupBolt userNameLookupBolt = new JdbcLookupBolt(connectionProvider, selectSql, lookupMapper)
        .withQueryTimeoutSecs(30);
```

### JdbcTridentState for lookup
我们还可以在trident topologies 中查询trident  state。

```java
JdbcState.Options options = new JdbcState.Options()
        .withConnectionProvider(connectionProvider)
        .withJdbcLookupMapper(new SimpleJdbcLookupMapper(new Fields("user_name"), Lists.newArrayList(new Column("user_id", Types.INTEGER))))
        .withSelectQuery("select user_name from user_details where user_id = ?");
        .withQueryTimeoutSecs(30);
```

## Example:
可以在 `src/test/java/topology` 目录中找到一个可运行的例子。

### Setup
* 确保您为您选择的数据库包含JDBC实现依赖关系，作为构建配置的一部分。

* 测试拓扑执行以下查询，因此您的预期数据库必须支持这些查询才能使测试拓扑工作。

```SQL
create table if not exists user (user_id integer, user_name varchar(100), dept_name varchar(100), create_date date);
create table if not exists department (dept_id integer, dept_name varchar(100));
create table if not exists user_department (user_id integer, dept_id integer);
insert into department values (1, 'R&D');
insert into department values (2, 'Finance');
insert into department values (3, 'HR');
insert into department values (4, 'Sales');
insert into user_department values (1, 1);
insert into user_department values (2, 2);
insert into user_department values (3, 3);
insert into user_department values (4, 4);
select dept_name from department, user_department where department.dept_id = user_department.dept_id and user_department.user_id = ?;
```
### Execution
使用storm jar命令运行`org.apache.storm.jdbc.topology.UserPersistanceTopology`类。 参考该课程第5小节storm jar org.apache.storm.jdbc.topology.UserPersistanceTopology <dataSourceClassName> <dataSource.url> <user> <password> [拓扑名称]

要使其与Mysql一起工作，您可以将以下内容添加到pom.xml中

```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.31</version>
</dependency>
```

您可以使用maven程序集插件生成具有依赖关系的单个jar。 要使用插件，将以下内容添加到您的pom.xml并执行
`mvn clean compile assembly:single`

```
<plugin>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <archive>
            <manifest>
                <mainClass>fully.qualified.MainClass</mainClass>
            </manifest>
        </archive>
        <descriptorRefs>
            <descriptorRef>jar-with-dependencies</descriptorRef>
        </descriptorRefs>
    </configuration>
</plugin>
```

Mysql Example:
```
storm jar ~/repo/incubator-storm/external/storm-jdbc/target/storm-jdbc-0.10.0-SNAPSHOT-jar-with-dependencies.jar org.apache.storm.jdbc.topology.UserPersistanceTopology  com.mysql.jdbc.jdbc2.optional.MysqlDataSource jdbc:mysql://localhost/test root password UserPersistenceTopology
```

您可以针对应显示新插入的行的用户表执行select查询：

```
select * from user;
```

For trident you can view `org.apache.storm.jdbc.topology.UserPersistanceTridentTopology`.

对于trident，您可以查看`org.apache.storm.jdbc.topology.UserPersistanceTridentTopology`。


