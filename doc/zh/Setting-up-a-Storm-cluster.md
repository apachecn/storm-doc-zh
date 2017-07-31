---
title: 设置Storm集群
layout: documentation
documentation: true

---
本页概述了启动和运行Storm集群的步骤,如果您使用AWS,您应该查看[storm-部署](https://github.com/nathanmarz/storm-deploy/wiki)项目.[storm-部署](https://github.com/nathanmarz/storm-deploy/wiki) 在EC2上完全自动化配置和安装Storm集群. 它还为您设置Ganglia,以便您可以监视CPU,磁盘和网络使用情况.

如果您在运行Strom集群时遇到困难,请首先在 [Troubleshooting](Troubleshooting.html) 页寻求解决. 再者, 查看或者发送邮件列表.

下面是部署Storm集群的步骤总结:

1. 设置 Zookeeper 集群.
2. 设置Nimbus和worker 节点的安装环境
3. 在集群节点下载解压 Storm
4. 在storm.yaml中设置必要的配置
5. 在指导下使用 storm 脚本来启动相应的守护进程.（备注:master包括nimbus和UI进程,slave包括supervisor和logviewer）

### 设置Zookeeper集群

Storm 使用 Zookeeper 协调管理集群. Zookeeper **并不是** 用于消息传递, 所以 Storm 对Zookeeper造成的负载压力非常低. 单节点Zookeeper集群在大多数情况下应该是足够的,但是如果您想要故障转移或部署大型Storm集群,则可能需要较大的Zookeeper集群. 部署Zookeeper的说明是[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html). 

关于Zookeeper部署的几点注意事项:

1.在监督下运行Zookeeper至关重要,因为Zookeeper是故障快速的,如果遇到任何错误的情况都将退出进程. 有关详细信息,请参阅[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_supervision) . 

2.建立一个cron定时任务来压缩Zookeeper的数据和事务日志至关重要. Zookeeper守护进程本身不会这样做,如果没有设置cron,Zookeeper将很快耗尽磁盘空间. 有关详细信息,请参阅[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperAdmin.html#sc_maintenance).

### Nimbus 和 worker 节点的安装环境

接下来你需要准备Nimbus 和 worker 节点的安装环境:

1. Java 7 (备注:建议使用Java 8,毕竟很多的项目开发环境已经迁移到8以上)
2. Python 2.6.6

这些依赖版本是Storm已经测试过的. Storm 在不同的Java 或Python版本上也许会存在问题.


### 下载解压 Storm

接下来,下载一个Storm版本,并解压zip文件到Nimbus和每个worker机器上的某个目录下. Storm版本可以从这里[下载](http://github.com/apache/storm/releases).

### 在storm.yaml中设置必要的配置

Storm 发布包中在目录`conf/storm.yaml` 下包含一个默认的配置文件. 你可以在[这里]({{page.git-blob-base}}/conf/defaults.yaml)查看默认值. storm.yaml 中的存在的配置项会覆盖掉 defaults.yaml中相应的配置项. 下面一些配置是集群运行时所必要的:

1) **storm.zookeeper.servers**: 这是一个Storm集群所依赖 Zookeeper 集群的hosts列表. 类似于:

```yaml
storm.zookeeper.servers:
  - "111.222.333.444"
  - "555.666.777.888"
```

如果配置的Zookeeper集群不是默认的端口, 你应该设置 **storm.zookeeper.port** 选项.

2) **storm.local.dir**: Nimbus 和 Supervisor 守护进程需要配置一个本地目录来存储少量状态信息(例如jars包,配置文件等等).
 您应该在每个机器上创建该目录,给予适当的权限,然后使用此配置填写目录位置. 例如:

```yaml
storm.local.dir: "/mnt/storm"
```
如果您在windows下运行Strom,应该如下:
```yaml
storm.local.dir: "C:\\storm-local"
```
如果您使用相对路径,那么路径是相对于(STORM_HOME).
您也可以使用默认值 `$STORM_HOME/storm-local`

3) **nimbus.seeds**: worker节点需要知道哪些机器是主机的候选者,以便下载 topology jar和confs(nimbus.host 在1.0之后已经废弃,这里实现了HA). 例如:

