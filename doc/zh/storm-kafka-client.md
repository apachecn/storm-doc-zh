---
title: Storm Kafka 集成（0.10.x+）
layout: documentation
documentation: true
---

# 使用 kafka-client jar 进行 Storm Apache Kafka 集成
这部分包含新的 Apache Kafka consumer API.

## 兼容性

Apache Kafka 版本 0.10+

## 写入Kafka
您可以通过创建 org.apache.storm.kafka.bolt.KafkaBolt 实例并将其作为组件附加到您的topology.如果您使用 trident ,您可以通过使用以下对象完成 org.apache.storm.kafka.trident.TridentState, org.apache.storm.kafka.trident.TridentStateFactory and
org.apache.storm.kafka.trident.TridentKafkaUpdater.

您需要为以下两个接口提供实现

###TupleToKafkaMapper 和 TridentTupleToKafkaMapper
这些接口有两个抽象方法:

```java
    K getKeyFromTuple(Tuple/TridentTuple tuple);
    V getMessageFromTuple(Tuple/TridentTuple tuple);
```

顾名思义,这两个方法被调用将tuple映射到Kafka message的key和message本身.  如果你只想要一个字段
作为键和一个字段作为值,那么您可以使用提供的FieldNameBasedTupleToKafkaMapper.java
实现. 在KafkaBolt中,使用默认构造函数构造FieldNameBasedTupleToKafkaMapper需要一个字段名称为"key"和"message"的字段以实现向后兼容. 或者,您也可以使用非默认构造函数指定不同的键和消息字段.
在使用TridentKafkaState 时你必须明确key和message的字段名称,因为TridentKafkaState默认的构造函数没有设置参数.在构造FieldNameBasedTupleToKafkaMapper的实例时应明确这些.

###KafkaTopicSelector 和 trident KafkaTopicSelector
这个接口只有一个方法:

```java
public interface KafkaTopicSelector {
    String getTopics(Tuple/TridentTuple tuple);
}
```

该接口的实现应该要根据tuple的 key/message 返回相应的Kafka的topic,如果返回 null 则该消息将被忽略掉.如果您只需要一个静态topic名称,那么可以使用 DefaultTopicSelector.java 并在构造函数中设置topic的名称.

`FieldNameTopicSelector` 和 `FieldIndexTopicSelector` 用于选择 tuple 要发送到的topic,用户只需要指定tuple中存储 topic名称的字段名称或字段索引即可(即tuple中的某个字段是kafka topic的名称).当topic的名称不存在时, `Field*TopicSelector` 会将tuple写入到默认的topic.请确保默认topic已经在kafka中创建并且在`Field*TopicSelector`正确设置.

