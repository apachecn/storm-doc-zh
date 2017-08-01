---
title: Storm 状态管理
layout: documentation
documentation: true
---
# 核心 Storm 中的状态支持
Storm 核心为 Bolt 提供用于保存和重新获取其操作状态的抽象. 提供一个基于内存的默认状态实现，同时还提供了一个使用 Redis 做状态保持的实现.

## 状态管理
若 Bolt 需要通过框架来管理和保持其状态, 应该实现接口 `IStatefulBolt`,或者继承类 `BaseStatefulBolt`,然后实现方法 `void initState(T state)`. 方法 `initState` 在 Bolt 使用保存的历史状态进行初始化期间通过框架执行. 执行时机在 `prepare` 方法之后，在 Bolt 开始处理 Tuple 数据之前.

当前支持的唯一一种 `State` 实现是提供 key-value 映射的 `KeyValueState`.

例如, 一个单词计数 bolt 可以使用 key-value 状态抽象实现单词计数, 步骤如下.

1. 继承 `BaseStatefulBolt` 类, 添加一个 `KeyValueState` 实例变量, 用于存储单词到单词数量的映射.
2. 在 init 方法中用之前保存的状态来初始化 Bolt. 这里面含有上次程序运行的时候框架最后一次提交的单词计数.
3. 在 `execute` 方法中, 更新单词计数.

 ```java
 public class WordCountBolt extends BaseStatefulBolt<KeyValueState<String, Long>> {
 private KeyValueState<String, Long> wordCounts;
 private OutputCollector collector;
 ...
     @Override
     public void prepare(Map stormConf, TopologyContext context, OutputCollector collector) {
       this.collector = collector;
     }
     @Override
     public void initState(KeyValueState<String, Long> state) {
       wordCounts = state;
     }
     @Override
     public void execute(Tuple tuple) {
       String word = tuple.getString(0);
       Integer count = wordCounts.get(word, 0);
       count++;
       wordCounts.put(word, count);
       collector.emit(tuple, new Values(word, count));
       collector.ack(tuple);
     }
 ...
 }
 ```
4. 框架周期性的检查并保存 Bolt 的状态 (默认每秒一次). 频率可以通过设置 storm config 的 `topology.state.checkpoint.interval.ms`来自己定义。
5. 对于状态持久化, 可以设置 storm config 中的 `topology.state.provider` 来使用支持持久化的 state provider. 例如, 若使用基于 Redis 的 key-value 状态实现, 需要在 storm.yaml 文件中设置 `topology.state.provider: org.apache.storm.redis.state.RedisKeyValueStateProvider`. provider 实现代码的 jar 包需要放在 class path 下, 在这个例子中, 需要把 `storm-redis-*.jar` 置于 extlib 目录下.
6. state provider 的属性可以通过设置 `topology.state.provider.config` 来进行覆盖. 对于 Redis state, 是一个具有下列属性的 JSON 字符串.

 ```
 {
   "keyClass": "Optional fully qualified class name of the Key type.",
   "valueClass": "Optional fully qualified class name of the Value type.",
   "keySerializerClass": "Optional Key serializer implementation class.",
   "valueSerializerClass": "Optional Value Serializer implementation class.",
   "jedisPoolConfig": {
     "host": "localhost",
     "port": 6379,
     "timeout": 2000,
     "database": 0,
     "password": "xyz"
     }
 }
 ```

## 检查点机制
检查点通过一个内部的 checkpoint spout 来触发，触发周期在 `topology.state.checkpoint.interval.ms` 指定. 如果在拓扑中至少有一个 `IStatefulBolt`, topology builder 会自动添加 checkpoint spout. 对于有状态的拓扑, topology builder 使用 `StatefulBoltExecutor` 包装 `IStatefulBolt`, 负责在收到 checkpoint tuple 的时候来执行状态提交. 无状态的 Bolt 被包装在 `CheckpointTupleForwarder`, 仅会转发 checkpoint tuple 以确保其可以贯穿整个拓扑DAG(有向无环图). checkpoint tuple 在一个名为 `$checkpoint` 的内部 stream 中流动. topology builder 组织 checkpoint spout 源流出的 checkpoint stream 穿过整个拓扑.

```
              default                         default               default
[spout1]   ---------------> [statefulbolt1] ----------> [bolt1] --------------> [statefulbolt2]
                          |                 ---------->         -------------->
                          |                   ($chpt)               ($chpt)
                          |
[$checkpointspout] _______| ($chpt)
```

