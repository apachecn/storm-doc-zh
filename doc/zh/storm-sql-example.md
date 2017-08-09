---
title: Storm SQL 示例
layout: documentation
documentation: true
---

This page shows how to use Storm SQL by showing the example of processing Apache logs. 
This page is written by "how-to" style so you can follow the step and learn how to utilize Storm SQL step by step. 
本页通过处理 Apache 日志的例子来展示如何使用 strom SQL.
本页使用 "how-to" 风格书写, 因此可以根据步骤学习如何一步一步的学习使用 Storm SQL.

## 准备

This page assumes that Apache Zookeeper, Apache Storm and Apache Kafka is installed locally and running with properly configured.
For convenience, this page assumes that Apache Kafka 0.10.0 is installed via `brew`.
本页假定 Apache Zookeeper、Apache Storm、Apache Kafka 都本地安装并且正确配置运行.
方便起见, 本页假设 Apache Kafka 0.10.0 是通过 `brew` 安装.

We'll use below tools to prepare the JSON data which will be fed to the input data source. 
Since they're Python projects, this page assumes Python 2.7 with `pip`, `virtualenv` is installed locally. 
If you're using Python 3, you may need to convert some places to be compatible with 3 manually while feeding data. 
我们将使用下列工具为输入数据源生成 JSON 数据.
因为他们都是 Python 工程, 本页假设已经安装了 Python 2.7、`pip`、`virtualenv`.
如果你使用 Python 3, 需要在生成数据的时候修改一些与Python 3不兼容的代码.

* https://github.com/kiritbasu/Fake-Apache-Log-Generator
* https://github.com/rory/apache-log-parser

## 创建 topic

In this page, we will use four topics, `apache-logs`, `apache-errorlogs`, `apache-slowlogs`.
Please create topics according to your environment. 
在本页, 我们会使用4个 topic, `apache-logs`, `apache-errorlogs`, `apache-slowlogs`.
请根据你的环境来创建 topic.

For Apache Kafka 0.10.0 with brew installed,
对于使用 brew 安装的 Apache Kafka 0.10.0

```
kafka-topics --create --topic apache-logs --zookeeper localhost:2181 --replication-factor 1 --partitions 5
kafka-topics --create --topic apache-errorlogs --zookeeper localhost:2181 --replication-factor 1 --partitions 5
kafka-topics --create --topic apache-slowlogs --zookeeper localhost:2181 --replication-factor 1 --partitions 5
```

## 灌入数据

Let's feed the data to input topics. In this page we will generate fake Apache logs, and parse to JSON format, and feed JSON to Kafka topic. 
让我们提供数据给输入 topic. 在本页中我们将生成假的 Apache 日志, 并转换为 JSON 格式, 并把 JSON 灌入 Kafka topic.

Let's create your working directory, since we will clone the project and also setup virtualenv.
开始创建你的工作目录, 用于克隆项目并且设置 virtualenv.

In your working directory, `virtualenv env` to setup virtualenv to env directory, and activate.
在你的工作目录, `virtualenv env` 把 evn 目录设置为 virtualenv 目录, 然后激活虚拟环境.

```
$ virtualenv env
$ source env/bin/activate
```

Feel free to `deactivate` when you're done with example.
完成例子以后, 可以随意 `deactivate`, 退出 python 虚拟环境.
 
### 安装和修改 Fake-Apache-Log-Generator

`Fake-Apache-Log-Generator` is not presented to package, and also we need to modify the script.
`Fake-Apache-Log-Generator` 对包不可见, 我们还需要修改一下脚本.

```
$ git clone https://github.com/kiritbasu/Fake-Apache-Log-Generator.git
$ cd Fake-Apache-Log-Generator
```

Open `apache-fake-log-gen.py` and replace `while (flag):` statements to below:
打开 `apache-fake-log-gen.py`, 将 `while (flag):` 语句替换成下面的语句:

