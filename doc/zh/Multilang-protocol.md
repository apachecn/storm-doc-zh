---
title: 多朗协议
layout: documentation
documentation: true
---
本页介绍了Storm 0.7.1中的multilang 协议。0.7.1之前的版本使用了一个有些不同的协议，文档位于 [here](Storm-multi-language-protocol-(versions-0.7.0-and-below\).html).

# Storm 多朗协议

## Shell 组件

通过ShellBolt,ShellSpout和ShellProcess类实现对多语言的支持。这些类实现IBolt和ISpout接口以及执行脚本的协议或程序通过Shell使用Java的ProcessBuilder类。

### 包装Shell脚本

默认情况下，ShellPorcess假定您的代码打包在您的jar的resources子目录下的拓扑Jar内，默认情况会更改当前的工作目录，该可执行线程是从Jar中提取的资源目录。一个Jar没有存储其中文件的权限。这包括允许Shell脚本由操作系统加载和运行的执行位。因此，在大多数示例中，脚本具有`python mybolt.py`的形式，因为python可执行文件已经在主管上，mybolt忆打包在jar的资源目录中。

如果你想打包更复杂的东西，像一个新版本的python本身，你需要改用blod这个存储和一个支持权限的`.tgz` 档案。

可以看这个文档 [Blob Store](distcache-blobstore.html) 有更加详细的说明怎么运送jar的细节。

使用ShellBolt/ShellSpout与可执行文件+脚本一起发布在blod store cache中。

```
changeChildCWD(false);
```

在ShellBolt/ShellSpout的构造函数中。shell命令将相对于工作者的cwd。哪里的资源链接。

所以如果我发送python与一个名为`newPython`和一个python ShellSpout的符号链接我并发送到`shell_spout.py`，我会有如下写法

```
public MyShellSpout() {
    super("./newPython/bin/python", "./shell_spout.py");
    changeChildCWD(false);
}
```

## 输出字段

输出字段是Thrift拓扑定义的一部分。这就意味着当您在java中的multing时，您需要创建一个扩展ShellBolt的bolt,实现IRichBolt,并声明`declareOutputFields`(类似于ShellSpout)中的字段。

您可以学习更多关于 [Concepts](Concepts.html)

## 协议序言

一个简单的协议是通过STDIN和STDOUT来实现的执行脚本或程序。与该过程交换的所有数据为JSON格式，几乎可以支持任何语言。

# 包装你的东西

要在集群上运行Shell组件，那就是shelled的脚本必须在jar中提供的`resources/`目录中给master。

但是，在本地机器的开发或测试过程中，资源目录只需要在类路径中。

## 协议

Notes:

* 该协议的两端使用线读机制，所以一定要从输入中剪掉换行符并将其追加到输出中。
* 所有JSON输入和输出都由包含"end"的单行终止。请注意，此分隔符本身不是JSON编码的。
* 下面的项目符号是从脚本作者的角度编写的STDIN和STDOUT。

### 初始化握手

两种类型的shell组件的初始化握手是相同的：

* STDIN: 设置信息。这是一个具有Storm配置，PID目录和拓扑上下文的JSON对象，像这样：

```
{
    "conf": {
        "topology.message.timeout.secs": 3,
        // etc
    },
    "pidDir": "...",
    "context": {
        "task->component": {
            "1": "example-spout",
            "2": "__acker",
            "3": "example-bolt1",
            "4": "example-bolt2"
        },
        "taskid": 3,
        // Everything below this line is only available in Storm 0.10.0+
        "componentid": "example-bolt"
        "stream->target->grouping": {
        	"default": {
        		"example-bolt2": {
        			"type": "SHUFFLE"}}},
        "streams": ["default"],
 		"stream->outputfields": {"default": ["word"]},
	    "source->stream->grouping": {
	    	"example-spout": {
	    		"default": {
	    			"type": "FIELDS",
	    			"fields": ["word"]
	    		}
	    	}
	    }
	    "source->stream->fields": {
	    	"example-spout": {
	    		"default": ["word"]
	    	}
	    }
	}
}
```

您的脚本应该在此目录中创建一个以其PID命名的空文件。例如，PID为1234，因此在目录中创建名为1234的空文件。这个文件让主管知道PID,以便稍后关闭该过程。

从Storm 0.10.0起，Storm发送到shell组件的上下文一直是大大增强包括可用于JVM组件的拓扑上下文的所有方面。一个关键的补充是能够确定拓扑结构中的shell组件的源和目标（即输入和输出）`stream->target->grouping` and `source->stream->grouping` 字典。在这些嵌套字典的最内层，分组被表示为一个最低限度具有`type`键的字典，但也可以有一个`fields`键，指定`FIELDS`分组中涉及哪些字段。

* STDOUT: 你的PID，在JSON对象中，像 `{"pid": 1234}`。shell组件将PID记录到其日志中。

