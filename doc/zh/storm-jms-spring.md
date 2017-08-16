---
title: Storm JMS 集成
layout: documentation
documentation: true
---

### 使用 Spring 的 JMS 支持连接到 JMS

穿件一个 Spring 的 applicationContext.xml 文件, 它定义了一个或多个 destination (topic/queue) beans, 以及一个 connecton factory.

	<?xml version="1.0" encoding="UTF-8"?>
	<beans 
	  xmlns="http://www.springframework.org/schema/beans" 
	  xmlns:amq="http://activemq.apache.org/schema/core"
	  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
	  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">
	
		<amq:queue id="notificationQueue" physicalName="backtype.storm.contrib.example.queue" />
		
		<amq:topic id="notificationTopic" physicalName="backtype.storm.contrib.example.topic" />
	
		<amq:connectionFactory id="jmsConnectionFactory"
			brokerURL="tcp://localhost:61616" />
		
	</beans>