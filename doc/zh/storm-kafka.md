---
title: Storm Kafka Integration
layout: documentation
documentation: true
---

提供核心的 Storm 和Trident 的spout实现，用来从Apache Kafka 0.8x版本消费数据.

##Spouts
我们支持 Trident 和 core Storm 的spout.对于这两种spout实现，我们使用BorkerHosts接口来跟踪Kafka broker host partition 映射关系，用KafkaConfig来控制Kafka 相关参数.

###BrokerHosts
为了初始化 Kafka spout/emitter，你需要构造一个 BrokerHosts 标记接口的实例。当前，我们支持以下两种实现方式.

####ZkHosts
如果你想要动态的跟踪Kafka broker partition 映射关系，你应该使用ZkHosts。这个类使用 Kafka Zookeeper实体跟踪 brokerHost->分区映射.
你可以调用下面的方法来得到一个实例.
```java
    public ZkHosts(String brokerZkStr, String brokerZkPath)
    public ZkHosts(String brokerZkStr)
```
ZkStr 字符串格式是 ip:port（例如：localhost:2181）.brokerZkPath 是存储所有 topic 和 partition信息的zk 根路径.默认情况下，Kafka使用 /brokers路径.

默认情况下，broker-partition 映射关系60s秒从Zookeeper刷新一次.如果你想要改变这个时间，你需要设置 host.refreshFreqSecs 配置.

####StaticHosts
这是一种可替代的实现，broker->partition 信息是静态的.要构造这个类的实例，你需要先构造一个 GlobalPartitionInformation 的实例.

```java
    Broker brokerForPartition0 = new Broker("localhost");//localhost:9092
    Broker brokerForPartition1 = new Broker("localhost", 9092);//localhost:9092 but we specified the port explicitly
    Broker brokerForPartition2 = new Broker("localhost:9092");//localhost:9092 specified as one string.
    GlobalPartitionInformation partitionInfo = new GlobalPartitionInformation();
    partitionInfo.addPartition(0, brokerForPartition0);//mapping from partition 0 to brokerForPartition0
    partitionInfo.addPartition(1, brokerForPartition1);//mapping from partition 1 to brokerForPartition1
    partitionInfo.addPartition(2, brokerForPartition2);//mapping from partition 2 to brokerForPartition2
    StaticHosts hosts = new StaticHosts(partitionInfo);
```

###KafkaConfig
构造一个KafkaSpout的实例，第二件事情就是要实例化KafkaConfig。
```java
    public KafkaConfig(BrokerHosts hosts, String topic)
    public KafkaConfig(BrokerHosts hosts, String topic, String clientId)
```

BrokerHosts可以通过多个BrokerHosts接口实现.topic 就是Kafka topic 的名称.可选择的ClientId就是当前消费的offset存储的zk的路径.

有两个KafkaConfig 继承类正在被使用.

Spoutconfig是KafkaConfig的扩展，它支持Zookeeper 连接信息的其他字段，并且可以控制KafkaSpout的行为.Zkroot就是用来存储消费者offset信息的根路径.id是唯一的，用来标识spout.
```java
public SpoutConfig(BrokerHosts hosts, String topic, String zkRoot, String id);
public SpoutConfig(BrokerHosts hosts, String topic, String id);
```
除此之外，SpoutConfig包含下面这些字段，用来控制KafkaSpout的行为：
```java
    // setting for how often to save the current Kafka offset to ZooKeeper
    public long stateUpdateIntervalMs = 2000;

    // Retry strategy for failed messages
    public String failedMsgRetryManagerClass = ExponentialBackoffMsgRetryManager.class.getName();

    // Exponential back-off retry settings.  These are used by ExponentialBackoffMsgRetryManager for retrying messages after a bolt
    // calls OutputCollector.fail(). These come into effect only if ExponentialBackoffMsgRetryManager is being used.
    // Initial delay between successive retries
    public long retryInitialDelayMs = 0;
    public double retryDelayMultiplier = 1.0;
    
    // Maximum delay between successive retries    
    public long retryDelayMaxMs = 60 * 1000;
    // Failed message will be retried infinitely if retryLimit is less than zero. 
    public int retryLimit = -1;     

```
核心KafkaSpout只接口一个SpoutConfig实例

