---
title: Storm Druid 集成
layout: documentation
documentation: true
---

# Storm Druid Bolt 和 TridentState

该模块提供了将数据写入[Druid](http://druid.io/) 数据存储的核心Strom和Trident bolt(螺栓)的实现。
该实现使用Druid's的[Tranquility库](https://github.com/druid-io/tranquility)向druid发送消息。

一些实施细节从现有的借用 [Tranquility Storm Bolt](https://github.com/druid-io/tranquility/blob/master/docs/storm.md).
这个新的Bolt(螺栓)增加了支持最新的storm释放，并保持在storm回购的bolt(螺栓)。

### Core Bolt
下面的例子描述了使用 `org.apache.storm.druid.bolt.DruidBeamBolt`的核心bolt(螺栓)默认情况下，该bolt(螺栓)希望收到元组，其中"事件"字段提供您的事件类型。可以通过实现ITupleDruidEventMapper接口来更改此逻辑。

```java

   DruidBeamFactory druidBeamFactory = new SampleDruidBeamFactoryImpl(new HashMap<String, Object>());
   DruidConfig druidConfig = DruidConfig.newBuilder().discardStreamId(DruidConfig.DEFAULT_DISCARD_STREAM_ID).build();
   ITupleDruidEventMapper<Map<String, Object>> eventMapper = new TupleDruidEventMapper<>(TupleDruidEventMapper.DEFAULT_FIELD_NAME);
   DruidBeamBolt<Map<String, Object>> druidBolt = new DruidBeamBolt<Map<String, Object>>(druidBeamFactory, eventMapper, druidConfig);
   topologyBuilder.setBolt("druid-bolt", druidBolt).shuffleGrouping("event-gen");
   topologyBuilder.setBolt("printer-bolt", new PrinterBolt()).shuffleGrouping("druid-bolt" , druidConfig.getDiscardStreamId());

```


### Trident State

```java
    DruidBeamFactory druidBeamFactory = new SampleDruidBeamFactoryImpl(new HashMap<String, Object>());
    ITupleDruidEventMapper<Map<String, Object>> eventMapper = new TupleDruidEventMapper<>(TupleDruidEventMapper.DEFAULT_FIELD_NAME);

    final Stream stream = tridentTopology.newStream("batch-event-gen", new SimpleBatchSpout(10));

    stream.peek(new Consumer() {
        @Override
        public void accept(TridentTuple input) {
             LOG.info("########### Received tuple: [{}]", input);
         }
    }).partitionPersist(new DruidBeamStateFactory<Map<String, Object>>(druidBeamFactory, eventMapper), new Fields("event"), new DruidBeamStateUpdater());

```

### 样品工厂实现
Druid bolt 必须配置一个 BeamFactory. 您可以使用它们其中一个来实现 [DruidBeams builder's] (https://github.com/druid-io/tranquility/blob/master/core/src/main/scala/com/metamx/tranquility/druid/DruidBeams.scala) "buildBeam()" method.
See the [Configuration documentation](https://github.com/druid-io/tranquility/blob/master/docs/configuration.md) for details.
For more details refer [Tranquility library](https://github.com/druid-io/tranquility) docs.

```java

public class SampleDruidBeamFactoryImpl implements DruidBeamFactory<Map<String, Object>> {

    @Override
    public Beam<Map<String, Object>> makeBeam(Map<?, ?> conf, IMetricsContext metrics) {


        final String indexService = "druid/overlord"; // The druid.service name of the indexing service Overlord node.
        final String discoveryPath = "/druid/discovery"; // Curator service discovery path. config: druid.discovery.curator.path
        final String dataSource = "test"; //The name of the ingested datasource. Datasources can be thought of as tables.
        final List<String> dimensions = ImmutableList.of("publisher", "advertiser");
        List<AggregatorFactory> aggregators = ImmutableList.<AggregatorFactory>of(
                new CountAggregatorFactory(
                        "click"
                )
        );
        // Tranquility needs to be able to extract timestamps from your object type (in this case, Map<String, Object>).
        final Timestamper<Map<String, Object>> timestamper = new Timestamper<Map<String, Object>>()
        {
            @Override
            public DateTime timestamp(Map<String, Object> theMap)
            {
                return new DateTime(theMap.get("timestamp"));
            }
        };

        // Tranquility uses ZooKeeper (through Curator) for coordination.
        final CuratorFramework curator = CuratorFrameworkFactory
                .builder()
                .connectString((String)conf.get("druid.tranquility.zk.connect")) //take config from storm conf
                .retryPolicy(new ExponentialBackoffRetry(1000, 20, 30000))
                .build();
        curator.start();

        // The JSON serialization of your object must have a timestamp field in a format that Druid understands. By default,
        // Druid expects the field to be called "timestamp" and to be an ISO8601 timestamp.
        final TimestampSpec timestampSpec = new TimestampSpec("timestamp", "auto", null);

        // Tranquility needs to be able to serialize your object type to JSON for transmission to Druid. By default this is
        // done with Jackson. If you want to provide an alternate serializer, you can provide your own via ```.objectWriter(...)```.
        // In this case, we won't provide one, so we're just using Jackson.
        final Beam<Map<String, Object>> beam = DruidBeams
                .builder(timestamper)
                .curator(curator)
                .discoveryPath(discoveryPath)
                .location(DruidLocation.create(indexService, dataSource))
                .timestampSpec(timestampSpec)
                .rollup(DruidRollup.create(DruidDimensions.specific(dimensions), aggregators, QueryGranularities.MINUTE))
                .tuning(
                        ClusteredBeamTuning
                                .builder()
                                .segmentGranularity(Granularity.HOUR)
                                .windowPeriod(new Period("PT10M"))
                                .partitions(1)
                                .replicants(1)
                                .build()
                )
                .druidBeamConfig(
                      DruidBeamConfig
                           .builder()
                           .indexRetryPeriod(new Period("PT10M"))
                           .build())
                .buildBeam();

        return beam;
    }
}

```

Example code is available [here.](https://github.com/apache/storm/tree/master/external/storm-druid/src/test/java/org/apache/storm/druid)