当到了检查周期, checkpoint tuples 被 checkpoint spout 发射出来. 一旦接收到 state tuple, Bolt 的状态就会被保存, 然后 checkpoint tuple 会转发到下一个组件. 每一个 Bolt 在保存状态之前, 会在所有的输入流上等待 checkpoint 到达, 使得状态表现为一个跨整个拓扑的持续的状态. 一旦 checkpoint spout 从所有的 Bolt 中接收到ACK消息, 状态提交就完成了, 事务会被 checkpoint spout 记录为已提交.

checkpoint 当前不会检查 Spout 的状态. 目前, 一旦所有的 Bolt 被检查完毕, 并且一旦 checkpoint tuple 被 ack, Spout 发射的 tuples 也会被 ack.
这也意味着, `topology.state.checkpoint.interval.ms` 要小于 `topology.message.timeout.secs`. 

状态提交的工作方式就像一个具有 `准备` 和 `提交` 阶段的三段式提交协议, 以达到跨整个拓扑的状态的保存操作具有一致性和原子性.

### 恢复
恢复阶段会在拓扑首次启动的时候触发. 如果前置事务没有成功装备好, 会向拓扑中发送一个 `rollback` 消息, Bolt 会丢弃已经就绪的事务. 如果前置事务成功准备好但是未提交, 会向拓扑中发送一个 `commit` 消息让所有已经就绪的事务可以被提交. 当这些步骤完成后, Bolt 状态初始化完成.

恢复也会在其中一个 Bolt 未成功确认 checkpoint 消息或者 worker 在这中间挂了的时候触发. 因此, 当 supervisor 重启一个 worker, checkpoint 机制会确保 Bolt 使用之前的状态初始化, 同时检查操作会从上次离开的点继续执行.

### 可靠性
Storm 使用 acking 机制在 tuples 处理失败的时候进行重新发送. 有可能状态已经提交但是 worker 在确认(ack) tuple 之前挂掉. 在这种情况下重新发送的 tuple 会导致状态重复更新. 当前, `StatefulBoltExecutor ` 在接收到一个流中的 checkpoint tuple 以后继续从一个流中获取并处理 tuple, 同时等待 checkpoint 到达其他输入流以保存状态. 这也可能导致恢复期间造成重复的状态更新.

状态抽象并不能消除重复, 当前仅提供'至少一次'的保障.

为了提供'至少一次'的保障, 有状态拓扑中的所有 Bolt 都会对 Tuple 进行标记, 同时在处理完成后发射并确认输入 Tuple. 对于无状态的 Bolt, 继承 `BaseBasicBolt` 可以自动管理"标记/确认"操作. 有状态的 Bolt 标记 Tuple同时在处理完成后发射和确认tuple, 就像上面"状态管理"一节中的 `WordCountBolt`.

### IStateful bolt 钩子
IStateful 接口提供钩子方法用以在有状态 Bolt 中可以实现一些自定义的动作
```java
    /**
     * This is a hook for the component to perform some actions just before the
     * framework commits its state.
     */
    void preCommit(long txid);

    /**
     * This is a hook for the component to perform some actions just before the
     * framework prepares its state.
     */
    void prePrepare(long txid);

    /**
     * This is a hook for the component to perform some actions just before the
     * framework rolls back the prepared state.
     */
    void preRollback();
```
这个功能是可选的, 并且有状态 Bolt 未提供任何实现. 提供这个功能是为了可以在状态抽象的顶层(我们可能想在有状态 Bolt 的状态准备好之前做一些其他动作如提交或者回滚的地方)建立其他系统级组件.

## 提供自定义状态实现
当前唯一支持的 `State` 实现是提供 key-value 的映射的 `KeyValueState`.

自定义状态实现应当为接口 `org.apache.storm.State` 的方法提供实现. 这些方法是`void prepareCommit(long txid)`, `void commit(long txid)`, `rollback()`.  `commit()` 方法是可选的且在 Bolt 管理自己的状态的时候非常有用. 这些当前仅用于内部系统 Bolt, 例如 CheckpointSpout 在保存自己状态的时候.

`KeyValueState` 的实现也应当实现定义在接口 `org.apache.storm.state.KeyValueState` 中的方法.

### State provider
框架通过对应的 `StateProvider` 来实例化状态. 一个自定义的状态应当也提供一个可以加载和返回基于命名空间的状态的 `StateProvider` 实现. 每一个状态属于一个独有的命名空间. 命名空间通常是每个 Task 唯一的, 因此每个任务可以有自己的状态. StateProvider 和相应的 State 实现应该位于 Storm 的 class path 下（一般放在 extlib 目录中).
