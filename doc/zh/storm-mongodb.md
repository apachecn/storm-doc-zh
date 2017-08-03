---
title: Storm MongoDB Integration
layout: documentation
documentation: true
---

Storm/Trident集成[MongoDB](https://www.mongodb.org/)。该包中包括核心bolts和trident states，允许storm topology将storm tuples插入到数据库集合中，或者针storm topology中的数据库集合执行更新查询。

## Insert into Database
此包中包含用于将数据插入数据库集合的bolt和trident state。

### MongoMapper
使用MongoDB在集合中插入数据的主要API是 `org.apache.storm.mongodb.common.mapper.MongoMapper` 接口：

```java
public interface MongoMapper extends Serializable {
    Document toDocument(ITuple tuple);
}
```

### SimpleMongoMapper
`storm-mongodb`包括一个通用的`MongoMapper`实现，称为`SimpleMongoMapper`，可以将Storm元组映射到一个数据库文件。
`SimpleMongoMapper`假定storm tuple具有与您要写入的数据库集合中的文档字段名称相同的字段。

```java
public class SimpleMongoMapper implements MongoMapper {
    private String[] fields;

    @Override
    public Document toDocument(ITuple tuple) {
        Document document = new Document();
        for(String field : fields){
            document.append(field, tuple.getValueByField(field));
        }
        return document;
    }

    public SimpleMongoMapper withFields(String... fields) {
        this.fields = fields;
        return this;
    }
}
```

### MongoInsertBolt
要使用`MongoInsertBolt`，您可以通过指定url，collectionName和将 storm tuple转换为DB文档的 `MongoMapper`实现来构造它的一个实例。 以下是标准的URI连接方案：
 `mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]`

有关Mongo URI的更多选项信息（例如：写关注选项），您可以访问
https://docs.mongodb.org/manual/reference/connection-string/#connections-connection-options

 ```java
String url = "mongodb://127.0.0.1:27017/test";
String collectionName = "wordcount";

MongoMapper mapper = new SimpleMongoMapper()
        .withFields("word", "count");

MongoInsertBolt insertBolt = new MongoInsertBolt(url, collectionName, mapper);
 ```

### MongoTridentState
我们还支持在trident topologies中持久化trident state 。 要创建一个Mongo持久的trident state，您需要使用url，collectionName，“MongoMapper”实例初始化它。 见下面的例子：

 ```java
        MongoMapper mapper = new SimpleMongoMapper()
                .withFields("word", "count");

        MongoState.Options options = new MongoState.Options()
                .withUrl(url)
                .withCollectionName(collectionName)
                .withMapper(mapper);

        StateFactory factory = new MongoStateFactory(options);

        TridentTopology topology = new TridentTopology();
        Stream stream = topology.newStream("spout1", spout);

        stream.partitionPersist(factory, fields,  new MongoStateUpdater(), new Fields());
 ```
 **NOTE**:
 >如果没有提供唯一的索引，在发生故障的情况下，trident state插入可能会导致重复的文档。

## Update from Database
包中包含用于从数据库集合更新数据的bolt。

### SimpleMongoUpdateMapper
`storm-mongodb`包括一个通用的`MongoMapper`实现，称为`SimpleMongoUpdateMapper`，可以将Storm元组映射到数据库文档。 `SimpleMongoUpdateMapper`假定风暴元组具有与您要写入的数据库集合中的文档字段名称相同的字段。
`SimpleMongoUpdateMapper`使用`$ set`运算符来设置文档中字段的值。 有关更新操作的更多信息，可以访问
https://docs.mongodb.org/manual/reference/operator/update/

```java
public class SimpleMongoUpdateMapper implements MongoMapper {
    private String[] fields;

    @Override
    public Document toDocument(ITuple tuple) {
        Document document = new Document();
        for(String field : fields){
            document.append(field, tuple.getValueByField(field));
        }
        return new Document("$set", document);
    }

    public SimpleMongoUpdateMapper withFields(String... fields) {
        this.fields = fields;
        return this;
    }
}
```


 
### QueryFilterCreator
用于创建MongoDB查询过滤器的主要API是 `org.apache.storm.mongodb.common.QueryFilterCreator` 接口：

 ```java
public interface QueryFilterCreator extends Serializable {
    Bson createFilter(ITuple tuple);
}
 ```

### SimpleQueryFilterCreator
`storm-mongodb`包括一个通用的`QueryFilterCreator`实现，称为`SimpleQueryFilterCreator`，可以通过给定的Tuple创建一个MongoDB查询过滤器。 `QueryFilterCreator`使用`$ eq`运算符匹配等于指定值的值。 有关查询运算符的更多信息，可以访问
https://docs.mongodb.org/manual/reference/operator/query/

 ```java
public class SimpleQueryFilterCreator implements QueryFilterCreator {
    private String field;
    
    @Override
    public Bson createFilter(ITuple tuple) {
        return Filters.eq(field, tuple.getValueByField(field));
    }

    public SimpleQueryFilterCreator withField(String field) {
        this.field = field;
        return this;
    }

}
 ```

### MongoUpdateBolt
要使用`MongoUpdateBolt`，你可以通过指定Mongo url，collectionName，一个`QueryFilterCreator`实现和一个````MongoMapper``实现来将storm tuple转换成DB文档来构造一个实例。

 ```java
        MongoMapper mapper = new SimpleMongoUpdateMapper()
                .withFields("word", "count");

        QueryFilterCreator updateQueryCreator = new SimpleQueryFilterCreator()
                .withField("word");
        
        MongoUpdateBolt updateBolt = new MongoUpdateBolt(url, collectionName, updateQueryCreator, mapper);

        //if a new document should be inserted if there are no matches to the query filter
        //updateBolt.withUpsert(true);
 ```
 
 或者为 `QueryFilterCreator`使用匿名内部类实现：
 
  ```java
        MongoMapper mapper = new SimpleMongoUpdateMapper()
                .withFields("word", "count");

        QueryFilterCreator updateQueryCreator = new QueryFilterCreator() {
            @Override
            public Bson createFilter(ITuple tuple) {
                return Filters.gt("count", 3);
            }
        };
        
        MongoUpdateBolt updateBolt = new MongoUpdateBolt(url, collectionName, updateQueryCreator, mapper);

        //if a new document should be inserted if there are no matches to the query filter
        //updateBolt.withUpsert(true);
 ```