```
        elapsed_us = random.randint(1 * 1000,1000 * 1000) # 1 ms to 1 sec
        seconds=random.randint(30,300)
        increment = datetime.timedelta(seconds=seconds)
        otime += increment

        ip = faker.ipv4()
        dt = otime.strftime('%d/%b/%Y:%H:%M:%S')
        tz = datetime.datetime.now(pytz.timezone('US/Pacific')).strftime('%z')
        vrb = numpy.random.choice(verb,p=[0.6,0.1,0.1,0.2])

        uri = random.choice(resources)
        if uri.find("apps")>0:
                uri += `random.randint(1000,10000)`

        resp = numpy.random.choice(response,p=[0.9,0.04,0.02,0.04])
        byt = int(random.gauss(5000,50))
        referer = faker.uri()
        useragent = numpy.random.choice(ualist,p=[0.5,0.3,0.1,0.05,0.05] )()
        f.write('%s - - [%s %s] %s "%s %s HTTP/1.0" %s %s "%s" "%s"\n' % (ip,dt,tz,elapsed_us,vrb,uri,resp,byt,referer,useragent))

        log_lines = log_lines - 1
        flag = False if log_lines == 0 else True
```

to make sure fake elapsed_us is included to fake log.
要确保 elapsed_us 包含在假日志中.