接下来会发生什么取决于组件的类型：

### Spouts

Shell spouts 是同步的. 其余的发生在一段时间(true)循环：

* STDIN: 下一个，ack,激活，停用或失败命令。

"next" 相当于ISpout's的`nextTuple`。看起来就像：

```
{"command": "next"}
```

"ack" 看起来像:

```
{"command": "ack", "id": "1231231"}
```

"activate" 相当于ISpout's的 `activate`:
```
{"command": "activate"}
```

"deactivate" 相当于ISpout's的 `deactivate`:
```
{"command": "deactivate"}
```

"fail" 看起来像:

```
{"command": "fail", "id": "1231231"}
```

* STDOUT: 您以前命令的输出结果。这可以是一系列发射和日志。

An emit looks like:

```
{
	"command": "emit",
	// The id for the tuple. Leave this out for an unreliable emit. The id can
    // be a string or a number.
	"id": "1231231",
	// The id of the stream this tuple was emitted to. Leave this empty to emit to default stream.
	"stream": "1",
	// If doing an emit direct, indicate the task to send the tuple to
	"task": 9,
	// All the values in this tuple
	"tuple": ["field1", 2, 3]
}
```

如果不直接执行emit,则将立即收到STDIN上以元数组发布的元组为JSON数组。

"log" 将在工作日志中记录一条消息。看起来像：

```
{
	"command": "log",
	// the message to log
	"msg": "hello world!"
}
```

* STDOUT: "sync"命令结束发射和日志的顺序。看起来像：

```
{"command": "sync"}
```

同步之后，ShellSpout将不会读取您的输出，直到它发送另一个next,ack，或fail命令。

请注意，与ISpout类似，工作人员的所有spouts将在下一次，确认或失败后被锁定，直到您同步。也像ISpout,如果没有元组为下一个发出，您应该睡眠少量的时间才能同步。ShellSpout不会自动为您做睡眠。

### Bolts

The shell bolt 是异步的. 您将在STDIN上收到元组，只要它们可用，您可以发出，确认或失败，并随时通过写入SDTOUT，如下所示:

* STDIN: 一个元组！这是一个这样的JSON编码结构: 

```
{
    // The tuple's id - this is a string to support languages lacking 64-bit precision
	"id": "-6955786537413359385",
	// The id of the component that created this tuple
	"comp": "1",
	// The id of the stream this tuple was emitted to
	"stream": "1",
	// The id of the task that created this tuple
	"task": 9,
	// All the values in this tuple
	"tuple": ["snow white and the seven dwarfs", "field2", 3]
}
```

* STDOUT: An ack, fail, emit, or log. Emits look like:

```
{
	"command": "emit",
	// The ids of the tuples this output tuples should be anchored to
	"anchors": ["1231231", "-234234234"],
	// The id of the stream this tuple was emitted to. Leave this empty to emit to default stream.
	"stream": "1",
	// If doing an emit direct, indicate the task to send the tuple to
	"task": 9,
	// All the values in this tuple
	"tuple": ["field1", 2, 3]
}
```

如果不直接执行emit，那么您将会收到在STDIN上发布元组的任务ids作为JSON数组。请注意，由于异步性质的shell bolt协议，当你读后你可以收不到任务的ids。你可以改为阅读要处理的先前发布或新元组的任务ids。但是你将按照相应的排放顺序接收任务id列表。

An ack 看起来像:

```
{
	"command": "ack",
	// the id of the tuple to ack
	"id": "123123"
}
```

A fail 看起来像:

```
{
	"command": "fail",
	// the id of the tuple to fail
	"id": "123123"
}
```

A "log" 将在工作日志中记录一条消息。看起来像：

```
{
	"command": "log",
	// the message to log
	"msg": "hello world!"
}
```

* 请注意，从0.7.1版本起，不再需要一个shell bolt进行 '同步'操作。 

### 心跳处理 (0.9.3 及以上)

直到Storm 0.9.3,心跳在ShellSpout/ShellBolt与它们之间多个子进程检测挂/子进程。任何通过多镜头与Storm进行连接的库，必须对听筒采取以下措施：

#### Spout

Shell spouts 是同步的,因此子流程总是在`next()`的末尾发送`sync`命令，所以你不必为支持spouts的心跳做很多工作。也就是说，在`next()`期间，不要让子进程睡眠超过工作超时。

#### Bolt

Shell bolts 是异步的, 因此ShellBolt将定期向其子进程发送心跳元组。心跳元组看起来像：

```
{
	"id": "-6955786537413359385",
	"comp": "1",
	"stream": "__heartbeat",
	// this shell bolt's system task id
	"task": -1,
	"tuple": []
}
```

当子进程接收到心跳元组时，它必须发送一个`sync`命令回到ShellBolt。
