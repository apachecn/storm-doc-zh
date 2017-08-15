---
title: Storm Solr 集成
layout: documentation
documentation: true
---

针对 Apache Solr 的 Storm 和 Trident 集成.
该软件包包括一个 bolt和 trident state，它们可以使 Storm topology 将 storm  tuples 的内容索引到 Solr collections.
 
# Index Storm tuples 到 Solr collection 中
bolt 和 trident state 使用一个提供的 mappers（映射器）来构建一个 `SolrRequest` 对象, 负责对 Solr 进行更新调用，从而更新指定的 collection 的 index.
 
# 使用示例
在本节中，我们将提供一些关于如何构建 Storm 和 Trident topologies 来索引 Solr 的简单代码片段.
在随后的章节中，我们详细介绍了 Storm Solr 集成的两个关键组件，`SolrUpdateBolt` 和 `Mappers`，`SolrFieldsMapper` 和`SolrJsonMapper`。

## Storm Bolt 与 JSON Mapper 和 Count Based Commit Strategy

```java
    new SolrUpdateBolt(solrConfig, solrMapper, solrCommitStgy)
    
    // zkHostString for Solr 'gettingstarted' example
    SolrConfig solrConfig = new SolrConfig("127.0.0.1:9983");
    
    // JSON Mapper used to generate 'SolrRequest' requests to update the "gettingstarted" Solr collection with JSON content declared the tuple field with name "JSON"
    SolrMapper solrMapper = new SolrJsonMapper.Builder("gettingstarted", "JSON").build(); 
     
    // Acks every other five tuples. Setting to null acks every tuple
    SolrCommitStrategy solrCommitStgy = new CountBasedCommit(5);          
```

## Trident Topology 与 Fields Mapper
```java
    new SolrStateFactory(solrConfig, solrMapper);
    
    // zkHostString for Solr 'gettingstarted' example
    SolrConfig solrConfig = new SolrConfig("127.0.0.1:9983");
    
    /* Solr Fields Mapper used to generate 'SolrRequest' requests to update the "gettingstarted" Solr collection. The Solr index is updated using the field values of the tuple fields that match static or dynamic fields declared in the schema object build using schemaBuilder */ 
    SolrMapper solrMapper = new SolrFieldsMapper.Builder(schemaBuilder, "gettingstarted").build();

    // builds the Schema object from the JSON representation of the schema as returned by the URL http://localhost:8983/solr/gettingstarted/schema/ 
    SchemaBuilder schemaBuilder = new RestJsonSchemaBuilder("localhost", "8983", "gettingstarted")
```

## SolrUpdateBolt
`SolrUpdateBolt` 让 tuples 直接流入到 Apache Solr 中.
该 Solr index 使用 `SolrRequest` 请求来更新.
该 `SolrUpdateBolt` 可以通过实现 `SolrConfig`, `SolrMapper` 和可选的 `SolrCommitStrategy` 方法来配置它.

使用 `SolrMapper` 实现中定义的策略从 tuples 中提取 stream 到 Solr 的数据.

`SolrRquest` 可以用来发送每个 tuple，也可以根据 `SolrCommitStrategy` 实现中定义的策略进行发送.
如果 `SolrCommitStrategy` 已经到位并且批处理中的一个元组失败，则该 batch 不会被提交，并且该 bathc 中的所有 tuple 都标记为`Fail`，并重试.
另一方面，如果所有的 tuple 都成功，则 `SolrRequest` 被提交，所有 tuple 都被成功地 acked（确认）.

`SolrConfig` 是包含 Solr 配置的 class，可用于 Storm Solr bolt.
bolt 中需要的任何配置都应该放在这个 class 中.

## SolrMapper
`SorlMapper` 实现定义了从 tuple 中提取信息的策略.
public method `toSolrRequest` 接收一个 tuple 或一组 tuple，并返回一个用于更新 Solr 索引的 `SolrRequest` 对象.

### SolrJsonMapper
`SolrJsonMapper` `创建一个 Solr 更新请求，该请求被发送到由 Solr 定义的作为 JSON 格式请求的 URL endpoint.

要创建一个 `SolrJsonMapper`，客户端必须指定要更新的 collection 的名称以及包含用于更新 Solr 索引的 JSON 对象的 tuple field（元组字段）.
如果 tuple 不包含指定的字段，则在调用方法 `toSolrRequest` 时抛出 `SolrMapperException`.
如果该字段存在，则其值可以是 JSON 格式的内容的 String，或将被序列化为 JSON 的 Java 对象.
 
下面的代码片段，说明如何创建一个 `SolrJsonMapper` `对象，以更新 `getsstarted` Solr collection，并使用名称为 `JSON` 的 tuple 字段中声明的 JSON 内容

``` java
    SolrMapper solrMapper = new SolrJsonMapper.Builder("gettingstarted", "JSON").build();
```

### SolrFieldsMapper
`SolrFieldsMapper` 创建一个 Solr 更新请求，该请求被发送到处理 `SolrInputDocument` 对象的更新的 Solr URL endpoint.