For convenience, you can skip cloning project and download modified file from here: [apache-fake-log-gen.py (gist)](https://gist.github.com/HeartSaVioR/79fd4e461604fabecf535ffece47e6c2)
为了方便, 你可以跳过克隆项目这一步, 直接从这里下载修改过的文件: [apache-fake-log-gen.py (gist)](https://gist.github.com/HeartSaVioR/79fd4e461604fabecf535ffece47e6c2)

### 安装 apache-log-parser 并编写转换脚本

`apache-log-parser` 模块可以通过 `pip` 命令安装.

```
$ pip install apache-log-parser
```

Since apache-log-parser is a library, in order to parse fake log we need to write small python script.
Let's create file `parse-fake-log-gen-to-json-with-incrementing-id.py` with below content: 
因为 `apache-log-parser` 是一个 python 库, 为了转换日志我们需要写编写一个小脚本.
我们创建一个文件 `parse-fake-log-gen-to-json-with-incrementing-id.py` 包含以下内容:

```
import sys
import apache_log_parser
import json

auto_incr_id = 1
parser_format = '%a - - %t %D "%r" %s %b "%{Referer}i" "%{User-Agent}i"'
line_parser = apache_log_parser.make_parser(parser_format)
while True:
  # we'll use pipe
  line = sys.stdin.readline()
  if not line:
    break
  parsed_dict = line_parser(line)
  parsed_dict['id'] = auto_incr_id
  auto_incr_id += 1

  # works only python 2, but I don't care cause it's just a test module :)
  parsed_dict = {k.upper(): v for k, v in parsed_dict.iteritems() if not k.endswith('datetimeobj')}
  print json.dumps(parsed_dict)
```

### 将转换后的 JSON Apache Log 灌入 Kafka

OK! We're prepared to feed the data to Kafka topic. Let's use `kafka-console-producer` to feed parsed JSON.
好了! 我们已经准备好将数据写入 Kafka topic. 下面使用 `kafka-console-producer` 来灌入 JSON.

```
$ python apache-fake-log-gen.py -n 0 | python parse-fake-log-gen-to-json-with-incrementing-id.py | kafka-console-producer --broker-list localhost:9092 --topic apache-logs
```

and execute below to another terminal session to confirm data is being fed.
打开另一个终端执行下面的命令, 确认数据已经进入 topic.

```
$ kafka-console-consumer --zookeeper localhost:2181 --topic apache-logs
```

If you can see the json like below, it's done:
如果看到如下的 json, 就说明搞定了:

```
{"TIME_US": "757467", "REQUEST_FIRST_LINE": "GET /wp-content HTTP/1.0", "REQUEST_METHOD": "GET", "RESPONSE_BYTES_CLF": "4988", "TIME_RECEIVED_ISOFORMAT": "2021-06-30T22:02:53", "TIME_RECEIVED_TZ_ISOFORMAT": "2021-06-30T22:02:53-07:00", "REQUEST_HTTP_VER": "1.0", "REQUEST_HEADER_USER_AGENT__BROWSER__FAMILY": "Firefox", "REQUEST_HEADER_USER_AGENT__IS_MOBILE": false, "REQUEST_HEADER_USER_AGENT__BROWSER__VERSION_STRING": "3.6.13", "REQUEST_URL_FRAGMENT": "", "REQUEST_HEADER_USER_AGENT": "Mozilla/5.0 (X11; Linux x86_64; rv:1.9.7.20) Gecko/2010-10-13 13:52:34 Firefox/3.6.13", "REQUEST_URL_SCHEME": "", "REQUEST_URL_PATH": "/wp-content", "REQUEST_URL_QUERY_SIMPLE_DICT": {}, "TIME_RECEIVED_UTC_ISOFORMAT": "2021-07-01T05:02:53+00:00", "REQUEST_URL_QUERY_DICT": {}, "STATUS": "200", "REQUEST_URL_NETLOC": "", "REQUEST_URL_QUERY_LIST": [], "REQUEST_URL_QUERY": "", "REQUEST_URL_USERNAME": null, "REQUEST_HEADER_USER_AGENT__OS__VERSION_STRING": "", "REQUEST_URL_HOSTNAME": null, "REQUEST_HEADER_USER_AGENT__OS__FAMILY": "Linux", "REQUEST_URL": "/wp-content", "ID": 904128, "REQUEST_HEADER_REFERER": "http://white.com/terms/", "REQUEST_URL_PORT": null, "REQUEST_URL_PASSWORD": null, "TIME_RECEIVED": "[30/Jun/2021:22:02:53 -0700]", "REMOTE_IP": "88.203.90.62"}
```

## 例子: 过滤错误日志
 
In this example we'll filter error logs from entire logs and store them to another topics. `project` and `filter` features will be used.
在这个例子中, 我们将从所有的日志中过滤出错误日志并且存储到另一个 topic 中. 将会用到 `project` 和 `filter` 特性.

The content of script file is here:
脚本文件的内容如下:

```
CREATE EXTERNAL TABLE APACHE_LOGS (ID INT PRIMARY KEY, REMOTE_IP VARCHAR, REQUEST_URL VARCHAR, REQUEST_METHOD VARCHAR, STATUS VARCHAR, REQUEST_HEADER_USER_AGENT VARCHAR, TIME_RECEIVED_UTC_ISOFORMAT VARCHAR, TIME_US DOUBLE) LOCATION 'kafka://localhost:2181/brokers?topic=apache-logs'
CREATE EXTERNAL TABLE APACHE_ERROR_LOGS (ID INT PRIMARY KEY, REMOTE_IP VARCHAR, REQUEST_URL VARCHAR, REQUEST_METHOD VARCHAR, STATUS INT, REQUEST_HEADER_USER_AGENT VARCHAR, TIME_RECEIVED_UTC_ISOFORMAT VARCHAR, TIME_ELAPSED_MS INT) LOCATION 'kafka://localhost:2181/brokers?topic=apache-error-logs' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.storm.kafka.ByteBufferSerializer"}}'
INSERT INTO APACHE_ERROR_LOGS SELECT ID, REMOTE_IP, REQUEST_URL, REQUEST_METHOD, CAST(STATUS AS INT) AS STATUS_INT, REQUEST_HEADER_USER_AGENT, TIME_RECEIVED_UTC_ISOFORMAT, (TIME_US / 1000) AS TIME_ELAPSED_MS FROM APACHE_LOGS WHERE (CAST(STATUS AS INT) / 100) >= 4
```

Save this file to `apache_log_error_filtering.sql`.
把文件保存为 `apache_log_error_filtering.sql`.

Let's take a look at the script.
让我们过一遍这个脚本.

The first statement defines the table `APACHE_LOGS` which represents the input stream. The `LOCATION` clause specifies the ZkHost (`localhost:2181`), the path of the brokers in ZooKeeper (`/brokers`) and the topic (`apache-logs`).
Note that Kafka data source requires primary key to be defined. That's why we put integer id for parsed JSON data.
第一个语句定义了一个表 `APACHE_LOGS` 代表输入流. `LOCATION` 从句指定了 ZkHost (`localhost:2181`), brokers路径(`/brokers`) 和 topic (`apache-logs`).
注意 Kafka 数据源必须定义一个主键. 这就是为什么我们为 JSON 数据设置了一个整数 id.

Similarly, the second statement specifies the table `APACHE_ERROR_LOGS` which represents the output stream. The `TBLPROPERTIES` clause specifies the configuration of [KafkaProducer](http://kafka.apache.org/documentation.html#producerconfigs) and is required for a Kafka sink table.
同样, 第二个语句指定了表 `APACHE_ERROR_LOGS`代表输出流. `TBLPROPERTIES` 从句指定了[KafkaProducer](http://kafka.apache.org/documentation.html#producerconfigs)的配置, 从句对于 Kafka sink表 是必须的.
 
The last statement defines the topology. Storm SQL only define the topology and run topology on DML statement. 
DDL statements define input data source, output data source, and user defined function which will be referred by DML statement.
最后的语句定义了一个 topology. Storm SQL 只会在 DML 语句上定义和运行 topology.
DDL 语句定义输入数据源、输出数据源、以及可以被DML语句引用的用户定义函数(user defined function).

Let's look at where statement first. Since we want to filter error logs, we divide status by 100 and compare quotient is equal or greater than 4. (easier representation is `>= 400`)
Since status in JSON is string format (hence represented as VARCHAR for APACHE_LOGS table), we apply CAST(STATUS AS INT) to convert to integer type before applying division.
Now we have filtered only error logs. 
我们先看 where 语句. 由于我们想过滤错误日志, 我们使用状态码除以100, 比较得到的商是否等于或者大于4.(简单的说就是 statu_code >= 400)
由于JSON中的状态码是字符串格式(因此在 APACHE_LOGS 表中是 VARCHAR 格式). 我们在应用除法之前, 使用 CAST(STATUS AS INT) 先把状态码转换为整数.
现在我们只有 error 日志了.

Let's transform some columns to match the output stream. In this statement we apply CAST(STATUS AS INT) to convert to integer type, and divide TIME_US by 1000 to convert microsecond to millisecond.
让我们转换一些列以和输出流想匹配. 在这个语句中, 我们使用 CAST(STATUS AS INT) 转化为整数类型, 然后使用 1000 除 TIME_US 将毫秒转换成秒.

Last, insert statement stores filtered and transformed rows (tuples) to the output stream.  
最后, insert 语句将过滤和转换后的行(tuples)存入输出流.

To run this example, users need to include the data sources (`storm-sql-kafka` in this case) and its dependency in the
class path. Dependencies for Storm SQL are automatically handled when users run `storm sql`. 
Users can include data sources at the submission step like below:
要运行这个例子, 用户需要包含数据源(本例中是 `storm-sql-kafka`) 和的所有依赖到 class path 中. 当用户运行 `storm sql` 命令的时候 Storm SQL 的依赖会被自动处理.
用户可以在提交阶段包含数据源依赖, 如下：

```
$ $STORM_DIR/bin/storm sql apache_log_error_filtering.sql apache_log_error_filtering --artifacts "org.apache.storm:storm-sql-kafka:2.0.0-SNAPSHOT,org.apache.storm:storm-kafka:2.0.0-SNAPSHOT,org.apache.kafka:kafka_2.10:0.8.2.2^org.slf4j:slf4j-log4j12,org.apache.kafka:kafka-clients:0.8.2.2"
```

Above command submits the SQL statements to StormSQL. The option of storm sql is `storm sql [script file] [topology name]`. 
Users need to modify each artifacts' version if users are using different version of Storm or Kafka.
上面的命令提交 SQL 语句到 StormSQL. storm sql 命令的选项是 `storm sql [script file] [topology name]`.
如果用户使用了不同版本的 Storm 或者 Kafka ，需要修改每个 artifacts 的版本号与之对应.
 
If your statements pass the validation phase, topology will be shown to Storm UI page.
如果你的语句通过了验证阶段, 会在 Storm UI 页面上显示 topology.

You can see the output via console:
你可以在控制台上看到下面的输出:

```
$ kafka-console-consumer --zookeeper localhost:2181 --topic apache-error-logs
```

and the output will be similar to:
输出类似下面的内容:

```
{"ID":854643,"REMOTE_IP":"4.227.214.159","REQUEST_URL":"/wp-content","REQUEST_METHOD":"GET","STATUS":404,"REQUEST_HEADER_USER_AGENT":"Mozilla/5.0 (Windows 98; Win 9x 4.90; it-IT; rv:1.9.2.20) Gecko/2015-06-03 11:20:16 Firefox/3.6.17","TIME_RECEIVED_UTC_ISOFORMAT":"2021-03-28T19:14:44+00:00","TIME_RECEIVED_TIMESTAMP":1616958884000,"TIME_ELAPSED_MS":274.222}
{"ID":854693,"REMOTE_IP":"223.50.249.7","REQUEST_URL":"/apps/cart.jsp?appID=5578","REQUEST_METHOD":"GET","STATUS":404,"REQUEST_HEADER_USER_AGENT":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_6_6; rv:1.9.2.20) Gecko/2015-11-06 00:20:43 Firefox/3.8","TIME_RECEIVED_UTC_ISOFORMAT":"2021-03-28T21:41:02+00:00","TIME_RECEIVED_TIMESTAMP":1616967662000,"TIME_ELAPSED_MS":716.851}
...
```

You can also run Storm SQL runner to see the logical plan via placing `--explain` to topology name:
你可以运行 Storm SQL runner, 将 topology 名称替换为 `--explain` 来查看逻辑执行计划.

```
$ $STORM_DIR/bin/storm sql apache_log_error_filtering.sql --explain --artifacts "org.apache.storm:storm-sql-kafka:2.0.0-SNAPSHOT,org.apache.storm:storm-kafka:2.0.0-SNAPSHOT,org.apache.kafka:kafka_2.10:0.8.2.2^org.slf4j:slf4j-log4j12,org.apache.kafka:kafka-clients:0.8.2.2"
```

and the output will be similar to:
输入类似下面的内容:

```
LogicalTableModify(table=[[APACHE_ERROR_LOGS]], operation=[INSERT], updateColumnList=[[]], flattened=[true]), id = 8
  LogicalProject(ID=[$0], REMOTE_IP=[$1], REQUEST_URL=[$2], REQUEST_METHOD=[$3], STATUS=[CAST($4):INTEGER NOT NULL], REQUEST_HEADER_USER_AGENT=[$5], TIME_RECEIVED_UTC_ISOFORMAT=[$6], TIME_ELAPSED_MS=[/($7, 1000)]), id = 7
    LogicalFilter(condition=[>=(/(CAST($4):INTEGER NOT NULL, 100), 4)]), id = 6
      EnumerableTableScan(table=[[APACHE_LOGS]]), id = 5
```

It might be not same as you are seeing if Storm SQL applies query optimizations.
如果 Storm SQL 应用了查询优化, 你可能看到的输出会和上面的不一样.

We're executing the first Storm SQL topology! Please kill the topology when you see enough output and the logs.
我们正在执行第一个 Storm SQL topology! 如果你看到了足够多的输出和日志, 请杀掉 topology.

To be concise, we'll skip explaining the things we've already seen.
为了简洁, 我们不再解释我们已经看到的东西.

## 例子: 过滤时间较长的日志

In this example we'll filter slow logs from entire logs and store them to another topics. `project` and `filter`, and `User Defined Function (UDF)` features will be used.
This is very similar to `filtering error logs` but we'll see how to define `User Defined Function (UDF)`.

The content of script file is here:

```
CREATE EXTERNAL TABLE APACHE_LOGS (ID INT PRIMARY KEY, REMOTE_IP VARCHAR, REQUEST_URL VARCHAR, REQUEST_METHOD VARCHAR, STATUS VARCHAR, REQUEST_HEADER_USER_AGENT VARCHAR, TIME_RECEIVED_UTC_ISOFORMAT VARCHAR, TIME_US DOUBLE) LOCATION 'kafka://localhost:2181/brokers?topic=apachelogs' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.storm.kafka.ByteBufferSerializer"}}'
CREATE EXTERNAL TABLE APACHE_SLOW_LOGS (ID INT PRIMARY KEY, REMOTE_IP VARCHAR, REQUEST_URL VARCHAR, REQUEST_METHOD VARCHAR, STATUS INT, REQUEST_HEADER_USER_AGENT VARCHAR, TIME_RECEIVED_UTC_ISOFORMAT VARCHAR, TIME_RECEIVED_TIMESTAMP BIGINT, TIME_ELAPSED_MS INT) LOCATION 'kafka://localhost:2181/brokers?topic=apacheslowlogs' TBLPROPERTIES '{"producer":{"bootstrap.servers":"localhost:9092","acks":"1","key.serializer":"org.apache.storm.kafka.IntSerializer","value.serializer":"org.apache.storm.kafka.ByteBufferSerializer"}}'
CREATE FUNCTION GET_TIME AS 'org.apache.storm.sql.runtime.functions.scalar.datetime.GetTime2'
INSERT INTO APACHE_SLOW_LOGS SELECT ID, REMOTE_IP, REQUEST_URL, REQUEST_METHOD, CAST(STATUS AS INT) AS STATUS_INT, REQUEST_HEADER_USER_AGENT, TIME_RECEIVED_UTC_ISOFORMAT, GET_TIME(TIME_RECEIVED_UTC_ISOFORMAT, 'yyyy-MM-dd''T''HH:mm:ssZZ') AS TIME_RECEIVED_TIMESTAMP, TIME_US / 1000 AS TIME_ELAPSED_MS FROM APACHE_LOGS WHERE (TIME_US / 1000) >= 100
```

Save this file to `apache_log_slow_filtering.sql`.

We can skip the first 2 statements since it's almost same to the last example.

The third statement defines the `User defined function`. We're defining `GET_TIME` which uses `org.apache.storm.sql.runtime.functions.scalar.datetime.GetTime2` class.

The implementation of GetTime2 is here:

```
package org.apache.storm.sql.runtime.functions.scalar.datetime;

import org.joda.time.format.DateTimeFormat;
import org.joda.time.format.DateTimeFormatter;

public class GetTime2 {
    public static Long evaluate(String dateString, String dateFormat) {
        try {
            DateTimeFormatter df = DateTimeFormat.forPattern(dateFormat).withZoneUTC();
            return df.parseDateTime(dateString).getMillis();
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }
    }
}
```

This class can be used for UDF since it defines static `evaluate` method. The SQL type of parameters and return are determined by Calcite which Storm SQL depends on. 

Note that this class should be in classpath, so in order to define UDF, you need to create jar file which contains UDF classes and run `storm sql` with `--jar` option.
This page assumes that GetTime2 is in classpath, for simplicity.
 
The last statement is very similar to filtering error logs. The only new thing is that we call `GET_TIME(TIME_RECEIVED_UTC_ISOFORMAT, 'yyyy-MM-dd''T''HH:mm:ssZZ')` to convert string time to unix timestamp (BIGINT).

Let's execute it.

```
$ $STORM_DIR/bin/storm sql apache_log_slow_filtering.sql apache_log_slow_filtering --artifacts "org.apache.storm:storm-sql-kafka:2.0.0-SNAPSHOT,org.apache.storm:storm-kafka:2.0.0-SNAPSHOT,org.apache.kafka:kafka_2.10:0.8.2.2^org.slf4j:slf4j-log4j12,org.apache.kafka:kafka-clients:0.8.2.2"
```

You can see the output via console:

```
$ kafka-console-consumer --zookeeper localhost:2181 --topic apache-slow-logs
```

and the output will be similar to:

```
{"ID":890502,"REMOTE_IP":"136.156.159.160","REQUEST_URL":"/list","REQUEST_METHOD":"GET","STATUS":200,"REQUEST_HEADER_USER_AGENT":"Mozilla/5.0 (Windows NT 5.01) AppleWebKit/5311 (KHTML, like Gecko) Chrome/13.0.860.0 Safari/5311","TIME_RECEIVED_UTC_ISOFORMAT":"2021-06-05T03:44:59+00:00","TIME_RECEIVED_TIMESTAMP":1622864699000,"TIME_ELAPSED_MS":638.579}
{"ID":890542,"REMOTE_IP":"105.146.3.190","REQUEST_URL":"/search/tag/list","REQUEST_METHOD":"DELETE","STATUS":200,"REQUEST_HEADER_USER_AGENT":"Mozilla/5.0 (X11; Linux i686) AppleWebKit/5332 (KHTML, like Gecko) Chrome/13.0.891.0 Safari/5332","TIME_RECEIVED_UTC_ISOFORMAT":"2021-06-05T05:54:27+00:00","TIME_RECEIVED_TIMESTAMP":1622872467000,"TIME_ELAPSED_MS":403.957}
...
```

That's it! Supposing we have UDF which queries geo location via remote ip, we can filter via geo location, or enrich geo location to transformed result.

## Summary

We looked through several simple use cases for Storm SQL to learn Storm SQL features. If you haven't looked at [Storm SQL integration](storm-sql.html) and [Storm SQL language](storm-sql-reference.html), you need to read it to see full supported features. 

Note that Storm SQL is running on Trident, which is micro-batch, and also no strong typed. Sink doesn't actually check the type.
(You may noticed that the types of some of output fields are different than output table schema.)

Its behavior is subject to change when Storm SQL changes its backend API to core (tuple by tuple, low-level or high-level) one.