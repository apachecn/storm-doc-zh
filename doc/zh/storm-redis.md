---
title: Storm Redis Integration
layout: documentation
documentation: true
---

Storm/Trident integration for [Redis](http://redis.io/)

Storm/Trident 集成 [Redis](http://redis.io/)

Storm-redis uses Jedis for Redis client.

Storm-redis使用Jedis为Redis客户端。

## 用法

### 如何使用它？

使用它作为一个maven依赖：

```xml
<dependency>
    <groupId>org.apache.storm</groupId>
    <artifactId>storm-redis</artifactId>
    <version>${storm.version}</version>
    <type>jar</type>
</dependency>
```

### 常用Bolt

Storm-redis提供了基本的Bolt实现， ```RedisLookupBolt``` and ```RedisStoreBolt```。

根据名称可以知道其功能，```RedisLookupBolt```使用键从Redis中检索值，而```RedisStoreBolt```将键/值存储到Redis。 一个元组将匹配一个键/值对，您可以将匹配模式定义为“`TupleMapper```。

您还可以从```RedisDataTypeDescription```中选择数据类型来使用。请参考 ```RedisDataTypeDescription.RedisDataType```来查看支持哪些数据类型。在一些数据类型（散列和排序集）中，它需要额外的键和从元组转换的元素成为元素。

这些接口与 ```RedisLookupMapper``` 和 ```RedisStoreMapper```组合，分别适合 ```RedisLookupBolt``` 和```RedisStoreBolt```。

#### RedisLookupBolt示例

```java

class WordCountRedisLookupMapper implements RedisLookupMapper {
    private RedisDataTypeDescription description;
    private final String hashKey = "wordCount";

    public WordCountRedisLookupMapper() {
        description = new RedisDataTypeDescription(
                RedisDataTypeDescription.RedisDataType.HASH, hashKey);
    }

    @Override
    public List<Values> toTuple(ITuple input, Object value) {
        String member = getKeyFromTuple(input);
        List<Values> values = Lists.newArrayList();
        values.add(new Values(member, value));
        return values;
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("wordName", "count"));
    }

    @Override
    public RedisDataTypeDescription getDataTypeDescription() {
        return description;
    }

    @Override
    public String getKeyFromTuple(ITuple tuple) {
        return tuple.getStringByField("word");
    }

    @Override
    public String getValueFromTuple(ITuple tuple) {
        return null;
    }
}

```

```java

JedisPoolConfig poolConfig = new JedisPoolConfig.Builder()
        .setHost(host).setPort(port).build();
RedisLookupMapper lookupMapper = new WordCountRedisLookupMapper();
RedisLookupBolt lookupBolt = new RedisLookupBolt(poolConfig, lookupMapper);
```

####  RedisStoreBolt示例

```java

class WordCountStoreMapper implements RedisStoreMapper {
    private RedisDataTypeDescription description;
    private final String hashKey = "wordCount";

    public WordCountStoreMapper() {
        description = new RedisDataTypeDescription(
            RedisDataTypeDescription.RedisDataType.HASH, hashKey);
    }

    @Override
    public RedisDataTypeDescription getDataTypeDescription() {
        return description;
    }

    @Override
    public String getKeyFromTuple(ITuple tuple) {
        return tuple.getStringByField("word");
    }

    @Override
    public String getValueFromTuple(ITuple tuple) {
        return tuple.getStringByField("count");
    }
}
```

```java

JedisPoolConfig poolConfig = new JedisPoolConfig.Builder()
                .setHost(host).setPort(port).build();
RedisStoreMapper storeMapper = new WordCountStoreMapper();
RedisStoreBolt storeBolt = new RedisStoreBolt(poolConfig, storeMapper);
```

### 非简单的 Bolt

如果您的场景不适合 ```RedisStoreBolt```和 ```RedisLookupBolt```，Storm-redis还提供了 ```AbstractRedisBolt```，让您扩展和应用业务逻辑。

```java

    public static class LookupWordTotalCountBolt extends AbstractRedisBolt {
        private static final Logger LOG = LoggerFactory.getLogger(LookupWordTotalCountBolt.class);
        private static final Random RANDOM = new Random();

        public LookupWordTotalCountBolt(JedisPoolConfig config) {
            super(config);
        }

        public LookupWordTotalCountBolt(JedisClusterConfig config) {
            super(config);
        }

        @Override
        public void execute(Tuple input) {
            JedisCommands jedisCommands = null;
            try {
                jedisCommands = getInstance();
                String wordName = input.getStringByField("word");
                String countStr = jedisCommands.get(wordName);
                if (countStr != null) {
                    int count = Integer.parseInt(countStr);
                    this.collector.emit(new Values(wordName, count));

                    // print lookup result with low probability
                    if(RANDOM.nextInt(1000) > 995) {
                        LOG.info("Lookup result - word : " + wordName + " / count : " + count);
                    }
                } else {
                    // skip  
                    LOG.warn("Word not found in Redis - word : " + wordName);
                }
            } finally {
                if (jedisCommands != null) {
                    returnInstance(jedisCommands);
                }
                this.collector.ack(input);
            }
        }

        @Override
        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            // wordName, count
            declarer.declare(new Fields("wordName", "count"));
        }
    }

```

### Trident State 用法 

1. RedisState和RedisMapState，它提供Jedis接口，仅用于单次重新启动。

2. RedisClusterState和RedisClusterMapState，它们提供JedisCluster接口，仅用于redis集群。

RedisState
```java
        JedisPoolConfig poolConfig = new JedisPoolConfig.Builder()
                                        .setHost(redisHost).setPort(redisPort)
                                        .build();
        RedisStoreMapper storeMapper = new WordCountStoreMapper();
        RedisLookupMapper lookupMapper = new WordCountLookupMapper();
        RedisState.Factory factory = new RedisState.Factory(poolConfig);

        TridentTopology topology = new TridentTopology();
        Stream stream = topology.newStream("spout1", spout);

        stream.partitionPersist(factory,
                                fields,
                                new RedisStateUpdater(storeMapper).withExpire(86400000),
                                new Fields());

        TridentState state = topology.newStaticState(factory);
        stream = stream.stateQuery(state, new Fields("word"),
                                new RedisStateQuerier(lookupMapper),
                                new Fields("columnName","columnValue"));
```

RedisClusterState
```java
        Set<InetSocketAddress> nodes = new HashSet<InetSocketAddress>();
        for (String hostPort : redisHostPort.split(",")) {
            String[] host_port = hostPort.split(":");
            nodes.add(new InetSocketAddress(host_port[0], Integer.valueOf(host_port[1])));
        }
        JedisClusterConfig clusterConfig = new JedisClusterConfig.Builder().setNodes(nodes)
                                        .build();
        RedisStoreMapper storeMapper = new WordCountStoreMapper();
        RedisLookupMapper lookupMapper = new WordCountLookupMapper();
        RedisClusterState.Factory factory = new RedisClusterState.Factory(clusterConfig);

        TridentTopology topology = new TridentTopology();
        Stream stream = topology.newStream("spout1", spout);

        stream.partitionPersist(factory,
                                fields,
                                new RedisClusterStateUpdater(storeMapper).withExpire(86400000),
                                new Fields());

        TridentState state = topology.newStaticState(factory);
        stream = stream.stateQuery(state, new Fields("word"),
                                new RedisClusterStateQuerier(lookupMapper),
                                new Fields("columnName","columnValue"));
```