要创建一个 `SolrFieldsMapper`，客户端必须指定要更新的集合的名称以及 `SolrSchemaBuilder`。
Solr 的 `Schema` 用于提取有关 Solr schema 字段和相应类型的信息.
该 metadata 用于从 tuples 中获取信息.
只有匹配静态或动态 Solr 字段的 tuple 字段才会添加到文档中
与 schema 不匹配的 tuple 字段不添加到准备进行索引的 `SolrInputDocument` 中.
针对不匹配 schema 的 tuple 字段打印出 debug log message，因此不进行索引.

`SolrFieldsMapper` 支持多 value 的字段.
多 value 的  tuple 字段必须被 tokenized（分词）.
默认的 token（词元）是 `|`.
可以通过调用作为 `SolrFieldsMapper.Builder` builder class 中的方法`org.apache.storm.solr.mapper.SolrFieldsMapper.Builder.setMultiValueFieldToken` 来指定任意的 token（词元）.

下面的代码片段演示了如何去创建一个 `SolrFieldsMapper` 对象以更新 `gettingstarted` 的 Solr collection.
该多个 value 的字段使用 `%` 而不是默认的 `|` 来拆分每个 value.
要使用默认的 token 您可以省略对 `setMultiValueFieldToken` 方法的调用.

``` java
    new SolrFieldsMapper.Builder(
            new RestJsonSchemaBuilder("localhost", "8983", "gettingstarted"), "gettingstarted")
                .setMultiValueFieldToken("%").build();
```

# 构建并且运行附带的示例
为了能够运行这些示例，您必须首先在 `storm-solr` 包中构建 `java ` 代码，然后生成具有所有依赖项的 `uber jar`.

## 构建 Storm Apache Solr 集成的代码

`mvn clean install -f REPO_HOME/storm/external/storm-solr/pom.xml`
 
## 使用 Maven Shade 插件来构建 Uber Jar

 添加以下配置到 `REPO_HOME/storm/external/storm-solr/pom.xml` 中:
 
 ```
 <plugin>
     <groupId>org.apache.maven.plugins</groupId>
     <artifactId>maven-shade-plugin</artifactId>
     <version>2.4.1</version>
     <executions>
         <execution>
             <phase>package</phase>
             <goals>
                 <goal>shade</goal>
             </goals>
             <configuration>
                 <transformers>
                     <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                         <mainClass>org.apache.storm.solr.topology.SolrJsonTopology</mainClass>
                     </transformer>
                 </transformers>
             </configuration>
         </execution>
     </executions>
</plugin>
 ```

通过运行命令以创建 uber jar:

`mvn package -f REPO_HOME/storm/external/storm-solr/pom.xml`

这将创建一个与以下模式匹配的名称和位置的 uber jar 文件:
 
`REPO_HOME/storm/external/storm/target/storm-solr-0.11.0-SNAPSHOT.jar`

## 运行示例
复制文件 `REPO_HOME/storm/external/storm-solr/target/storm-solr-0.11.0-SNAPSHOT.jar` 到 `STORM_HOME/extlib`

**The code examples provided require that you first run the [Solr gettingstarted](http://lucene.apache.org/solr/quickstart.html) example** 

### 运行 Storm Topology

    STORM_HOME/bin/storm jar REPO_HOME/storm/external/storm-solr/target/storm-solr-0.11.0-SNAPSHOT-tests.jar org.apache.storm.solr.topology.SolrFieldsTopology
 
    STORM_HOME/bin/storm jar REPO_HOME/storm/external/storm-solr/target/storm-solr-0.11.0-SNAPSHOT-tests.jar org.apache.storm.solr.topology.SolrJsonTopology

### 运行 Trident Topology

    STORM_HOME/bin/storm jar REPO_HOME/storm/external/storm-solr/target/storm-solr-0.11.0-SNAPSHOT-tests.jar org.apache.storm.solr.trident.SolrFieldsTridentTopology

    STORM_HOME/bin/storm jar REPO_HOME/storm/external/storm-solr/target/storm-solr-0.11.0-SNAPSHOT-tests.jar org.apache.storm.solr.trident.SolrJsonTridentTopology

### 验证结果

上述的 Storm 和 Trident topologies 索引了 Solr `getsstarted` collect，其对象具有以下的 'id` 模式:

\*id_fields_test_val\* 用于 `SolrFieldsTopology` 和  `SolrFieldsTridentTopology`

\*json_test_val\* 用于 `SolrJsonTopology` 和 `SolrJsonTridentTopology`

查询这些模式的 Solr，您将看到由 Storm Apache Solr 集成索引的值:

    curl -X GET -H "Content-type:application/json" -H "Accept:application/json" http://localhost:8983/solr/gettingstarted_shard1_replica2/select?q=*id_fields_test_val*&wt=json&indent=true

    curl -X GET -H "Content-type: application/json" -H "Accept: application/json" http://localhost:8983/solr/gettingstarted_shard1_replica2/select?q=*id_fields_test_val*&wt=json&indent=true

您还可以通过打开 Apache Solr UI 并在查询页面中的 `q` 文本框中粘贴 `id` 模式来查看结果

    http://localhost:8983/solr/#/gettingstarted_shard1_replica2/query