```yaml
nimbus.seeds: ["111.222.333.44"]
```
鼓励您填写**机器的FQDN **(Fully Qualified Domain Name,全域名)列表. 如果要设置Nimbus HA,则必须解决运行nimbus的所有机器的FQDN.当您只想设置“伪分布式”集群时您可能希望将其保留为默认值,仍然鼓励您填写FQDN.

4) **supervisor.slots.ports**: 对于每个worker节点,您可以使用此配置设置在该计算机上运行的worker数量. 每个worker使用单个端口接收消息,并且此设置定义哪些端口打开以供使用. 如果您在此定义五个端口,那么Storm将分配最多五个worker在本机上运行. 如果您定义了三个端口,Storm将只能运行三个worker. 默认情况下,此设置被配置为在端口6700,6701,6702和6703上运行4个worker:

```yaml
supervisor.slots.ports:
    - 6700
    - 6701
    - 6702
    - 6703
```

### 监控 Supervisors 健康状态

Storm提供了一种机制,管理员可以通过该机制配置supervisor进程定期运行管理员提供的脚本,以确定节点是否健康. 管理员可以通过执行位于storm.health.check.dir中的脚本来确定supervisor节点是否处于健康状态. 如果脚本检测到节点处于不正常状态,则必须标准流输出ERROR开头的信息. supervisor将定期运行健康检查目录中的脚本并检查输出. 如果脚本的输出包含字符串ERROR,如上所述,superviosr进程将关闭所有worker并退出. 

如果supervisor 已经有监控脚本,通过执行 "/bin/storm node-health-check"可以确定 supervisor 是否可以启动 或 supervisor 节点是否健康

健康检查目录的地址通过以下配置项设置:

```yaml
storm.health.check.dir: "healthchecks"

```
脚本必须有执行权限.
健康检查脚本可以设置超时失败,在时间段内脚本未执行完毕则被标记为失败,脚本运行超时时间设置:

```yaml
storm.health.check.timeout.ms: 5000
```

### 配置外部libs和环境变量 (可选)


490/5000
如果需要外部库或自定义插件的支持,可以将这些jar放在extlib/ 和 extlib-daemon/ 目录中. 请注意,extlib-daemon/ 目录存储仅由守护进程（Nimbus,Supervisor,DRPC,UI,Logviewer）所调用加载的jar,例如HDFS和自定义调度库. 因此,用户可以配置两个环境变量STORM_EXT_CLASSPATH和STORM_EXT_CLASSPATH_DAEMON,以便包含外部classpath和仅限守护进程调用的外部classpath.


### 在指导下使用 "storm" 脚本启动相应的守护进程

最后一步是启动所有的Storm守护进程. 在指导下运行这些守护进程是至关重要的. Storm是一个__fail-fast__系统,这意味着每当遇到意外错误时,进程将停止. Storm的设计使得它可以在任何时候安全地停止,并在进程重新启动时正确恢复. 这就是为什么Storm不会在进程中保持状态 - 如果Nimbus或者Supervisors重新启动,运行的topologies不会受到影响. 以下是运行Storm守护进程的方法:

1. **Nimbus**: 在主节点(Nimbus)运行命令行 "bin/storm nimbus".
2. **Supervisor**: 在每个supervisor节点运行命令行 "bin/storm supervisor". supervisor守护程序负责启动和停止该机器上的worker进程.
3. **UI**:通运行命令“bin/storm ui”,运行Storm UI（一般选择在Nimbus节点启动一个,可以从浏览器访问群集和topologies的诊断信息）.可以通过浏览网页浏览器访问http://{ui host}:8080.

正如你所看到的,运行守护进程非常简单.守护进程将日志输出到您解压缩Storm时的 logs/ 目录.
