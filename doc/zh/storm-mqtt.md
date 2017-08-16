---
title: Storm MQTT(Message Queuing Telemetry Transport, 消息队列遥测传输) 集成
layout: documentation
documentation: true
---

## About
MQTT是IoT(Internet of Things)应用程序中经常使用的轻量经发布/订阅协议。

Further information can be found at http://mqtt.org. The HiveMQ website has a great series on 
[MQTT Essentials](http://www.hivemq.com/mqtt-essentials/).

功能点如下：

* 完整的MQTT支持 (e.g. last will, QoS 0-2, retain, etc.)
* Spout(喷口) 实现订阅MQTT主题
* 用于发布MQTT消息的bolt(螺栓)实现
* 用于发布MQTT消息的trident(三叉线)功能实现 
* 身份验证和TLS/SSL支持
* 用户定义的"映射器"用于将MQTT消息转换为元组(订阅者)
* 用户定义的"映射器" User-defined "mappers" 用于将元组转换为MQTT消息(发布者)


## 快速开始
要快速查看MQTT集成操作，请按照以下说明进行操作。

**启动 MQTT 的代理和发布者**

以下命令将在端口1883上创建一个MQTT代码，并启动一个随机发布的发布者温度/湿度值到MQTT主题。

打开终端并执行以下命令(根据需要更改路径)：

```bash
java -cp examples/target/storm-mqtt-examples-*-SNAPSHOT.jar org.apache.storm.mqtt.examples.MqttBrokerPublisher
```

**运行示例拓扑**

使用Flux运行示例拓扑。这将启动由MQTT Spout(喷口)组成的本地模式集群和拓扑发布到只收录信息的
bolt(螺栓)。

在单独的终端中，运行以下命令(请注意，"storm" 可执行文件必须位于你的PATH上)：

```bash
storm jar ./examples/target/storm-mqtt-examples-*-SNAPSHOT.jar org.apache.storm.flux.Flux ./examples/src/main/flux/sample.yaml --local
```

您应该可以看到来自MQTT的数据被bolt(螺栓)记录：

```
27020 [Thread-17-log-executor[3 3]] INFO  o.a.s.f.w.b.LogInfoBolt - {user=tgoetz, deviceId=1234, location=office, temperature=67.0, humidity=65.0}
27030 [Thread-17-log-executor[3 3]] INFO  o.a.s.f.w.b.LogInfoBolt - {user=tgoetz, deviceId=1234, location=office, temperature=47.0, humidity=85.0}
27040 [Thread-17-log-executor[3 3]] INFO  o.a.s.f.w.b.LogInfoBolt - {user=tgoetz, deviceId=1234, location=office, temperature=69.0, humidity=94.0}
27049 [Thread-17-log-executor[3 3]] INFO  o.a.s.f.w.b.LogInfoBolt - {user=tgoetz, deviceId=1234, location=office, temperature=4.0, humidity=98.0}
27059 [Thread-17-log-executor[3 3]] INFO  o.a.s.f.w.b.LogInfoBolt - {user=tgoetz, deviceId=1234, location=office, temperature=51.0, humidity=12.0}
27069 [Thread-17-log-executor[3 3]] INFO  o.a.s.f.w.b.LogInfoBolt - {user=tgoetz, deviceId=1234, location=office, temperature=27.0, humidity=65.0}
```

允许本地集群退出，或者通过键入Cntrl-C 来停止。

**MQTT容错实战**

在拓扑关闭之后，MQTT Spout(喷口)创建的MQTT订阅将与代理持续存在，并且它将继续接收和排队消息(只要代理正在运行)。

如果您再次运行拓扑(当代理程序仍在运行时)，当spout(喷口)最初连接到MQTT代理时，它会收到所有的信息，
错过的信息就当作失败的消息。您应该看到这是一个消息的爆发，接着再以每秒两个消息的速率显示。

这是因为在默认情况下，MQTT Spout(喷口)在订阅时会创建一个*会话*这意味着它要求它的代理在离线时持有并重新
提交其错过的任何消息。另外一个重要因素是'MqttBorkerPublisher'发布MQTT Qos 为'1'的消息,这意味着*至少一次交付*。

有关MQTT容错的更多信息，请参阅下的的**交付保证**部分。


## 交付保证
在Storm中，***MQTT Spout(喷口)至少提供一次传递***，具体取决于发布者的配置以及MQTT的spout(喷口)。

MQTT协议定义了以下QoS级别：

* `0` - 最多一次 (AKA "Fire and Forget")
* `1` - 起码一次
* `2` - 完全一次

这里可能有点混乱，因为MQTT协议规范并没有真正解决一个节点补一个灾难性事件完全焚烧的事实。这与Storm的可靠性形成了一个鲜明的对比，该模型期望并拥抱节点的概念。 

所以弹性最终取决于基础的MQTT实现和基础架构。

###推荐

*你将永远不会得到一次处理这个spout(喷口)。它可以与三叉线一起使用，但不会提供事物语义。

如果您需要可靠性的保证(即 *至少一次处理*):

1. 对于MQTT 发布者(Storm之外),发布QoS为'1'的消息，以便在spout脱机时，代理保存消息。
2. 使用spout的默认值 (`cleanSession = false` and `qos = 1`)
3. 如果可以，请确保接收和MQTT消息和任何结果是幂等的。
4. 确保您的MQTT代理不会因为网络分区而死亡或孤立。为自然灾害和人为灾害及网络分区做好准备。以及焚化和破坏的发生。





## 配置
有关完整的配置选项，请参阅JavaDoc for
 `org.apache.storm.mqtt.common.MqttOptions`.

### 信息映射器
要定义MQTT消息如何映射到Storm元组，您可以使用该实现配置MQTT spout `org.apache.storm.mqtt.MqttMessageMapper` 接口, 如下所示:

```java
public interface MqttMessageMapper extends Serializable {

    Values toValues(MqttMessage message);

    Fields outputFields();
}
```

 `MqttMessage` 类包含消息发布的主题("String")和消息的有效负载(`byte[]`). 例如，这是一个 `MqttMessageMapper` 实现，它基于这两者的内容生成元组消息主题和有效负载：

```java
/**
 * Given a topic name: "users/{user}/{location}/{deviceId}"
 * and a payload of "{temperature}/{humidity}"
 * emits a tuple containing user(String), deviceId(String), location(String), temperature(float), humidity(float)
 *
 */
public class CustomMessageMapper implements MqttMessageMapper {
    private static final Logger LOG = LoggerFactory.getLogger(CustomMessageMapper.class);


    public Values toValues(MqttMessage message) {
        String topic = message.getTopic();
        String[] topicElements = topic.split("/");
        String[] payloadElements = new String(message.getMessage()).split("/");

        return new Values(topicElements[2], topicElements[4], topicElements[3], Float.parseFloat(payloadElements[0]), 
                Float.parseFloat(payloadElements[1]));
    }

    public Fields outputFields() {
        return new Fields("user", "deviceId", "location", "temperature", "humidity");
    }
}
```

### 元组映射器
使用MQTT bolt(螺栓)或Trident功能发布MQTT消息时，您需要将元组数据映射到MQTT消息(主题/负载)。这是通过实现
`org.apache.storm.mqtt.MqttTupleMapper` 接口完成的:

```java
public interface MqttTupleMapper extends Serializable{

    MqttMessage toMessage(ITuple tuple);

}
```
例如，一个简单的 `MqttTupleMapper`  实现可能如下所示:

```java
public class MyTupleMapper implements MqttTupleMapper {
    public MqttMessage toMessage(ITuple tuple) {
        String topic = "users/" + tuple.getStringByField("userId") + "/" + tuple.getStringByField("device");
        byte[] payload = tuple.getStringByField("message").getBytes();
        return new MqttMessage(topic, payload);
    }
}
```

### MQTT Spout(喷口)平行度
建议您对MQTT spout(喷口)使用并行度1，否则最终会出现多个实例的端口订阅机同的主题，导致重复消费。

如果您要并行化spout(喷口)，建议您在拓扑中使用多个spout(喷口)实例，并使用MQTT主题选择器对数据进行分组。
如何实现分区的策略的方法最终由您的MQTT主题结构决定。举个例子，如果你按区域划分主题(如东/西)，你可以做类似于以下内容：

```java
String spout1Topic = "users/east/#";
String spout2Topic = "users/west/#";
```

然后通过预订一个(bolt)螺栓将每个流加入到结果流中。


### 使用 Flux

以上Flux YAML 配置创建了示例中使用的拓扑：

```yaml
name: "mqtt-topology"

components:
   ########## MQTT Spout Config ############
  - id: "mqtt-type"
    className: "org.apache.storm.mqtt.examples.CustomMessageMapper"

  - id: "mqtt-options"
    className: "org.apache.storm.mqtt.common.MqttOptions"
    properties:
      - name: "url"
        value: "tcp://localhost:1883"
      - name: "topics"
        value:
          - "/users/tgoetz/#"

# topology configuration
config:
  topology.workers: 1
  topology.max.spout.pending: 1000

# spout definitions
spouts:
  - id: "mqtt-spout"
    className: "org.apache.storm.mqtt.spout.MqttSpout"
    constructorArgs:
      - ref: "mqtt-type"
      - ref: "mqtt-options"
    parallelism: 1

# bolt definitions
bolts:
  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1


streams:
  - from: "mqtt-spout"
    to: "log"
    grouping:
      type: SHUFFLE

```


### 使用 Java

同样，你可以使用Storm Core Java API 来创建相同的拓扑结构：

```java
TopologyBuilder builder = new TopologyBuilder();
MqttOptions options = new MqttOptions();
options.setTopics(Arrays.asList("/users/tgoetz/#"));
options.setCleanConnection(false);
MqttSpout spout = new MqttSpout(new StringMessageMapper(), options);

MqttBolt bolt = new LogInfoBolt();

builder.setSpout("mqtt-spout", spout);
builder.setBolt("log-bolt", bolt).shuffleGrouping("mqtt-spout");

return builder.createTopology();
```

## SSL/TLS
如果要连接的MQTT代理需要SSL或SSL客户端身份验证，则需要配置具有适当URI的端口以及包含必要证书的秘钥库/信任库文件的位置。

### SSL/TLS URIs
要通过SSL/TLS连接使用前缀为 `ssl://` 或 `tls://` 而不是 `tcp://`. 进一步控制该算法可以指定一个特定的协议:

 * `ssl://` 使用JVM默认版本的SSL 协议。
 * `sslv*://` 使用特定版本的SSL协议，其中 `*` 替换为版本 (e.g. `sslv3://`)
 * `tls://` 使用JVM默认版本的TLS 协议。
 * `tlsv*://` 使用特定版本的TLS 协议，其中 `*` 替换为版本 (e.g. `tlsv1.1://`)
 
 
### 指定 密钥库/信任域的位置
 
`MqttSpout`, `MqttBolt` and `MqttPublishFunction` 都有构造函数，它们使用一个 `KeyStoreLoader` 实例来加载 TLS/SSL连接所需的证书。例如：
 
```java
 public MqttSpout(MqttMessageMapper type, MqttOptions options, KeyStoreLoader keyStoreLoader)
```
 
`DefaultKeyStoreLoader` 类可用于从本地文件系统加载证书。 请注意，密钥库/信任库需要在可能执行spout(喷口)/bolt(螺栓)的所有工作节点上可用。要使用 `DefaultKeyStoreLoader` 指定密钥库/信任库文件的位置，并设置必要的密码:

```java
DefaultKeyStoreLoader ksl = new DefaultKeyStoreLoader("/path/to/keystore.jks", "/path/to/truststore.jks");
ksl.setKeyStorePassword("password");
ksl.setTrustStorePassword("password");
//...
```

如果您的密钥库/信任库证书存储在单个文件中，则可以使用单参数构架函数：


```java
DefaultKeyStoreLoader ksl = new DefaultKeyStoreLoader("/path/to/keystore.jks");
ksl.setKeyStorePassword("password");
//...
```

还可以使用Flux来配置SSL/TLS: 

```yaml
name: "mqtt-topology"

components:
   ########## MQTT Spout Config ############
  - id: "mqtt-type"
    className: "org.apache.storm.mqtt.examples.CustomMessageMapper"

  - id: "keystore-loader"
    className: "org.apache.storm.mqtt.ssl.DefaultKeyStoreLoader"
    constructorArgs:
      - "keystore.jks"
      - "truststore.jks"
    properties:
      - name: "keyPassword"
        value: "password"
      - name: "keyStorePassword"
        value: "password"
      - name: "trustStorePassword"
        value: "password"

  - id: "mqtt-options"
    className: "org.apache.storm.mqtt.common.MqttOptions"
    properties:
      - name: "url"
        value: "ssl://raspberrypi.local:8883"
      - name: "topics"
        value:
          - "/users/tgoetz/#"

# topology configuration
config:
  topology.workers: 1
  topology.max.spout.pending: 1000

# spout definitions
spouts:
  - id: "mqtt-spout"
    className: "org.apache.storm.mqtt.spout.MqttSpout"
    constructorArgs:
      - ref: "mqtt-type"
      - ref: "mqtt-options"
      - ref: "keystore-loader"
    parallelism: 1

# bolt definitions
bolts:

  - id: "log"
    className: "org.apache.storm.flux.wrappers.bolts.LogInfoBolt"
    parallelism: 1


streams:

  - from: "mqtt-spout"
    to: "log"
    grouping:
      type: SHUFFLE

```

## Committer Sponsors

 * P. Taylor Goetz ([ptgoetz@apache.org](mailto:ptgoetz@apache.org))