TridentKafkaConfig是KafkaConfig的另外一个扩展. 
TridentKafkaEmitter只接受TridentKafkaConfig作为参数.

KafkaConfig类也有一些公共变量来控制你的应用程序的行为。以下是默认值：
```java
    public int fetchSizeBytes = 1024 * 1024;
    public int socketTimeoutMs = 10000;
    public int fetchMaxWait = 10000;
    public int bufferSizeBytes = 1024 * 1024;
    public MultiScheme scheme = new RawMultiScheme();
    public boolean ignoreZkOffsets = false;
    public long startOffsetTime = kafka.api.OffsetRequest.EarliestTime();
    public long maxOffsetBehind = Long.MAX_VALUE;
    public boolean useStartOffsetTimeIfOffsetOutOfRange = true;
    public int metricsTimeBucketSizeInSecs = 60;
```

除MultiScheme之外，大部分都可以读命名就可以理解。
###MultiScheme
MultiScheme 是一个用来规定 ByteBuffer 如何Kafka 消费，并转换成一个 storm tuple.并且会控制 output field的命名.

```java
  public Iterable<List<Object>> deserialize(ByteBuffer ser);
  public Fields getOutputFields();
```

默认的 `RawMultiScheme` 接受 `ByteBuffer` 参数，并返回一个 tuple.就是将ByteBuffer 转换成 `byte[]`.outPutField 的名称是 “bytes”。还有可替换的的实现，像 `SchemeAsMultiScheme` 和 `KeyValueSchemeAsMultiScheme`，他们会将 `ByteBuffer` 转换成 `String`.

当然还有个`SchemeAsMultiScheme` 的扩展类，`MessageMetadataSchemeAsMultiScheme`，MessageMetadataSchemeAsMultiScheme有一个额外的反序列化方法，会接受ByteBuffer 信息，还会伴随着`Partition` 和 `offset` 信息.


```java
public Iterable<List<Object>> deserializeMessageWithMetadata(ByteBuffer message, Partition partition, long offset)

```

上面这个方法对于审计/重新处理Kafka topic上任意一个点的消息非常有用，保存了每条消息的partition和offset,而不是保留整个消息.

###Failed message retry
FailedMsgRetryManager是一个定义失败消息的重试策略的接口。默认实现是ExponentialBackoffMsgRetryManager，它在连续重试之间以指数延迟重试。要使用自定义实现，请将SpoutConfig.failedMsgRetryManagerClass设置为完整的实现类名称。下面是接口：
```java
    // Spout initialization can go here. This can be called multiple times during lifecycle of a worker. 
    void prepare(SpoutConfig spoutConfig, Map stormConf);

    // Message corresponding to offset has failed. This method is called only if retryFurther returns true for offset.
    void failed(Long offset);

    // Message corresponding to offset has been acked.  
    void acked(Long offset);

    // Message corresponding to the offset, has been re-emitted and under transit.
    void retryStarted(Long offset);

    /**
     * The offset of message, which is to be re-emitted. Spout will fetch messages starting from this offset
     * and resend them, except completed messages.
     */
    Long nextFailedMessageToRetry();

    /**
     * @return True if the message corresponding to the offset should be emitted NOW. False otherwise.
     */
    boolean shouldReEmitMsg(Long offset);

    /**
     * Spout will clean up the state for this offset if false is returned. If retryFurther is set to true,
     * spout will called failed(offset) in next call and acked(offset) otherwise 
     */
    boolean retryFurther(Long offset);

    /**
     * Clear any offsets before kafkaOffset. These offsets are no longer available in kafka.
     */
    Set<Long> clearOffsetsBefore(Long kafkaOffset);
``` 

