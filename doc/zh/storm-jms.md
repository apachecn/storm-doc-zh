---
title: Storm JMS 集成
layout: documentation
documentation: true
---

## 关于 Storm JMS
Storm JMS是在Storm框架内集成JMS消息传递的通过框架。

Storm-JMS 允许您通过JMS spout(喷口)将数据注入到Storm，并通过通用JMS bolt(螺栓)从Storm 消费数据。

JMS Spout(喷口)和Bolt(螺栓)都是数据不可知的。要使用它们，您需要提供一个简单的Java类，用于桥接JMS和Storm
API 以及封装和特定域的逻辑。


## 组件

### JMS Spout(喷口)
JMS Spout(喷口)组件允许将发布到JMS主题或队列的数据由Storm拓扑消费。
JMS Spout(喷口)连接到JMS目标(主题或队列)，并根据收到的JMS消息的内容发送给Storm "Tuple"对象。


### JMS Bolt(螺栓)
JMS Bolt(螺栓)组件允许将Storm 拓扑中的数据发布到JMS目标（主题或队列）。

JMS Bolt(螺栓)连接到JMS目标，并根据接收的Storm "Tuple"对象发布JMS消息。

[Example Topology](storm-jms-example.html)


[Using Spring JMS](storm-jms-spring.html)

