---
title: Clojure DSL
layout: documentation
documentation: true
---
Storm配有Clojure DSL，用于定义spouts(喷口)，bolts(螺栓)和topologies(拓扑)。 Clojure DSL可以访问Java API暴露的所有内容，因此如果您是Clojure用户，您可以直接编写Storm拓扑，根本不需要使用Java。 Clojure DSL 的源码在 [org.apache.storm.clojure]({{page.git-blob-base}}/storm-core/src/clj/org/apache/storm/clojure.clj)命名空间中定义。

本页概述了Clojure DSL的所有功能，包括：

1. Defining topologies(定义拓扑)
2. `defbolt`
3. `defspout`
4. Running topologies in local mode or on a cluster(在本地模式或集群上运行拓扑)
5. Testing topologies(测试拓扑)

### Defining topologies(定义拓扑)

请使用`topology`函来定义topology(拓扑)。`topology`有两个参数：“spout specs(规格)”的映射和“bolt specs(规格)” 的映射。每个spouts specs(规格)和bolt specs(规格) 通过指定输入和并行度 来将组件的代码连接到topology中。


我们来看一下[storm-starter ]({{page.git-blob-base}}/examples/storm-starter/src/clj/org/apache/storm/starter/clj/word_count.clj):项目中的topology定义示例：

```clojure
(topology
 {"1" (spout-spec sentence-spout)
  "2" (spout-spec (sentence-spout-parameterized
                   ["the cat jumped over the door"
                    "greetings from a faraway land"])
                   :p 2)}
 {"3" (bolt-spec {"1" :shuffle "2" :shuffle}
                 split-sentence
                 :p 5)
  "4" (bolt-spec {"3" ["word"]}
                 word-count
                 :p 6)})
```

spout和bolt specs(规格)的映射是从组件ID到相应规格的映射。组件ID必须在映射上是唯一的。就像在Java中定义topologies(拓扑)一样，在声明topologies(拓扑)中的bolts 输入时使用的组件ID。

#### spout-spec（spout-规格）

`spout-spec`作为spout实现（实现 [IRichSpout](javadocs/org/apache/storm/topology/IRichSpout.html))的对象）的可选关键字参数的参数 。当前存在的唯一选项是：`:p`选项，它指定了spout的并行性。如果您省略`:p`，则spout将作为单个任务执行。

#### bolt-spec（bolt-规格）

`bolt-spec`作为bolt实现（实现IRichBolt的对象）的可选关键字参数的参数

输入声明是从stream ids到stream groupings的映射。stream id 可以有以下两种形式中的一种：

1. `[==component id== ==stream id==]`: Subscribes to a specific stream on a component(订阅组件上的特定流)
2. `==component id==`: Subscribes to the default stream on a component(订阅组件上的默认流)

stream grouping可以是以下之一：

1. `:shuffle`: 用shuffle grouping进行订阅
2.  Vector of field names, like ["id" "name"](多字段名, 像 `["id" "name"]`): 使用fields grouping订阅指定的字段
3. `:global`:  使用global grouping进行订阅
4. `:all`: 使用all grouping进行订阅
5. `:direct`: 使用direct grouping进行订阅

See [Concepts](Concepts.html) for more info on stream groupings. Here's an example input declaration showcasing the various ways to declare inputs:
有关stream groupings的更多信息，请参阅[概念](Concepts.html)。下面是一个输入声明的示例，展示各种声明输入的方法：

```clojure
{["2" "1"] :shuffle
 "3" ["field1" "field2"]
 ["4" "2"] :global}
```

此输入声明共计三个流。它通过随机分组来订阅组件“2”上的流“1”，在字段“field1”和“field2”上以fields grouping的方式订阅组件“3”上的默认流，使用全局分组在组件“4”上订阅流“2”。

像`spout-spec`一样，bolt-spec唯一当前支持的关键字参数是：p，它指定了bolt的并行性。

#### shell-bolt-spec (shell-bolt-规格)

`shell-bolt-spec` is used for defining bolts that are implemented in a non-JVM language. It takes as arguments the input declaration, the command line program to run, the name of the file implementing the bolt, an output specification, and then the same keyword arguments that `bolt-spec` accepts.
`shell-bolt-spec`用于定义以非JVM语言实现的bolts。它作为输入声明参数，在命令行程序中运行，用文件的名称实现bolt，输出规范 以及接受的相同关键字参数作为参数的 `bolt-spec` 。

这有一个shell-bolt-spec的例子：

```clojure
(shell-bolt-spec {"1" :shuffle "2" ["id"]}
                 "python"
                 "mybolt.py"
                 ["outfield1" "outfield2"]
                 :p 25)
```

输出声明的语法在下面的defbolt部分中有更详细的描述。有关Storm的工作原理的详细信息，请参阅[使用Storm的非JVM语言](Using-non-JVM-languages-with-Storm.html) 。

### defbolt

`defbolt` 用于在Clojure中定义bolts。这里对bolts有一个限制，那就是他必须是可序列化的，这就是为什么你不能仅仅具体化`IRichBolt`来实现一个bolts（closures不可序列化）。 `defbolt` 在这个限制的基础上为定义bolts提供了一种更好的语法，而不仅仅是实现一个Java接口的。