#### Version incompatibility
在1.0之前的Storm版本中，MultiScheme方法接受一个 `byte []` 而不是 `ByteBuffer`。 MultScheme和相关的方案apis在版本1.0中被更改为接受ByteBuffer而不是byte []。

这意味着，在1.0版及更高版本之前，1.0版的kafka spouts将无法使用。在Storm 1.0及更高版本中运行拓扑时，必须确保storm-kafka版本至少为1.0。1.0之前的 topology jar 必须重新和storm-kafka 1.0版本构建，以便在Storm 1.0及更高版本的群集中运行。
### Examples

#### Core Spout

```java
BrokerHosts hosts = new ZkHosts(zkConnString);
SpoutConfig spoutConfig = new SpoutConfig(hosts, topicName, "/" + topicName, UUID.randomUUID().toString());
spoutConfig.scheme = new SchemeAsMultiScheme(new StringScheme());
KafkaSpout kafkaSpout = new KafkaSpout(spoutConfig);
```

#### Trident Spout
```java
TridentTopology topology = new TridentTopology();
BrokerHosts zk = new ZkHosts("localhost");
TridentKafkaConfig spoutConf = new TridentKafkaConfig(zk, "test-topic");
spoutConf.scheme = new SchemeAsMultiScheme(new StringScheme());
OpaqueTridentKafkaSpout spout = new OpaqueTridentKafkaSpout(spoutConf);
```


### How KafkaSpout stores offsets of a Kafka topic and recovers in case of failures

如上面的KafkaConfig属性所示，您可以通过设置 `KafkaConfig.startOffsetTime` 来控制从Kafka topic 的哪个端口开始读取，如下所示：