### 设置 Kafka producer 属性
你可以在 topology 通过调用 `KafkaBolt.withProducerProperties()` 和 `TridentKafkaStateFactory.withProducerProperties()` 设置kafka producer的所有属性. Kafka producer[配置](http://kafka.apache.org/documentation.html#newproducerconfigs)
选择 "Important configuration properties for the producer" 查看更多详情.
所有的kafka producer配置项的key都在 `org.apache.kafka.clients.producer.ProducerConfig`类中

### 使用通配符匹配 Kafka topic
通过添加如下属性开启通配符匹配(此功能是为了storm可以动态读取多个kafka topic中的数据,并支持动态发现.看相关功能的实现需求[feture](https://issues.apache.org/jira/browse/STORM-817))

```
     Config config = new Config();
     config.put("kafka.topic.wildcard.match",true);

```

之后,您可以指定一个通配符topic,例如clickstream.*.log. 这将匹配clickstream.my.log,clickstream.cart.log等topic


### bolt 和 Trident 的Kafka Producer实现

For the bolt :

```java
        TopologyBuilder builder = new TopologyBuilder();

        Fields fields = new Fields("key", "message");
        FixedBatchSpout spout = new FixedBatchSpout(fields, 4,
                    new Values("storm", "1"),
                    new Values("trident", "1"),
                    new Values("needs", "1"),
                    new Values("javadoc", "1")
        );
        spout.setCycle(true);
        builder.setSpout("spout", spout, 5);
        //set producer properties.
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("acks", "1");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        KafkaBolt bolt = new KafkaBolt()
                .withProducerProperties(props)
                .withTopicSelector(new DefaultTopicSelector("test"))
                .withTupleToKafkaMapper(new FieldNameBasedTupleToKafkaMapper());
        builder.setBolt("forwardToKafka", bolt, 8).shuffleGrouping("spout");

        Config conf = new Config();

        StormSubmitter.submitTopology("kafkaboltTest", conf, builder.createTopology());
```

For Trident:

```java
        Fields fields = new Fields("word", "count");
        FixedBatchSpout spout = new FixedBatchSpout(fields, 4,
                new Values("storm", "1"),
                new Values("trident", "1"),
                new Values("needs", "1"),
                new Values("javadoc", "1")
        );
        spout.setCycle(true);

        TridentTopology topology = new TridentTopology();
        Stream stream = topology.newStream("spout1", spout);

        //set producer properties.
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("acks", "1");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        TridentKafkaStateFactory stateFactory = new TridentKafkaStateFactory()
                .withProducerProperties(props)
                .withKafkaTopicSelector(new DefaultTopicSelector("test"))
                .withTridentTupleToKafkaMapper(new FieldNameBasedTupleToKafkaMapper("word", "count"));
        stream.partitionPersist(stateFactory, fields, new TridentKafkaUpdater(), new Fields());

        Config conf = new Config();
        StormSubmitter.submitTopology("kafkaTridentTest", conf, topology.build());
```

## 读取Kafka (Spouts)

### 配置

spout通过使用`KafkaSpoutConfig`类来指定配置. 此类使用Builder模式,可以通过调用其中一个Builders构造函数或通过调用KafkaSpoutConfig类中的静态方法创建一个Builder.创建builder的构造方法或静态方法需要几个键值（稍后可以更改）,但这是启动一个spout的所需的最小配置

`bootstrapServers`与Kafka Consumer Property"bootstrap.servers"相同.
配置项`topics' 配置的是spout将消费的kafka topic.可以是特定主题名称（1个或多个）的集合列表或正则表达式"Pattern",它指定
任何与正则表达式匹配的主题都将被消费.


在构造函数的情况下,您可能还需要指定key deserializer和value deserializer. 这是为了通过使用Java泛型来保证类型安全. 默认值为"StringDeserializer",可以通过调用"setKeyDeserializer"和"setValueDeserializer"进行覆盖.如果这些设置为null,代码将回退到kafka属性中设置的内容,但最好在这里明确,通过使用Java泛型来确保类型安全.

下面是一些需要特别注意的关键配置项.

`setFirstPollOffsetStrategy`允许你设置从哪里开始消费数据. 这在故障恢复和第一次启动spout的情况下会被使用. 可选的的值包括:

 * `EARLIEST` 无论之前的消费情况如何,spout会从每个kafka partition能找到的最早的offset开始的读取
 * `LATEST` 无论之前的消费情况如何,spout会从每个kafka partition当前最新的offset开始的读取
 * `UNCOMMITTED_EARLIEST` (默认值) spout 会从每个partition的最后一次提交的offset开始读取. 如果offset不存在或者过期, 则会依照 `EARLIEST`进行读取.
 * `UNCOMMITTED_LATEST` spout 会从每个partition的最后一次提交的offset开始读取, 如果offset不存在或者过期, 则会依照 `LATEST `进行读取.

`setRecordTranslator`可以修改spout如何将Kafka消费者message转换为tuple,以及将该tuple发布到哪个stream中.默认情况下,"topic","partition","offset","key"和"value"将被发送到"default"stream. 如果要将条目根据topic输出到不同的stream中,Storm提供了"ByTopicRecordTranslator". 有关如何使用这些的更多示例,请参阅下文.
`setProp`可用于设置kafka属性.
`setGroupId`可以让您设置kafka使用者组属性"group.id".
`setSSLKeystore`和`setSSLTruststore`允许你配置SSL认证.

### 使用举例

API是用java 8 lambda表达式写的. 它也可以用于java7及更低的版本.

#### 创建一个简单的不可靠spout
以下将消费kafka中"demo_topic"的所有消息,并将其发送到MyBolt,其中包含"topic","partition","offset","key","value".

```java

final TopologyBuilder tp = new TopologyBuilder();
tp.setSpout("kafka_spout", new KafkaSpout<>(KafkaSpoutConfig.builder("127.0.0.1:" + port, "demo_topic").build()), 1);
tp.setBolt("bolt", new myBolt()).shuffleGrouping("kafka_spout");
...

```

#### 通配符 Topics
通配符 topics 将消费所有符合通配符的topics.  在下面的例子中
"topic", "topic_foo" 和 "topic_bar" 适配通配符 "topic.*", 但是 "not_my_topic" 并不适配. 

```java

final TopologyBuilder tp = new TopologyBuilder();
tp.setSpout("kafka_spout", new KafkaSpout<>(KafkaSpoutConfig.builder("127.0.0.1:" + port, Pattern.compile("topic.*")).build()), 1);
tp.setBolt("bolt", new myBolt()).shuffleGrouping("kafka_spout");
...


```
#### 多个 Streams

这个案例使用 java 8 lambda 表达式.

```java

final TopologyBuilder tp = new TopologyBuilder();

//默认情况下,spout 消费但未被match到的topic的message的"topic","key"和"value"将发送到"STREAM_1"
ByTopicRecordTranslator<String, String> byTopic = new ByTopicRecordTranslator<>(
    (r) -> new Values(r.topic(), r.key(), r.value()),
    new Fields("topic", "key", "value"), "STREAM_1");
//topic_2 所有的消息的 "key" and "value" 将发送到 "STREAM_2"中
byTopic.forTopic("topic_2", (r) -> new Values(r.key(), r.value()), new Fields("key", "value"), "STREAM_2");

tp.setSpout("kafka_spout", new KafkaSpout<>(KafkaSpoutConfig.builder("127.0.0.1:" + port, "topic_1", "topic_2", "topic_3").build()), 1);
tp.setBolt("bolt", new myBolt()).shuffleGrouping("kafka_spout", "STREAM_1");
tp.setBolt("another", new myOtherBolt()).shuffleGrouping("kafka_spout", "STREAM_2");
...

```

#### Trident

```java
final TridentTopology tridentTopology = new TridentTopology();
final Stream spoutStream = tridentTopology.newStream("kafkaSpout",
    new KafkaTridentSpoutOpaque<>(KafkaSpoutConfig.builder("127.0.0.1:" + port, Pattern.compile("topic.*")).build()))
      .parallelismHint(1)
...

```

Trident不支持多个stream且不支持设置将strem分发到多个output. 并且,如果每个output 的topic的字段不一致会抛出异常而不会继续.

### 自定义 RecordTranslator(高级特性)

在大多数情况下,内置的SimpleRecordTranslator和ByTopicRecordTranslator应该满足您的使用. 如果您遇到需要定制的情况那么这个文档将会描述如何正确地做到这一点,涉及到一些不太常用的类.适用的要点是使用ConsumerRecord并将其转换为可以emitted 的"List <Object>". 难点是如何告诉spout将其发送到指定的stream中. 为此,您将需要返回一个"org.apache.storm.kafka.spout.KafkaTuple"的实例. 这提供了一个方法`routedTo`,它将说明tuple将要发送到哪个特定stream.

For Example:

```java
return new KafkaTuple(1, 2, 3, 4).routedTo("bar");
```
将会使tuple发送到"bar" stream中.

在编写自定义record translators时要小心,因为在Storm spout
中,它需要自我一致. `streams`方法应该返回这个translator将会尝试发到streams的set列表. 另外,`getFieldsFor`应该为每一个stream 返回一个有效的Fields对象(就是说通过字段名称可以拿到对应的正确的对象). 如果您使用Trident执行此操作,则Fields对象中指定字段的所有值必须在stream名称的List中,否则trident抛出异常.（原文:If you are doing this for Trident a value must be in the List returned by apply for every field in the Fields object for that stream）


### 手动分区控制 (高级特性)

默认情况下,Kafka将自动将partition分配给当前的一组spouts. 它处理很多事情,但在某些情况下,您可能需要手动分配partition.当spout 挂掉并重新启动,但如果处理不正确,可能会导致很多问题. 这可以通过子类化Subscription来处理,我们有几个实现,您可以查看有关如何执行此操作的示例. ManualPartitionNamedSubscription和ManualPartitionPatternSubscription. 再次强调,使用这些或自己实现时请务必注意.

## 使用Maven Shade Plugin构建Uber Jar

Add the following to `REPO_HOME/storm/external/storm-kafka-client/pom.xml`
```xml
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
                        <mainClass>org.apache.storm.kafka.spout.test.KafkaSpoutTopologyMain</mainClass>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

执行命令生成 uber jar:

`mvn package -f REPO_HOME/storm/external/storm-kafka-client/pom.xml`

uber jar 文件会生成在如下目录中:
 
`REPO_HOME/storm/external/storm-kafka-client/target/storm-kafka-client-1.0.x.jar`

### 运行 Storm Topology

复制`REPO_HOME/storm/external/storm-kafka-client/target/storm-kafka-client-*.jar` 到 `STORM_HOME/extlib`

使用kafka 命令行工具创建topic [test, test1, test2] 并使用 Kafka console producer 向topic添加数据

执行命令 `STORM_HOME/bin/storm jar REPO_HOME/storm/external/storm/target/storm-kafka-client-*.jar org.apache.storm.kafka.spout.test.KafkaSpoutTopologyMain`

开启debug级别日志可以看到每个topic的消息根据设定的stream和设定的shuffle grouping被重定向到相应的spout.

## Using storm-kafka-client with different versions of kafka

Storm-kafka客户端的Kafka依赖关系在maven中被定义为`provided`,这意味着它不会被拉入
作为传递依赖. 这允许您使用与您的kafka集群兼容的Kafka依赖版本.

当使用storm-kafka-client构建项目时,必须显式添加Kafka clients依赖关系. 例如,使用Kafka client 0.10.0.0,您将使用以下依赖 `pom.xml`:

```xml
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>0.10.0.0</version>
        </dependency>
```

你也可以在使用maven build时通过指定参数`storm.kafka.client.version` 来指定 kafka clients 版本
e.g. `mvn clean install -Dstorm.kafka.client.version=0.10.0.0`

选择kafka client版本时,您应该确保 - 
 1. kafka api是兼容的. storm-kafka-client模块仅支持** 0.10或更新的** kafka客户端API. 对于旧版本,
  您可以使用storm-kafka模块 (https://github.com/apache/storm/tree/master/external/storm-kafka).  
 2. 您选择的kafka client 应与broker兼容. 例如 0.9.x client 将无法使用
  0.8.x broker

#Kafka Spout 性能调整

Kafka spout 提供了两个内置参数来调节其性能. 参数可以通过 [KafkaSpoutConfig] (https://github.com/apache/storm/blob/1.0.x-branch/external/storm-kafka-client/src/main/java/org/apache/storm/kafka/spout/KafkaSpoutConfig.java) 的 [setOffsetCommitPeriodMs] (https://github.com/apache/storm/blob/1.0.x-branch/external/storm-kafka-client/src/main/java/org/apache/storm/kafka/spout/KafkaSpoutConfig.java#L189-L193) 和 [setMaxUncommittedOffsets] (https://github.com/apache/storm/blob/1.0.x-branch/external/storm-kafka-client/src/main/java/org/apache/storm/kafka/spout/KafkaSpoutConfig.java#L211-L217). 方法进行设置

* "offset.commit.period.ms" 控制spout多久向kafka注册一次offset
* "max.uncommitted.offsets" 控制没读取多少条message向kafka注册一次offset
<br/>

[Kafka consumer config] (http://kafka.apache.org/documentation.html#consumerconfigs) 参数也可能对spout的性能产生影响. 以下Kafka参数可能是spout性能中影响最大的一些参数：

* "fetch.min.bytes"
* "fetch.max.wait.ms"
* [Kafka Consumer] (http://kafka.apache.org/090/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html) Kafka spout 使用 [KafkaSpoutConfig] (https://github.com/apache/storm/blob/1.0.x-branch/external/storm-kafka-client/src/main/java/org/apache/storm/kafka/spout/KafkaSpoutConfig.java) 的 [setPollTimeoutMs] (https://github.com/apache/storm/blob/1.0.x-branch/external/storm-kafka-client/src/main/java/org/apache/storm/kafka/spout/KafkaSpoutConfig.java#L180-L184)方法设置读取数据的超时时间
<br/>
根据您的Kafka群集的结构,数据的分布和数据的可用性,这些参数必须正确配置. 请参考关于Kafka参数调整的Kafka文档.

### kafka spout配置默认值

目前 Kafka spout 有如下默认值,这在[blog post] (https://hortonworks.com/blog/microbenchmarking-storm-1-0-performance/)所述的测试环境中表现出了良好的性能 

* poll.timeout.ms = 200
* offset.commit.period.ms = 30000   (30s)
* max.uncommitted.offsets = 10000000
<br/>

# Kafka 自动提交offset模式 

如果可靠性对您不重要 - 也就是说,您不关心在失败情况下丢失tuple,并且要消除tuple跟踪的开销,那么您可以使用AutoCommitMode运行KafkaSpout.

你需要开启自动提交模式:
* 设置 Config.TOPOLOGY_ACKERS 为 0;
* 在Kafka consumer 配置中开启 *AutoCommitMode* ; 

下面是一个在KafkaSpout中开启AutoCommitMode的例子:

```java
KafkaSpoutConfig<String, String> kafkaConf = KafkaSpoutConfig
		.builder(String bootstrapServers, String ... topics)
		.setProp(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true")
		.setFirstPollOffsetStrategy(FirstPollOffsetStrategy.EARLIEST)
		.build();
```

*请注意,由于Kafka消费者定期进行提交offset,所以并不是完全符合 `At-Most-Once`.因为在KafkaSpout崩溃时,可能会重读某些tuple.*