在最充分的表现形势下，`defbolt`支持参数化bolts，并在bolts执行期间保持关闭状态。它还提供了用于定义不需要额外功能的bolts的快捷方式。 `defbolt`的签名如下所示：

(defbolt _name_ _output-declaration_ *_option-map_ & _impl_)

省略option map(选项映射)相当于具有{：prepare false}的option map(选项映射)。

#### Simple bolts (简单 bolts)

我们从最简单的defbolt形式开始吧。这是一个将包含句子的元组分割成每个单词的元组的示例bolt：

```clojure
(defbolt split-sentence ["word"] [tuple collector]
  (let [words (.split (.getString tuple 0) " ")]
    (doseq [w words]
      (emit-bolt! collector [w] :anchor tuple))
    (ack! collector tuple)
    ))
```

Since the option map is omitted, this is a non-prepared bolt. The DSL simply expects an implementation for the `execute` method of `IRichBolt`. The implementation takes two parameters, the tuple and the `OutputCollector`, and is followed by the body of the `execute` function. The DSL automatically type-hints the parameters for you so you don't need to worry about reflection if you use Java interop.
(由于感觉有不准确的地方，先留着方便优化。)
由于省略了option map(选项映射)，这是一个non-prepared bolt。 DSL只是期望执行一个IRichBolt的`execute`方法。该实现需要两个参数，即tuple(元组)和`OutputCollector`，后面是`execute`函数的正文。 DSL会为你自动提示参数，所以如果您使用Java交互，不需要担心反射问题。


This implementation binds `split-sentence` to an actual `IRichBolt` object that you can use in topologies, like so:
此实现将`split-sentence`绑定到一个可用于topologies实现的`IRichBolt`对象，如下所示：
```clojure
(bolt-spec {"1" :shuffle}
           split-sentence
           :p 5)
```


#### Parameterized bolts  (参数化 bolts)

有时候你想用其他参数来参数化你的bolts。例如，假设你想有一个可以接收到每个输入字符串后缀的bolts，并且希望在运行时设置该后缀。你可以在defbolt中通过在option map(选项映射)中包含：`:params`选项来执行此操作，如下所示：

```clojure
(defbolt suffix-appender ["word"] {:params [suffix]}
  [tuple collector]
  (emit-bolt! collector [(str (.getString tuple 0) suffix)] :anchor tuple)
  )
```

与前面的示例不同，`suffix-appender`将绑定到一个返回`IRichBolt`而不是直接作为`IRichBolt`对象的函数。这是通过在其option map(选项映射)中指定`:params`引起的。因此，在topology中使用`suffix-appender`，您可以执行以下操作：

```clojure
(bolt-spec {"1" :shuffle}
           (suffix-appender "-suffix")
           :p 10)
```

#### Prepared bolts (准备 bolts)


要做更复杂的bolts，如加入和流聚合的bolt，bolt需要存储状态。您可以通过在option map(选项映射)中创建一个通过包含`{:prepare true}`指定的prepared bolt 来实现此目的。例如，思考下这个实现单词计数的bolt：

```clojure
(defbolt word-count ["word" "count"] {:prepare true}
  [conf context collector]
  (let [counts (atom {})]
    (bolt
     (execute [tuple]
       (let [word (.getString tuple 0)]
         (swap! counts (partial merge-with +) {word 1})
         (emit-bolt! collector [word (@counts word)] :anchor tuple)
         (ack! collector tuple)
         )))))
```

prepared bolt的实现是通过一个函数 ，它将topology的配置“TopologyContext”和“OutputCollector”作为输入，并返回“IBolt”接口的一个实现。此设计允许您围绕`execute`和`cleanup`的实现时进行闭包。

在这个例子中，单词计数存储在一个名为`counts`的映射的闭包中。 `bolt`宏用于创建`IBolt`实现。 `bolt`宏是一种比简化实现界面更简洁的方法，它会自动提示所有的方法参数。该bolt实现了更新映射中的计数并发出新的单词计数的执行方法。

请注意， prepared bolts 中的`execute`方法只能作为元组的输入，因为`OutputCollector`已经在函数的闭包中（对于简单的bolts，collector是`execute`函数的第二个参数）。

Prepared bolts 可以像 simple bolts 一样进行参数化。

#### Output declarations (输出声明)

Clojure DSL具有用于bolt输出的简明语法。声明输出的最通用的方法就是从stream id到stream spec的映射。例如：

```clojure
{"1" ["field1" "field2"]
 "2" (direct-stream ["f1" "f2" "f3"])
 "3" ["f1"]}
```

stream id 是一个字符串，而stream spec(流规范)是个字段的向量或由`direct-stream`包装的字段的向量。 `direct stream`将流标记为direct stream（有关直接流的更多详细信息，请参阅[Concepts](Concepts.html) 和[Direct groupings](空的。。)）。


如果bolt只有一个输出流，您可以使用向量而不用输出声明的映射来定义bolt的默认流。例如：

```clojure
["word" "count"]
```