1. `kafka.api.OffsetRequest.EarliestTime()`: 从topic 初始位置读取消息 (例如，从最老的那个消息开始)
2. `kafka.api.OffsetRequest.LatestTime()`: 从topic尾部开始读取消息 (例如，新写入topic的信息)
3. 一个Unix时间戳，从当前 epoch 开始.（例如，可以通过System.currentTimeMillis(）），具体的可以查看Kafka FAQ中的 [How do I accurately get offsets of messages for a certain timestamp using OffsetRequest?](https://cwiki.apache.org/confluence/display/KAFKA/FAQ#FAQ-HowdoIaccuratelygetoffsetsofmessagesforacertaintimestampusingOffsetRequest?) .


当topology（拓扑）运行Kafka Spout ，并跟踪读取和发送的offset，并将状态信息存储到zk path  `SpoutConfig.zkRoot+ "/" + SpoutConfig.id`.在故障的情况下，它会从ZooKeeper的最后一次写入偏移中恢复。

> **Important:**  新部署topology（拓扑）时，请确保`SpoutConfig.zkRoot`和`SpoutConfig.id`的设置未被修改，
> 否则spout将无法从ZooKeeper中读取以前的消费者状态信息（即偏移量）导致意外的行为和/或数据丢失，具体取决于您的用例。


这意味着当topology（拓扑）运行一旦设置`KafkaConfig.startOffsetTime`将不会对 topology（拓扑）的后续运行产生影响，
因为现在 topology（拓扑）将依赖于ZooKeeper中的消费者状态信息（偏移量）来确定从哪里开始（更多准确地：简历）阅读。
如果要强制该端口忽略存储在ZooKeeper中的任何消费者状态信息，则应将参数`KafkaConfig.ignoreZkOffsets` 设置为true。如果为`true`，
则如上所述，spout 将始终从`KafkaConfig.startOffsetTime`定义的偏移量开始读取。


## Using storm-kafka with different versions of Kafka

Storm-kafka的Kafka依赖关系在maven中scope 定义为 `provided` ，这意味着它不会被作为传递依赖。这允许您使用与Kafka集群兼容的Kafka依赖关系版本。

当使用storm-kafka构建项目时，必须明确地添加Kafka依赖项。例如，要使用针对Scala 2.10构建的Kafka 0.8.1.1，您将在 `pom.xml` 中使用以下依赖关系：

```xml
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.10</artifactId>
            <version>0.8.1.1</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.zookeeper</groupId>
                    <artifactId>zookeeper</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```

请注意，排除ZooKeeper和log4j依赖关系以防止与Storm的依赖关系发生版本冲突。

您还可以覆盖从maven构建的kafka依赖关系版本，其中包含参数`storm.kafka.version`和`storm.kafka.artifact.id`，例如`mvn clean install -Dstorm.kafka.artifact.id = kafka_2.11 -Dstorm.kafka.version = 0.9.0.1`

选择kafka依赖版本时，您应该确保  -
1. kafka api与storm-kafka兼容。目前，storm-kafka模块仅支持0.9.x和0.8.x客户端API。如果要使用更高版本，应该使用storm-kafka-client模块替换。
2. 您选择的kafka客户端应与 broker 兼容。例如0.9.x客户端将无法使用0.8.x broker。


##Writing to Kafka as part of your topology
您可以创建一个org.apache.storm.kafka.bolt.KafkaBolt的实例，并将其作为组件附加到 topology（拓扑）中，或者如果您使用Trident，则可以使用org.apache.storm.kafka.trident.TridentState，org.apache .storm.kafka.trident.TridentStateFactory和org.apache.storm.kafka.trident.TridentKafkaUpdater。

您需要提供以下2个接口的实现:

###TupleToKafkaMapper and TridentTupleToKafkaMapper
这个接口有下面两个方法:

```java
    K getKeyFromTuple(Tuple/TridentTuple tuple);
    V getMessageFromTuple(Tuple/TridentTuple tuple);
```

顾名思义，这些方法被称为将 tuple 映射到Kafka key 和Kafka消息。
如果您只需要一个字段作为键和一个字段作为值，则可以使用提供的FieldNameBasedTupleToKafkaMapper.java实现。
在KafkaBolt中，如果使用默认构造函数构造FieldNameBasedTupleToKafkaMapper，则实现始终会查找字段名称为“key”和“message”的字段，以实现向后兼容性的原因。
或者，您也可以使用非默认构造函数指定不同的键和消息字段。在TridentKafkaState中，您必须指定键和消息的字段名称，因为没有默认构造函数。
在构造FieldNameBasedTupleToKafkaMapper实例时应该指定这些。

###KafkaTopicSelector and trident KafkaTopicSelector
This interface has only one method
```java
public interface KafkaTopicSelector {
    String getTopics(Tuple/TridentTuple tuple);
}
```
该接口的实现应该返回要发送 tuple的密钥/消息映射的topic,您可以返回一个null，该消息将被忽略。
如果您有一个静态的topic 名称，那么可以使用DefaultTopicSelector.java并在构造函数中设置主题的名称。
 `FieldNameTopicSelector`和`FieldIndexTopicSelector`用于支持决定哪个topic 应该从tuple 送消息。
 用户可以在tuple中指定字段名称或字段索引，selector将使用该值作为发布消息的topic 名称。
当找不到topic 名称时，`KafkaBolt`会将消息写入默认topic。请确保已创建默认topic。

### Specifying Kafka producer properties
`TridentKafkaStateFactory.withProducerProperties（）`来提供Storm拓扑中的所有生产属性。有关详细信息，请参阅http://kafka.apache.org/documentation.html#newproducerconfigs“ producer 的重要配置属性”部分。

###Using wildcard kafka topic match
您可以通过添加以下配置来进行通配符 topic 匹配
```
     Config config = new Config();
     config.put("kafka.topic.wildcard.match",true);

```

之后，您可以指定一个通配符 topic ，以匹配例如点击流。*记录。这将匹配所有流匹配clickstream.my.log，clickstream.cart.log等


###Putting it all together

对于bolt:
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

对于 Trident:

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


### Committer Sponsors
P. Taylor Goetz (ptgoetz@apache.org)