---
title: Pacemaker
layout: documentation
documentation: true
---


### 介绍

Pacemaker 是一个旨在处理 worker 心跳的 storm 守护进程.
随着 storm 的扩大, ZooKeeper 由于 worker 进行心跳的大量写入而开始成为瓶颈.
当 ZooKeeper 尝试维护一致性时, 会产生大量写入磁盘和跨网络的流量.

因为心跳是短暂的, 它们不需要被持久化到磁盘或跨节点同步; 会在内存的存储中来做.
这是 Pacemaker 的作用.
Pacemaker 作为一个简单的 key/value 存储, 具有类似 ZooKeeper 的目录样式键和字节数组值.

相应的 Pacemaker client 是 `ClusterState` 接口的插件 `org.apache.storm.pacemaker.pacemaker_state_factory`.
心跳调用由漏斗 `ClusterState` 由生产 `pacemaker_state_factory` 到 Pacemaker 服务进程, 而另一 set/get 操作被转发到 ZooKeeper.

------

### 配置

 - `pacemaker.host` : （已弃用） Pacemaker 守护程序正在运行的主机
 - `pacemaker.servers` : Pacemaker 守护程序正在运行的主机 - 这取代了 `pacemaker.host`
 - `pacemaker.port` : Pacemaker 监听端口
 - `pacemaker.max.threads` : Pacemaker 守护进程将用于处理请求的最大线程数.
 - `pacemaker.childopts` : 任何需要转到 Pacemaker 的 JVM 参数.（由 storm-deploy 项目使用）
 - `pacemaker.auth.method` : 使用的身份验证方法（以下更多信息）

#### 示例

要使 Pacemaker 启动并运行, 请在所有节点上的集群配置中设置以下选项：

```
storm.cluster.state.store: "org.apache.storm.pacemaker.pacemaker_state_factory"
```

Pacemaker 服务器还需要在所有节点上设置：

```
pacemaker.servers:
    - somehost.mycompany.com
    - someotherhost.mycompany.com
```

pacemaker.host 配置仍适用于单个 pacemaker, 尽管它已被弃用.

```
pacemaker.host: single_pacemaker.mycompany.com
```

然后启动所有的守护进程(包括 Pacemaker):

```
$ storm pacemaker
```

Storm 集群现在应该通过 Pacemaker 来推动所有 worker 的心跳.

### 安全

目前支持摘要（基于密码）和 Kerberos 安全性.
安全性目前只在于读取而不是写入.
写入可以由任何人执行, 而读取只能由授权和认证的用户执行.
这是未来发展的一个领域, 因为它让群集开放给 DoS 攻击, 但它阻止任何敏感信息到达未经授权的眼睛, 这是主要目标.

#### Digest

要配置摘要身份验证, 请 `pacemaker.auth.method: DIGEST` 在集群配置中设置托管 Nimbus 和 Pacemaker 的节点.
节点也必须 `java.security.auth.login.config` 设置为指向包含以下结构的 JAAS 配置文件：

```
PacemakerDigest {
    username="some username"
    password="some password";
};
```

配置了这些设置的任何节点将能够从 Pacemaker 中读取.
Worker 节点不需要设置这些配置, 并且可以保留 `pacemaker.auth.method: NONE` 设置, 因为它们不需要从 Pacemaker 守护进程读取.

#### Kerberos

要配置Kerberos身份验证, 请 `pacemaker.auth.method: KERBEROS` 在主机 Nimbus 和 Pacemaker 的节点上的集群配置中进行设置.
节点也必须 `java.security.auth.login.config` 设置为指向 JAAS 配置.

Nimbus 上的 JAAS 配置必须看起来像这样：

```
PacemakerClient {
    com.sun.security.auth.module.Krb5LoginModule required
    useKeyTab=true
    keyTab="/etc/keytabs/nimbus.keytab"
    storeKey=true
    useTicketCache=false
    serviceName="pacemaker"
    principal="nimbus@MY.COMPANY.COM";
};
                         
```

Pacemaker 上的 JAAS 配置必须如下所示：

```
PacemakerServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/etc/keytabs/pacemaker.keytab"
   storeKey=true
   useTicketCache=false
   principal="pacemaker@MY.COMPANY.COM";
};
```

- Nimbus 主机上 `PacemakerClient` 部分中的客户端用户主体必须与 Storm集群上的 `nimbus.daemon.user` 配置值相匹配.
- 在, 客户端的 `serviceName` 值必须与 Pacemaker主机上 `PacemakerServer` 部分的服务器的用户主体相匹配.

### 容错

Pacemaker 作为单个守护进程运行, 使其成为潜在的单点故障.

如果 Pacemaker 由 Nimbus 无法通过崩溃或网络分区, worker 将继续运行, Nimbus 将重复尝试重新连接.
Nimbus 的功能将受到干扰, 但 topology 本身将继续运行.
如果 Nimbus 和 Pacemaker 位于分区同一侧的集群分区, 分区另一侧的 worker 将无法心跳, Nimbus 将重新安排其他任务.
这可能是我们想要发生的事情.

### ZooKeeper 比较

与 ZooKeeper 相比, Pacemaker 使用更少的 CPU, 更少的内存, 当然也没有磁盘用于相同的负载, 这是由于缺乏维护节点之间的一致性的开销.
在千兆网络上, 有6000个节点的理论限制.然而, 实际限制可能在2000-3000节点之间.这些限制还没有被测试.
在拥有 topology 结构的270个管理员集群中, Pacemaker 的资源利用率是一个核心的70％, 在具有4 `Intel(R) Xeon(R) CPU E5530 @ 2.40GHz` 和24GiB RAM 的机器上的近1GiB 的RAM.

Pacemaker 在支持 HA.多个 Pacemaker 实例可以在 Storm 群集中一次使用, 以实现大规模的可扩展性.
只需将 Pacemaker 主机的名称包含在 pacemaker.servers 配置中, worker 和 Nimbus 将开始与他们进行通信.
他们也是容错的.只要至少有一个 Pacemaker 运行, 系统就会继续工作 - 只要它可以处理负载.