这段bolt输出的声明 为默认 stream id 上的字段[“word” “count”]。
#### Emitting, acking, and failing  (发射，确认和失败)


DSL可以使用`OutputCollector`：`emit-bolt！`，`emit-direct-bolt！`，`ack！`和`fail ！`，而不用直接在`OutputCollector`上使用Java方法.

1. `emit-bolt！`：将“OutputCollector”，发出的值（一个Clojure sequence）和`：anchor`以及`：stream`的关键字参数作为参数。 `：anchor`可以是个single tuple或一个list of tuples，`：stream`是要发送到的流的id。 若省略关键字参数则默认流会发出一个unanchored tuple。
2. `emit-direct-bolt！`：将`OutputCollector`作为参数，发送元组的任务id，发送的值，以及把`：anchor`和`：stream`的关键字参数作为参数。 此函数只能发出声明为direct streams的流。
3. `ack!`: 将“OutputCollector”作为元组确认参数。
4. `fail!`: 将“OutputCollector”作为元组失败参数

有关确认和锚定的更多信息，请参阅[保证消息处理](Guaranteeing-message-processing.html)。
### defspout

`defspout`用于定义Clojure中的喷口。像螺栓一样，喷口必须是可序列化的，所以您不能只是在“Clojure”中引用“IRichSpout”来执行喷口实现。 `defspout`围绕这个限制，为定义spouts提供了一个更好的语法，而不仅仅是实现一个Java接口。

`defspout`的签名如下：

(defspout _name_ _output-declaration_ *_option-map_ & _impl_)

如果你省略选项映射，则默认为{：prepare true}。 `defspout`的输出声明与`defbolt`语法相同。

这里有个实现`defspout`的一个例子[storm-starter]({{page.git-blob-base}}/examples/storm-starter/src/clj/org/apache/storm/starter/clj/word_count.clj):

```clojure
(defspout sentence-spout ["sentence"]
  [conf context collector]
  (let [sentences ["a little brown dog"
                   "the man petted the dog"
                   "four score and seven years ago"
                   "an apple a day keeps the doctor away"]]
    (spout
     (nextTuple []
       (Thread/sleep 100)
       (emit-spout! collector [(rand-nth sentences)])         
       )
     (ack [id]
        ;; You only need to define this method for reliable spouts
        ;; (such as one that reads off of a queue like Kestrel)
        ;; This is an unreliable spout, so it does nothing here
        ))))
```

该实现将topology配置的“TopologyContext”和“SpoutOutputCollector”作为输入。该实现返回一个`ISpout`对象。这里，`nextTuple`函数从`sentence`发出一个随机语句。

这个spout不是可靠的，所以`ack`和`fail`方法永远不会被调用。一个可靠的端口将在发出元组时添加一条消息ID，然后当元组完成或失败时，将会调用`ack`或`fail`。有关Storm中可靠性如何工作的更多信息，请参阅[保证消息处理](Guaranteeing-message-processing.html)。

`emit-spout！`将“SpoutOutputCollector”和新元组的参数作为参数发送，并接受作为关键字参数`：stream`和`：id`。 `：stream`为指定要发送的流，`：id`为指定元组的消息ID（在`ack'`和`fail`回调中使用）。省略这些参数会为默认输出流发出一个unanchored tuple。

这还有一个`emit-direct-spout !`函数，他会发出一个direct stream的元组，并附加一个任务id作为的第二个参数来发送这个元组。

Spouts可以像bolts一样进行参数化，在这种情况下，symbol绑定到返回“IRichSpout”的函数而不是“IRichSpout”本身。您还可以声明一个unprepared spout，它只定义`nextTuple`方法。以下是在运行时发出随机语句参数化的unprepared spout示例：

```clojure
(defspout sentence-spout-parameterized ["word"] {:params [sentences] :prepare false}
  [collector]
  (Thread/sleep 500)
  (emit-spout! collector [(rand-nth sentences)]))
```

以下示例说明了如何在`spout-spec`中使用此spout：
```clojure
(spout-spec (sentence-spout-parameterized
                   ["the cat jumped over the door"
                    "greetings from a faraway land"])
            :p 2)
```

### Running topologies in local mode or on a cluster (在本地模式或集群上运行topologies)

要想使用远程模式或本地模式提交topologies，只需像Java一样使用“StormSubmitter”或“LocalCluster”类。这就是Clojure DSL。

要创建topology配置，最简单的方法是使用org.apache.storm.config命名空间来定义所有可能配置的常量。常量与“Config”类中的静态常量相同，但是使用的是破折号而不是下划线。例如，这有一个topology配置，将workers数设置为15，并以调试模式配置topology：

```clojure
{TOPOLOGY-DEBUG true
 TOPOLOGY-WORKERS 15}
```

### Testing topologies （测试topologies）

关于测试Clojure中的topologies ，[博文](http://www.pixelmachine.org/2011/12/17/Testing-Storm-Topologies.html)及其[后续](http://www.pixelmachine.org/2011/ 12/21 / Testing-Storm-Topology-Part-2.html) 很好地概述了Storm的强大内置功能。
