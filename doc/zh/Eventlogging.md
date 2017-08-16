---
title: 拓扑事件检查器
layout: documentation
documentation: true
---

# 简介

拓扑事件检查器提供了在storm拓扑的不同阶段时查看元组的功能。
这可以用于在拓扑运行时检查在拓扑管线中的a spout(喷口)或a bolt(螺栓)处发射的元组，而不用停止或重新部署拓扑。从the spouts(喷口)到the bolts(螺栓)元组的正常流动是不受找开事件记录的影响。

## 启用事件日志记录

注意：首先事件日志记录需要将storm的"topolopy.eventlogger.executors"参数设置成非零的值。
详情请查询 [Configuration](#config) 章节内容。

可以通过在拓扑视图中的拓扑操作下单击“调试”按钮来记录事件。这会记录来自所有spouts（喷口）和bolts(螺栓)的元组以指定的采样百分比在拓扑中。

<div align="center">
<img title="Enable Eventlogging" src="images/enable-event-logging-topology.png" style="max-width: 80rem"/>

<p>Figure 1: Enable event logging at topology level.</p>
</div>

您还可以通过转到相应的组件页面来启用特定(spout)喷口或(bolt)螺栓级别的事件记录和
单击组件操作下的“调试”。

<div align="center">
<img title="Enable Eventlogging at component level" src="images/enable-event-logging-spout.png" style="max-width: 80rem"/>

<p>Figure 2: Enable event logging at component level.</p>
</div>

## 查看事件日志
Storm "logviewer" 应该运行查看已记录的元组。如果没有运行，则可以从Storm安装目录运行“bin/storm logviewer” 命令启动日志查看器。要查看元组，请从Storm UI中访问特定的spout(喷口)或bolt(螺栓)组件页面，然后单击组件摘要下的“事件”链接(如上图2所示)。

这将打开一个如下所示的视图，您可以在不同的页面之间导航并查看自己已记录的元组。

<div align="center">
<img title="Viewing logged tuples" src="images/event-logs-view.png" style="max-width: 80rem"/>

<p>Figure 3: Viewing the logged events.</p>
</div>

事件日志中的每一行都包含一个与从特定spout(喷口)/bolt(螺栓)(已逗号分隔的格式)发出的元组相对应的条目。

`Timestamp, Component name, Component task-id, MessageId (in case of anchoring), List of emitted values`

## 禁用事件日志

可以通过在Storm UI中的拓扑或组件操作下单击“停止调试”，在特定组件或拓扑级别上禁用事件日志。

<div align="center">
<img title="Disable Eventlogging at topology level" src="images/disable-event-logging-topology.png" style="max-width: 80rem"/>

<p>Figure 4: Disable event logging at topology level.</p>
</div>

## <a name="config"></a>Configuration
事件记录通过将事件(元组)从每个组件发送到内部事件日志记录工具。默认情况下，Storm不会启动任何事件记录器任务，但可以通过在运行拓扑时设置以下参数(通过在storm.yaml中设置或通过命令传递选项)轻松更改事件记录器任务。

| Parameter  | Meaning |
| -------------------------------------------|-----------------------|
| "topology.eventlogger.executors": 0      | No event logger tasks are created (default). |
| "topology.eventlogger.executors": 1      | One event logger task for the topology. |
| "topology.eventlogger.executors": nil      | One event logger task per worker. |


## 扩展事件日志
Strom提供了一个“IEventLogger”接口，由事件记录器螺栓用于记录事件。这个默认的实现是FileBasedEventLogger,它将事件记录到一个事件中。日志文件（`logs/workers-artifacts/<topology-id>/<worker-port>/events.log`）。可以添加“IEventLogger”接口的替代实现来扩展事件记录功能（例如，构建搜索索引或将事件记录到数据库中）
```java
/**
 * EventLogger interface for logging the event info to a sink like log file or db
 * for inspecting the events via UI for debugging.
 */
public interface IEventLogger {
    /**
    * Invoked during eventlogger bolt prepare.
    */
    void prepare(Map stormConf, TopologyContext context);

    /**
     * Invoked when the {@link EventLoggerBolt} receives a tuple from the spouts or bolts that has event logging enabled.
     *
     * @param e the event
     */
    void log(EventInfo e);

    /**
    * Invoked when the event logger bolt is cleaned up
    */
    void close();
}
```
