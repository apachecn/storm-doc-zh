---
title: 动态员工分析
layout: documentation
documentation: true
---

在多组户模式下，storm通过集群启动长时间运行的JVM，而无需sudo访问用户。Java堆，jstacks和Java分析这些JVM的自服务将提高用户在主动监听时分析和调试问题的能力。

storm 动态分析器可以让您动态地对库存群集上运行的JVM进行head-dump,jprofiler或jstack.它让用户从浏览器下载这些转存，并使用您最喜爱的工具进行分析。UI组件页面为组件和操作按钮提供列表工作人员。Logviewer可以让您下载这些日志生成的转储，有关详细信息，请参阅截图。

使用 Storm UI
-------------

为了请求堆转储，jstack,启动/停止/转储jprofile或重新启动一个工作者，点击运行的拓扑，然后点击特定的组件，然后您可以通过选中任何工作人员的执行者的框来选择工作人 执行程序表，然后在“分析和调试”部分中单击“开始”，“堆”，“堆栈”或“人工重启”。

![Selecting Workers](images/dynamic_profiling_debugging_4.png "Selecting Workers")

在 Exceutors表中，单击任何执行程序旁边的“操作”列中的复选框，并且自动选择属于同一个工作的任何其他执行程序。操作完成后，创建的任何文件输出文件将在“操作”列中的链接处可用。

![Profiling and Debugging](images/dynamic_profiling_debugging_1.png "Profiling and Debugging")

对于启动jprofile,提供以分钟为单位的超时（如果不需要则为10）。然后点击"开始"。

![After starting jprofile for worker](images/dynamic_profiling_debugging_2.png "After jprofile for worker ")

要停止jprofile日志记录，单击"停止"按钮。这将转储jprofile统计信息并停止分析。刷新该行的页面从UI消失。

单击“我的转储文件”，以转到用于特定于工作的转储文件列表的日志查看器UI。

![Dump Files Links for worker](images/dynamic_profiling_debugging_3.png "Dump Files Links for worker")

配置
-------------

可以将“worker.profiler.command”配置为指向特定的可插拔分析器，heapdump命令。如果插件不可用或jdk不支持JProfile航班录制，"worker.profiler.enabled"可以补禁用，以便工作JVM选项不会有"worker.profiler.childopts"。要使用不同的profiler插件，您可以更改这些配置。


